# SPEC: jira-enricher (v3.1)

> Второй скилл pipeline. Читает `pipeline/enriched.json`, для каждой задачи делает `jira_get_issue` БЕЗ changelog, дополняет данными из Jira (статус, эпик, команда, lead time). Дополнительно делает `jira_search` для подсчёта дочерних задач каждого уникального эпика.
>
> **Архитектурное правило (КРИТИЧНО):** агент в чате делает нативные tool calls, парсит JSON-ответы в своём контексте, накапливает результаты в виде JSON-строки. После каждого батча из 5 задач — передаёт накопленные данные в `helper.py` через bash (`echo '...' | python3 helper.py merge-batch`). Python скрипт **не делает MCP-вызовов**.

---

## 1. Контекст: что пошло не так в прошлой версии

В первом прогоне jira-enricher GigaCode попытался вызвать MCP **изнутри Python-скрипта**:

```python
# ❌ Это НЕ работает — NameError
result = mcp__Atlassian__jira_get_issue(key="CRSIGMA-26516")
```

В Python-окружении нет функций `mcp__*`. MCP tools — это **нативная способность агента** в чате GigaCode CLI, не Python-модули.

**Правильная модель:**

```
┌──────────────────────────────────────────────────────────┐
│ Агент (GigaCode CLI)                                     │
│                                                          │
│  1. Читает pipeline/enriched.json (через bash python3)   │
│  2. Для каждой задачи:                                   │
│     - Делает НАТИВНЫЙ tool call jira_get_issue           │
│     - Получает JSON-ответ в свой контекст                │
│     - Извлекает поля по правилам раздела 7               │
│     - Накапливает в текстовый JSON-батч                  │
│  3. После каждых 5 задач:                                │
│     - Формирует JSON-батч                                │
│     - Передаёт в helper.py через bash + stdin            │
│     - helper.py мерджит и пишет в enriched.json          │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│ helper.py (Python через bash)                            │
│                                                          │
│  - НЕ делает MCP-вызовов (это невозможно из Python)      │
│  - Принимает данные через stdin (JSON-batch)             │
│  - Читает pipeline/enriched.json                         │
│  - Мерджит batch в task.jira для каждой задачи           │
│  - Записывает обратно                                    │
└──────────────────────────────────────────────────────────┘
```

## 2. Место в pipeline

```
1. excel-parser              → pipeline/enriched.json (план)
2. jira-enricher  ◄── (этот скилл)
3. timing-analyzer
4. report-builder
```

## 3. Цели

- Прочитать `pipeline/enriched.json`, валидировать что `excel-parser` отработал
- Для каждой задачи с `cr_key` сделать **один** `jira_get_issue` БЕЗ `expand=changelog`
- Извлечь поля согласно CONTRACT.md секция "После jira-enricher"
- Применить mapping статус → category → phase
- Извлечь эпик через `customfield_11400` или `issuelinks "Implement in"`
- Извлечь команду через `customfield_22200` (массив строк) или fallback на assignee
- Посчитать `lead_time_days`
- Собрать уникальные эпики, для каждого сделать `jira_search('"Epic Link" = <key>')` для подсчёта дочерних
- Перезаписать `pipeline/enriched.json` с заполненной секцией `jira` и массивом `epics`
- Создать читаемый снимок `pipeline/step-2-after-jira-enricher.md`

## 4. Анти-цели

- **НЕ** запрашивать changelog — это работа `timing-analyzer`
- **НЕ** считать `phase_days` — у нас нет changelog
- **НЕ** генерировать финальный markdown-отчёт — это работа `report-builder` (только step-2 snapshot)
- **НЕ** читать Excel — только enriched.json

## 5. Формула вызова MCP — зафиксирована

### Для каждой задачи (нативный tool call агента в чате)

```
Tool: jira_get_issue

Параметры:
  key = <cr_key из enriched.json>
  fields = "summary,issuetype,status,project,created,updated,resolutiondate,
            reporter,assignee,priority,labels,description,parent,
            customfield_11400,customfield_22200,issuelinks"
```

**КРИТИЧНО:**
- НЕ передавать `expand=changelog` (это работа следующего скилла)
- НЕ использовать `fields="*"` (раздувает контекст)
- Это **нативный tool call агента**, не Python-функция

### Для каждого уникального эпика

После прохода по всем задачам — собрать уникальные `epic.key` (не пустые), для каждого:

```
Tool: jira_search

Параметры:
  jql = '"Epic Link" = <epic_key>'
  fields = "summary,status,issuetype"
  maxResults = 100
```

Результат — `len(issues)` записывается как `children_count_total`. Сами задачи **не** сохраняем.

## 6. Mapping статуса → category → phase

Используется единый mapping из CONTRACT.md секция "Маппинг статуса → category → phase".

Извлекается из `helper.py` функция `map_status(status_name) -> (category, phase)`.

## 7. Готовый код helper.py — используйте как основу

```python
# helper.py для jira-enricher

import json
import sys
import re
import os
from datetime import datetime, timezone

# === Mapping статусов ===

STATUS_MAP = {
    # not_started
    'backlog': ('not_started', None),
    'to do': ('not_started', None),
    'открыта': ('not_started', None),
    # analysis (А)
    'new': ('analysis', 'A'),
    'need info': ('analysis', 'A'),
    'analysis': ('analysis', 'A'),
    'анализ': ('analysis', 'A'),
    # development (Р)
    'in progress': ('development', 'R'),
    'разработка': ('development', 'R'),
    'готов к разработке': ('development', 'R'),
    # testing (Т)
    'ready for qa': ('testing', 'T'),
    'готов к тестированию': ('testing', 'T'),
    'начато тестирование': ('testing', 'T'),
    'тестирование': ('testing', 'T'),
    'st': ('testing', 'T'),
    'ift': ('testing', 'T'),
    'uat': ('testing', 'T'),
    'пси': ('testing', 'T'),
    'проверено на ифт/гот': ('testing', 'T'),
    'in discovery': ('testing', 'T'),
    # finished
    'done': ('finished', None),
    'resolved': ('finished', None),
    'closed': ('finished', None),
    'закрыт': ('finished', None),
    'закрыты': ('finished', None),
    'cancelled': ('finished', None),
}

def map_status(status_name):
    """Возвращает (category, phase) для имени статуса."""
    if not status_name:
        return ('unknown', None)
    key = str(status_name).strip().lower()
    return STATUS_MAP.get(key, ('unknown', None))

# === Парсинг JSON-ответа MCP ===

def parse_iso(s):
    """Парсит ISO 8601 с offset (например '2026-02-13T18:24:19.841+0300')."""
    if not s:
        return None
    if re.search(r'[+-]\d{4}$', s):
        s = s[:-2] + ':' + s[-2:]
    try:
        return datetime.fromisoformat(s)
    except (ValueError, TypeError):
        return None

def extract_epic(fields):
    """Эпик: customfield_11400 (для ASFC) → fallback на issuelinks 'Implement in' (для CRSIGMA)."""
    cf = fields.get('customfield_11400')
    if cf and isinstance(cf, str) and re.match(r'^[A-Z]+-\d+$', cf):
        return {'key': cf, 'name': None, 'source': 'customfield_11400'}

    for link in fields.get('issuelinks', []) or []:
        link_type = (link.get('type') or {}).get('outward', '')
        if link_type == 'Implement in' and 'outwardIssue' in link:
            outward = link['outwardIssue']
            return {
                'key': outward.get('key'),
                'name': (outward.get('fields') or {}).get('summary'),
                'source': 'issuelinks.Implement_in',
            }

    return {'key': None, 'name': None, 'source': None}

def extract_team(fields, assignee_obj):
    """Команда: customfield_22200 (МАССИВ СТРОК!) → fallback на assignee."""
    cf = fields.get('customfield_22200')
    if isinstance(cf, list) and len(cf) > 0:
        first = cf[0]
        # Может быть строкой типа "PALM.CSP.K7M"
        if isinstance(first, str) and first.strip():
            return {'value': first.strip(), 'source': 'customfield_22200'}
        # Или объектом — на всякий случай
        if isinstance(first, dict):
            val = first.get('value') or first.get('name')
            if val:
                return {'value': str(val), 'source': 'customfield_22200'}

    display = (assignee_obj or {}).get('displayName') or (assignee_obj or {}).get('display_name')
    if display:
        return {'value': display, 'source': 'assignee_fallback'}

    return {'value': None, 'source': None}

def compute_lead_time(fields):
    """Календарные дни от created до resolutiondate (если есть) или до now."""
    created = parse_iso(fields.get('created'))
    if not created:
        return None
    resolved = parse_iso(fields.get('resolutiondate'))
    end = resolved if resolved else datetime.now(timezone.utc).astimezone()
    return (end - created).days

def extract_jira_fields(response):
    """Главная функция — извлекает все нужные поля из ответа jira_get_issue.
    
    `response` — это либо весь JSON ответа, либо поле `fields` напрямую.
    Возвращает структуру для записи в task.jira согласно CONTRACT.md."""
    if not response:
        return {'found': False, 'error': 'empty response'}

    # Некоторые MCP возвращают поля плоско, некоторые в .fields
    fields = response.get('fields', response) if isinstance(response, dict) else {}
    if not fields:
        return {'found': False, 'error': 'no fields in response'}

    status_name = (fields.get('status') or {}).get('name')
    category, phase = map_status(status_name)

    assignee = fields.get('assignee') or {}
    reporter = fields.get('reporter') or {}

    return {
        'found': True,
        'summary': fields.get('summary'),
        'status': status_name,
        'status_category': category,
        'phase': phase,
        'issue_type': (fields.get('issuetype') or fields.get('issue_type') or {}).get('name'),
        'project': (fields.get('project') or {}).get('key'),
        'priority': (fields.get('priority') or {}).get('name'),
        'labels': fields.get('labels', []) or [],
        'created': fields.get('created'),
        'updated': fields.get('updated'),
        'resolutiondate': fields.get('resolutiondate'),
        'assignee': assignee.get('displayName') or assignee.get('display_name'),
        'reporter': reporter.get('displayName') or reporter.get('display_name'),
        'epic': extract_epic(fields),
        'team': extract_team(fields, assignee),
        'lead_time_days': compute_lead_time(fields),
        'fetched_at': now_iso(),
    }

def now_iso():
    return datetime.now(timezone.utc).astimezone().isoformat()

# === Главный entry-point: merge-batch ===

def merge_batch(enriched_path='pipeline/enriched.json'):
    """Читает JSON-батч из stdin (формат [{cr_key, jira: {...}}]),
    мерджит в enriched.json. Используется так:
    
        echo '[{"cr_key": "CRSIGMA-26516", "jira": {...}}]' | python3 helper.py merge-batch
    """
    batch_text = sys.stdin.read()
    batch = json.loads(batch_text)

    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    by_key = {item['cr_key']: item['jira'] for item in batch}
    
    updated_count = 0
    for task in enriched['tasks']:
        if task['cr_key'] in by_key:
            task['jira'] = by_key[task['cr_key']]
            updated_count += 1

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Merged {updated_count} tasks into {enriched_path}")

def merge_epics(enriched_path='pipeline/enriched.json'):
    """Читает epics-батч из stdin, мерджит в enriched.epics."""
    epics_text = sys.stdin.read()
    epics_data = json.loads(epics_text)

    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    enriched['epics'] = epics_data

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Saved {len(epics_data)} epics into {enriched_path}")

def finalize(enriched_path='pipeline/enriched.json'):
    """Обновить metadata: добавить jira-enricher в skills_completed, проставить enriched_at."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    enriched['metadata']['enriched_at'] = now_iso()
    completed = enriched['metadata'].setdefault('skills_completed', [])
    if 'jira-enricher' not in completed:
        completed.append('jira-enricher')

    # Сводная статистика
    tasks = enriched['tasks']
    stats = {
        'tasks_total': len(tasks),
        'tasks_found': sum(1 for t in tasks if (t.get('jira') or {}).get('found')),
        'tasks_not_found': sum(1 for t in tasks if t.get('jira') and not t['jira'].get('found')),
        'tasks_not_processed': sum(1 for t in tasks if t.get('jira') is None),
        'epics_unique': len(enriched.get('epics', [])),
    }
    enriched['metadata']['jira_stats'] = stats

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(json.dumps(stats, ensure_ascii=False))

def write_step2_markdown(enriched_path='pipeline/enriched.json',
                          md_path='pipeline/step-2-after-jira-enricher.md'):
    """Создать читаемый snapshot после jira-enricher."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    tasks = enriched['tasks']
    epics = enriched.get('epics', [])

    md = []
    md.append("# Снимок после jira-enricher\n")
    md.append(f"**Дата:** {enriched['metadata'].get('enriched_at')}\n")
    md.append(f"**Задач из плана:** {len(tasks)}\n")

    found = sum(1 for t in tasks if (t.get('jira') or {}).get('found'))
    not_found = sum(1 for t in tasks if t.get('jira') and not t['jira'].get('found'))
    md.append(f"**Найдено в Jira:** {found}\n")
    md.append(f"**Не найдено:** {not_found}\n")

    # Сводка по категориям
    md.append("\n## Сводка по статусам\n")
    md.append("| Категория | Количество |")
    md.append("|-----------|------------|")
    from collections import Counter
    cats = Counter(
        (t.get('jira') or {}).get('status_category', 'no_data')
        for t in tasks
    )
    for cat, n in cats.most_common():
        md.append(f"| {cat} | {n} |")

    # Таблица задач
    md.append("\n## Задачи\n")
    md.append("| # | CR | Статус | Фаза | Эпик | Команда | Lead time |")
    md.append("|---|-----|--------|------|------|---------|-----------|")
    for i, task in enumerate(tasks, 1):
        j = task.get('jira') or {}
        status = j.get('status', '—')
        phase = j.get('phase') or '—'
        epic = j.get('epic') or {}
        epic_str = '—'
        if epic.get('key'):
            ename = epic.get('name') or ''
            ename = (ename[:25] + '…') if len(ename) > 25 else ename
            epic_str = f"{epic['key']} {ename}".strip()
        team = (j.get('team') or {}).get('value', '—') or '—'
        lt = j.get('lead_time_days', '—')
        lt_str = f"{lt} д" if isinstance(lt, int) else '—'
        md.append(f"| {i} | {task['cr_key']} | {status} | {phase} | {epic_str} | {team} | {lt_str} |")

    # Эпики
    if epics:
        md.append(f"\n## Уникальные эпики ({len(epics)})\n")
        md.append("| Эпик | Имя | Задач из плана | Всего дочерних |")
        md.append("|------|-----|----------------|-----------------|")
        for e in epics:
            ename = (e.get('name') or '')[:40]
            from_plan = len(e.get('tasks_from_plan', []))
            total = e.get('children_count_total', '—')
            md.append(f"| {e['key']} | {ename} | {from_plan} | {total} |")

    md.append("\n## Следующий шаг\n")
    md.append("Запустить `timing-analyzer` для расчёта факта А/Р/Т (опционально).")
    md.append("Или сразу `report-builder` для финального отчёта.")

    with open(md_path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(md))

# === Entry point ===

if __name__ == '__main__':
    cmd = sys.argv[1] if len(sys.argv) > 1 else 'help'
    if cmd == 'merge-batch':
        merge_batch()
    elif cmd == 'merge-epics':
        merge_epics()
    elif cmd == 'finalize':
        finalize()
    elif cmd == 'write-step2':
        write_step2_markdown()
    else:
        print(f"Usage: python3 helper.py [merge-batch|merge-epics|finalize|write-step2]")
        sys.exit(1)
```

**Главное про этот код:**
- `extract_jira_fields(response)` — главная функция извлечения, **проверена на реальных JSON-ответах** для CRSIGMA-26516 (эпик через issuelinks) и ASFC-67203 (эпик через customfield_11400)
- `customfield_22200` обрабатывается как **массив строк** `["PALM.CSP.K7M"]`
- Entry point — это **CLI с подкомандами** (`merge-batch`, `merge-epics`, `finalize`, `write-step2`). Агент вызывает их через bash.
- helper.py **никогда не вызывает MCP** — это невозможно из Python в окружении GigaCode

## 8. Steps — как агент работает в чате

### Step 1. Валидация

Через bash:

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
print('skills:', d['metadata']['skills_completed'])
print('tasks:', len(d['tasks']))
print('keys:', [t['cr_key'] for t in d['tasks']])
"
```

Проверить:
- Файл существует
- `"excel-parser"` в `skills_completed`
- Массив `tasks` не пустой

Если нет — сообщить пользователю запустить `excel-parser`, завершить.

Сохранить список cr_keys для обработки.

### Step 2. Обработать задачи батчами по 5

**Главный цикл скилла.** Для каждых 5 задач:

#### Step 2.1. Для каждой задачи в батче

Агент в чате делает **нативный tool call** (это НЕ Python):

```
Tool: jira_get_issue
key = <cr_key>
fields = "summary,issuetype,status,project,created,updated,resolutiondate,reporter,assignee,priority,labels,description,parent,customfield_11400,customfield_22200,issuelinks"
```

Агент получает JSON-ответ. Извлекает поля **глазами** (читая JSON в контексте) и формирует объект для накопления.

Точная логика извлечения — реализована в `extract_jira_fields()` в helper.py. **Но во время самого tool call helper.py недоступен** — агент извлекает поля вручную, потом передаёт батч в helper для проверки/перезаписи.

**Возможные ошибки:**
- 404 / "issue not found" → `jira = {found: false, error: "404"}`
- timeout → `jira = {found: false, error: "timeout"}`
- Любая другая ошибка → записать `error: <текст>`, продолжить

#### Step 2.2. После 5 задач — мерджим батч

Агент формирует JSON-массив батча в формате:

```json
[
  {"cr_key": "CRSIGMA-26516", "jira": {...полный результат...}},
  {"cr_key": "CRSIGMA-23749", "jira": {...}},
  ...
]
```

И вызывает через bash:

```bash
echo '<JSON-батч из 5 задач>' | python3 ~/.gigacode/skills/jira-enricher/helper.py merge-batch
```

helper.py читает stdin, мерджит в `pipeline/enriched.json`. После этого следующий батч.

**Важно:** агент не должен пытаться передать JSON через аргумент командной строки (длина ограничена). Только через stdin (`echo ... | python3 helper.py merge-batch`).

#### Step 2.3. Прогресс

После каждого батча сообщить пользователю: "Обработано 5/28, 10/28, ..."

### Step 3. Опционально — догрузить имена эпиков

После всех задач у нас есть массив с `task.jira.epic.key`. Часть эпиков имеет `name = null` (те которые пришли через `customfield_11400`).

Собрать уникальные эпики без имени. Если их ≤15:
- Для каждого сделать tool call `jira_get_issue` с `fields="summary"`
- После всех — отправить batch update в helper

Если >15 — пропустить (имена можно достать в timing-analyzer или вручную).

### Step 4. Подсчёт дочерних задач для каждого эпика

Собрать массив **уникальных** эпиков из всех задач. Для каждого — нативный tool call:

```
Tool: jira_search
jql = '"Epic Link" = <epic_key>'
fields = "summary,status,issuetype"
maxResults = 100
```

Записать `children_count_total = len(result.issues)`.

Собрать в массив:

```json
[
  {
    "key": "ASFC-57216",
    "name": "ЦКП.ПГ-1 универсальная задача",
    "tasks_from_plan": ["CRSIGMA-26516", "ASFC-63820"],
    "children_count_total": 38,
    "fetched_at": "<now>"
  },
  ...
]
```

И отправить через bash:

```bash
echo '<epics-array>' | python3 ~/.gigacode/skills/jira-enricher/helper.py merge-epics
```

### Step 5. Финализация

```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py finalize
```

helper.py обновит:
- `metadata.enriched_at`
- `metadata.skills_completed` добавит `"jira-enricher"`
- `metadata.jira_stats` (сводная статистика)

Распечатает JSON со статистикой в stdout — агент покажет её пользователю.

### Step 6. Создать markdown-снимок

```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py write-step2
```

Создаст `pipeline/step-2-after-jira-enricher.md` с читаемой таблицей.

### Step 7. Сводка пользователю

В чат:
- Обработано задач: X из Y (M не найдены)
- Уникальных эпиков: N
- Команды: K через customfield_22200, L через assignee_fallback
- Созданы файлы:
  - `pipeline/enriched.json` (обновлён)
  - `pipeline/step-2-after-jira-enricher.md` (читаемый снимок)
- Следующий шаг: запустите `timing-analyzer` (для факта А/Р/Т) или сразу `report-builder`

## 9. КРИТИЧНО: формат вызова MCP

### ПРАВИЛЬНО (нативный tool call агента)

```
Шаг 2.1. Для каждого cr_key в текущем батче:

Сделать tool call:
  Tool: jira_get_issue
  Параметры:
    key = <cr_key>
    fields = "summary,issuetype,status,project,created,updated,resolutiondate,
              reporter,assignee,priority,labels,description,parent,
              customfield_11400,customfield_22200,issuelinks"

Получить JSON-ответ в контекст. Извлечь поля (status.name, customfield_22200[0],
customfield_11400 или issuelinks[].outwardIssue.key, и т.д.) — см. правила в SPEC.md разделе 7.

Накопить результат в текстовом виде для последующей передачи в helper.merge-batch.
```

### ❌ НЕПРАВИЛЬНО (попытка засунуть MCP в Python)

```python
# Это НЕ работает в Python-окружении GigaCode!
from mcp_atlassian import jira_get_issue
result = jira_get_issue(key="...")
# NameError: name 'jira_get_issue' is not defined
```

```python
# Тоже не работает
result = mcp__Atlassian__jira_get_issue(key="...")
# NameError
```

### Правило для агента

MCP tools (`jira_get_issue`, `jira_search`) вызываются **только** через нативный механизм tool_use агента в чате. Python (через `python3` в bash) используется **только** для:
- Чтения/записи `enriched.json`
- Парсинга JSON-ответов которые **уже получены** агентом и переданы через stdin
- Расчётов (даты, lead time)
- Генерации markdown

**Граница:** результат tool call → агент видит в контексте → передаёт текстом в bash через `echo '...' | python3 helper.py`.

## 10. Файлы которые скилл может создавать

| Файл | Назначение |
|------|------------|
| `pipeline/enriched.json` | Перезаписывается с дополнениями |
| `pipeline/step-2-after-jira-enricher.md` | Читаемый snapshot |
| `helper.py` (в `~/.gigacode/skills/jira-enricher/`) | CLI с подкомандами |

### Запрещённые файлы

То же что в `excel-parser`: только `helper.py`, никаких `main.py`, `process.py`, `run_*.py`, `generate_*.py`, `__pycache__`, виртуальных окружений.

## 11. Guardrails

- READ-ONLY для Jira (только `jira_get_issue` и `jira_search`)
- Точные `fields=` из раздела 5, не `fields="*"`
- НЕ запрашивать changelog
- НЕ ходить за дочерними задачами эпика дальше counter
- Не падать на одной ошибке Jira — продолжать
- НЕ пытаться вызывать MCP из Python-скрипта
- НЕ передавать большие batch через аргумент CLI — только через stdin

## 12. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| `pipeline/enriched.json` не существует | Попросить запустить excel-parser, завершить |
| `metadata.skills_completed` не содержит `excel-parser` | То же |
| Задача 404 в Jira | `task.jira = {found: false, error: "404"}`, продолжить |
| Jira timeout | Записать error, продолжить со следующей задачей |
| Статус не в mapping | `category = "unknown"`, `phase = null`, в `jira_stats` отметить |
| `customfield_11400` пустой и `issuelinks` без "Implement in" | `epic = {key: null, name: null, source: null}` |
| `customfield_22200` пустой и `assignee` пустой | `team = {value: null, source: null}` |
| `customfield_22200` пришёл как объект, не массив | helper.extract_team пытается оба формата (см. код в разделе 7) |
| Эпик к которому ссылается несколько задач плана | Собрать в `epics[].tasks_from_plan` все ключи |
| Дочерних у эпика 0 | `children_count_total = 0`, валидно |
| `jira_search` вернул >100 (limit) | Записать `children_count_total = 100`, в metadata пометка |
| Повторный запуск (idempotency) | Перезаписать поля jira и epics, не дублировать `skills_completed` |
| Батч из 5 задач, но 5-я ещё не дошла (конец) | Сделать финальный merge-batch для оставшихся (3-4 задачи) |

## 13. Антипаттерны

### Критические (повторение = провал скилла)

- **Попытка вызвать MCP из Python** — `mcp__Atlassian__jira_get_issue(...)`, `from mcp_atlassian import ...`, `def fetch_issue(...)` с псевдокодом. Это **физически невозможно** в окружении GigaCode CLI. Все tool calls — только нативные через чат.
- **Использовать `fields="*"`** — раздувает контекст. Точный список из раздела 5.
- **Запрашивать `expand=changelog`** — это работа timing-analyzer, не сейчас.
- **Создавать `main.py`, `process.py`** и подобные — только `helper.py`.
- **Передавать batch через аргумент CLI** — длина ограничена. Только stdin: `echo '...' | python3 helper.py merge-batch`.
- **Парсить ответ MCP внутри bash через `python3 -c '...'` с подставленным JSON** — слишком хрупко, спецсимволы ломают. Только через stdin.

### Обычные

- Падать на одной 404 вместо записи `found: false` и продолжения
- Параллельные вызовы MCP — только последовательно
- Перезаписывать `enriched.json` без `ensure_ascii=False` — кириллица превратится в `\u...`
- Хранить сырой JSON ответа MCP в `enriched.json` — только извлечённые поля
- Забыть финальный `finalize` или `write-step2` — пользователь не увидит snapshot

## 14. Критерий успеха

После запуска:
1. `pipeline/enriched.json` перезаписан, валиден
2. У каждой задачи поле `jira` заполнено (либо `found: true` с данными, либо `found: false`)
3. Массив `epics` непустой (если у задач есть эпики)
4. `metadata.skills_completed` содержит `["excel-parser", "jira-enricher"]`
5. `metadata.jira_stats` присутствует
6. Создан `pipeline/step-2-after-jira-enricher.md` с читаемой таблицей
7. В чате сводка с реальными цифрами

## 15. Что отложено

- v4: догрузка плановых дат ИФТ/ПСИ/ПРОМ из customfields 24300/29500/13700/22601/23703 (отдельный скилл `dates-enricher`)
- v6: расширенный поиск дочерних задач эпиков — получать **каждую** дочернюю задачу с её планом, не только counter
