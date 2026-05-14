# SPEC: jira-enricher (v3.2)

> Второй скилл pipeline. Читает `pipeline/enriched.json`, для каждой задачи делает `jira_get_issue` БЕЗ changelog, дополняет данными из Jira (статус, эпик, поток, lead time). Затем агрегирует уникальные эпики из всех задач и через `jira_search` подсчитывает дочерние задачи каждого эпика.
>
> **Изменения v3.2 vs v3.1:**
> 1. Добавлена явная функция `aggregate_epics()` — собирает уникальные эпики из `task.jira.epic` в `enriched.epics[]`. В v3.1 GigaCode сгенерил скилл который **не собирал** эпики в массив, из-за этого секция "Срез по эпикам" в финальном отчёте была пустой.
> 2. Добавлена функция `update_epic_children(key, count)` — обновляет counter дочерних задач для одного эпика. Используется после каждого `jira_search`.
> 3. Колонка в `pipeline/step-2-after-jira-enricher.md` — "Поток", не "Команда" (консистентность с финальным отчётом v3.2).
> 4. Все функции работы с эпиками протестированы на mock-данных — идемпотентны (повторный запуск не теряет counters).

---

## 1. Контекст: что пошло не так в v3.1

**Что работало:** агент успешно делал tool calls, helper.merge_batch корректно мерджил `task.jira` для каждой задачи. 28 из 28 задач имели заполненный `task.jira.epic.key`.

**Что не работало:** массив `enriched.epics[]` оставался **пустым**. Причина — после `merge_batch` никто не вызывал агрегацию эпиков. Функция `merge_epics()` была реализована но ожидала массив эпиков через stdin, а агент его не формировал (не было такого шага в SKILL.md).

**Решение v3.2:** Явная команда `aggregate-epics` которая читает уже записанные `task.jira.epic` и собирает их в `enriched.epics[]`. Вызывается агентом одной командой после всех `merge_batch`.

## 2. Архитектура агент↔Python (без изменений с v3.1)

```
АГЕНТ в чате:
  - Делает НАТИВНЫЕ tool calls (jira_get_issue, jira_search)
  - Получает JSON в контекст
  - Извлекает поля глазами
  - Передаёт batch в helper через bash + stdin

PYTHON через bash (helper.py):
  - НЕ делает MCP-вызовов (это NameError)
  - Принимает данные через stdin
  - Парсит, мерджит, пишет в pipeline/enriched.json
```

**Запрещено для агента:**
```python
result = mcp__Atlassian__jira_get_issue(...)  # NameError
from mcp_atlassian import jira_get_issue       # модуля нет
```

## 3. Место в pipeline

```
1. excel-parser              → pipeline/enriched.json (план)
2. jira-enricher  ◄── (этот скилл)
3. timing-analyzer
4. report-builder
```

## 4. Цели

- Прочитать `pipeline/enriched.json`, валидировать что `excel-parser` отработал
- Для каждой задачи с `cr_key` сделать **один** `jira_get_issue` БЕЗ `expand=changelog`
- Извлечь: статус, фаза, эпик, поток (из `customfield_22200`), lead time
- После всех задач **агрегировать уникальные эпики** в `enriched.epics[]` (НОВОЕ в v3.2)
- Для каждого эпика — `jira_search('"Epic Link" = <key>')` для подсчёта дочерних
- Обновить `metadata.skills_completed`
- Создать `pipeline/step-2-after-jira-enricher.md` (с колонкой "Поток", не "Команда")

## 5. Анти-цели

- **НЕ** запрашивать changelog — это работа `timing-analyzer`
- **НЕ** считать `phase_days`
- **НЕ** генерировать финальный markdown
- **НЕ** читать Excel

## 6. Формула вызова MCP

### Для каждой задачи

```
Tool: jira_get_issue
key = <cr_key>
fields = "summary,issuetype,status,project,created,updated,resolutiondate,
          reporter,assignee,priority,labels,description,parent,
          customfield_11400,customfield_22200,issuelinks"
```

КРИТИЧНО:
- **НЕ** передавать `expand=changelog` (это работа следующего скилла)
- **НЕ** использовать `fields="*"` (раздувает контекст)
- Это нативный tool call агента

### Для каждого уникального эпика

После агрегации эпиков:

```
Tool: jira_search
jql = '"Epic Link" = <epic_key>'
fields = "summary,status,issuetype"
maxResults = 100
```

Результат — `len(issues)` это `children_count_total`.

## 7. Mapping статуса → category → phase

Используется единый mapping из CONTRACT.md.

## 8. Извлечение полей — правила (из реальных данных)

### Эпик (порядок поиска)

1. **`fields.customfield_11400`** — для ASFC-задач строка типа `ASFC-65543`
2. **`fields.issuelinks`** — для CRSIGMA-задач, найти первую с `type.outward == "Implement in"`, взять `outward_issue.key` (с подчёркиванием!)

Если ни 1, ни 2 → `epic = {key: null, name: null, source: null}`.

### Поток (НЕ "Команда")

1. **`fields.customfield_22200`** — это **массив строк** типа `["PALM.CSP.K7M"]`. Берём первый элемент. Это техническая метка потока, не имя команды.
2. Если пусто — fallback на `fields.assignee.displayName` с пометкой `assignee_fallback`
3. Если и assignee пуст — `team = {value: null, source: null}`

## 9. Готовый код helper.py — используйте как основу

```python
# helper.py для jira-enricher v3.2

import json
import sys
import re
import os
from datetime import datetime, timezone

# === Mapping статусов ===

STATUS_MAP = {
    'backlog': ('not_started', None),
    'to do': ('not_started', None),
    'открыта': ('not_started', None),
    'new': ('analysis', 'A'),
    'need info': ('analysis', 'A'),
    'analysis': ('analysis', 'A'),
    'анализ': ('analysis', 'A'),
    'in progress': ('development', 'R'),
    'разработка': ('development', 'R'),
    'готов к разработке': ('development', 'R'),
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
    'done': ('finished', None),
    'resolved': ('finished', None),
    'closed': ('finished', None),
    'закрыт': ('finished', None),
    'закрыты': ('finished', None),
    'cancelled': ('finished', None),
}

def map_status(status_name):
    if not status_name:
        return ('unknown', None)
    return STATUS_MAP.get(str(status_name).strip().lower(), ('unknown', None))

def parse_iso(s):
    if not s:
        return None
    s = str(s)
    if re.search(r'[+-]\d{4}$', s):
        s = s[:-2] + ':' + s[-2:]
    try:
        return datetime.fromisoformat(s)
    except (ValueError, TypeError):
        return None

def now_iso():
    return datetime.now(timezone.utc).astimezone().isoformat()

# === Извлечение полей из ответа MCP ===

def extract_epic(fields):
    """Эпик: customfield_11400 (ASFC) → fallback на issuelinks 'Implement in' (CRSIGMA)."""
    cf = fields.get('customfield_11400')
    if cf and isinstance(cf, str) and re.match(r'^[A-Z]+-\d+$', cf):
        return {'key': cf, 'name': None, 'source': 'customfield_11400'}

    for link in fields.get('issuelinks', []) or []:
        link_type = (link.get('type') or {}).get('outward', '')
        if link_type == 'Implement in':
            # ВАЖНО: в Сбер-MCP поле называется outward_issue (с подчёркиванием)
            outward = link.get('outward_issue') or link.get('outwardIssue')
            if outward:
                return {
                    'key': outward.get('key'),
                    'name': (outward.get('fields') or {}).get('summary'),
                    'source': 'issuelinks.Implement_in',
                }

    return {'key': None, 'name': None, 'source': None}

def extract_team(fields, assignee_obj):
    """Поток (НЕ команда): customfield_22200 (массив строк PALM.*) → fallback на assignee."""
    cf = fields.get('customfield_22200')
    if isinstance(cf, list) and len(cf) > 0:
        first = cf[0]
        if isinstance(first, str) and first.strip():
            return {'value': first.strip(), 'source': 'customfield_22200'}
        if isinstance(first, dict):
            val = first.get('value') or first.get('name')
            if val:
                return {'value': str(val), 'source': 'customfield_22200'}

    display = (assignee_obj or {}).get('displayName') or (assignee_obj or {}).get('display_name')
    if display:
        return {'value': display, 'source': 'assignee_fallback'}

    return {'value': None, 'source': None}

def compute_lead_time(fields):
    created = parse_iso(fields.get('created'))
    if not created:
        return None
    resolved = parse_iso(fields.get('resolutiondate'))
    end = resolved if resolved else datetime.now(timezone.utc).astimezone()
    return (end - created).days

def extract_jira_fields(response):
    """Главная функция извлечения. Принимает ответ jira_get_issue."""
    if not response:
        return {'found': False, 'error': 'empty response'}

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

# === Главные CLI entry-points ===

def merge_batch(enriched_path='pipeline/enriched.json'):
    """Принимает batch [{cr_key, jira}] через stdin, мерджит task.jira."""
    batch_text = sys.stdin.read()
    batch = json.loads(batch_text)

    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    by_key = {item['cr_key']: item['jira'] for item in batch}
    updated = 0
    for task in enriched['tasks']:
        if task['cr_key'] in by_key:
            task['jira'] = by_key[task['cr_key']]
            updated += 1

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Merged {updated} tasks into {enriched_path}")

def aggregate_epics(enriched_path='pipeline/enriched.json'):
    """НОВОЕ В v3.2. Собирает уникальные эпики из task.jira.epic в enriched.epics[].
    
    Идемпотентна: повторный вызов сохраняет существующие children_count_total.
    Вызывается ОДИН РАЗ после всех merge_batch, до jira_search для подсчёта детей.
    """
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    # Существующие counters сохраняем (могут быть проставлены ранее)
    existing = {e['key']: e for e in (enriched.get('epics') or []) if e.get('key')}

    epics_by_key = {}
    for task in enriched.get('tasks', []):
        epic = (task.get('jira') or {}).get('epic') or {}
        key = epic.get('key')
        if not key:
            continue

        if key not in epics_by_key:
            # children_count_total: берём из existing если был, иначе None
            # (None != 0 — отличаем "не посчитано" от "0 детей")
            prev_count = existing.get(key, {}).get('children_count_total')
            epics_by_key[key] = {
                'key': key,
                'name': epic.get('name'),
                'tasks_from_plan': [],
                'children_count_total': prev_count,
                'fetched_at': now_iso(),
            }

        # Обновить имя если в существующем None а в новом есть
        if not epics_by_key[key].get('name') and epic.get('name'):
            epics_by_key[key]['name'] = epic.get('name')

        if task['cr_key'] not in epics_by_key[key]['tasks_from_plan']:
            epics_by_key[key]['tasks_from_plan'].append(task['cr_key'])

    # Сортировка для детерминизма (по убыванию задач, потом по ключу)
    enriched['epics'] = sorted(epics_by_key.values(),
                                key=lambda e: (-len(e['tasks_from_plan']), e['key']))

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Aggregated {len(enriched['epics'])} unique epics from tasks")

def update_epic_children(epic_key, children_count, enriched_path='pipeline/enriched.json'):
    """НОВОЕ В v3.2. Обновить children_count_total для одного эпика.
    
    Использование:
        python3 helper.py update-epic-children ASFC-57216 38
    
    Вызывается агентом после каждого jira_search для эпика.
    """
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    found = False
    for epic in enriched.get('epics', []):
        if epic.get('key') == epic_key:
            epic['children_count_total'] = int(children_count)
            epic['fetched_at'] = now_iso()
            found = True
            break

    if not found:
        print(f"ERROR: epic {epic_key} not in enriched.epics. Did you forget aggregate-epics?", 
              file=sys.stderr)
        sys.exit(1)

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Updated {epic_key}: children_count_total = {children_count}")

def list_epics_to_count(enriched_path='pipeline/enriched.json'):
    """Вывести JSON-массив ключей эпиков для которых нужно сделать jira_search.
    
    Это эпики у которых children_count_total = None (ещё не посчитано).
    """
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    keys = [e['key'] for e in enriched.get('epics', [])
            if e.get('key') and e.get('children_count_total') is None]
    print(json.dumps(keys, ensure_ascii=False))

def list_epics_without_names(enriched_path='pipeline/enriched.json', max_count=15):
    """Вывести JSON-массив ключей эпиков без имени (для опционального догруза).
    
    Возвращает только если их меньше max_count — иначе пустой массив (не догружаем).
    """
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    keys = [e['key'] for e in enriched.get('epics', [])
            if e.get('key') and not e.get('name')]
    if len(keys) > max_count:
        print(json.dumps([], ensure_ascii=False))
    else:
        print(json.dumps(keys, ensure_ascii=False))

def update_epic_name(epic_key, name, enriched_path='pipeline/enriched.json'):
    """Опционально обновить имя эпика (если получили его отдельным jira_get_issue)."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    for epic in enriched.get('epics', []):
        if epic.get('key') == epic_key:
            epic['name'] = name
            break

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Updated name for {epic_key}: {name}")

def finalize(enriched_path='pipeline/enriched.json'):
    """Финальная проставка metadata."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    enriched['metadata']['enriched_at'] = now_iso()
    completed = enriched['metadata'].setdefault('skills_completed', [])
    if 'jira-enricher' not in completed:
        completed.append('jira-enricher')

    tasks = enriched['tasks']
    epics = enriched.get('epics', [])
    stats = {
        'tasks_total': len(tasks),
        'tasks_found': sum(1 for t in tasks if (t.get('jira') or {}).get('found')),
        'tasks_not_found': sum(1 for t in tasks if t.get('jira') and not t['jira'].get('found')),
        'tasks_not_processed': sum(1 for t in tasks if t.get('jira') is None),
        'epics_unique': len(epics),
        'epics_with_children_count': sum(1 for e in epics if e.get('children_count_total') is not None),
        'epics_without_children_count': sum(1 for e in epics if e.get('children_count_total') is None),
    }
    enriched['metadata']['jira_stats'] = stats

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(json.dumps(stats, ensure_ascii=False))

def write_step2_markdown(enriched_path='pipeline/enriched.json',
                          md_path='pipeline/step-2-after-jira-enricher.md'):
    """Создать snapshot. КОЛОНКА 'Поток', НЕ 'Команда'."""
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

    # Таблица задач — колонка "Поток" (НЕ "Команда")
    md.append("\n## Задачи\n")
    md.append("| # | CR | Статус | Фаза | Эпик | Поток | Lead time |")
    md.append("|---|-----|--------|------|------|-------|-----------|")
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
        team_val = (j.get('team') or {}).get('value', '—') or '—'
        lt = j.get('lead_time_days', '—')
        lt_str = f"{lt} д" if isinstance(lt, int) else '—'
        md.append(f"| {i} | {task['cr_key']} | {status} | {phase} | {epic_str} | {team_val} | {lt_str} |")

    # Эпики
    if epics:
        md.append(f"\n## Уникальные эпики ({len(epics)})\n")
        md.append("| Эпик | Имя | Задач из плана | Всего дочерних |")
        md.append("|------|-----|----------------|-----------------|")
        for e in epics:
            ename = (e.get('name') or '')[:40]
            from_plan = len(e.get('tasks_from_plan', []))
            total = e.get('children_count_total')
            total_str = str(total) if total is not None else '—'
            md.append(f"| {e['key']} | {ename} | {from_plan} | {total_str} |")
    else:
        md.append("\n## Эпики\n*Эпики не агрегированы. Возможно агент пропустил шаг aggregate-epics.*")

    md.append("\n## Следующий шаг\n")
    md.append("Запустить `timing-analyzer` для расчёта факта А/Р/Т (опционально).")
    md.append("Или сразу `report-builder` для финального отчёта.")

    with open(md_path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(md))

    print(f"Created {md_path}")

# === Main ===

if __name__ == '__main__':
    cmd = sys.argv[1] if len(sys.argv) > 1 else 'help'
    if cmd == 'merge-batch':
        merge_batch()
    elif cmd == 'aggregate-epics':
        aggregate_epics()
    elif cmd == 'list-epics-to-count':
        list_epics_to_count()
    elif cmd == 'list-epics-without-names':
        list_epics_without_names()
    elif cmd == 'update-epic-children':
        if len(sys.argv) < 4:
            print("Usage: python3 helper.py update-epic-children <epic_key> <count>", file=sys.stderr)
            sys.exit(1)
        update_epic_children(sys.argv[2], sys.argv[3])
    elif cmd == 'update-epic-name':
        if len(sys.argv) < 4:
            print("Usage: python3 helper.py update-epic-name <epic_key> <name>", file=sys.stderr)
            sys.exit(1)
        update_epic_name(sys.argv[2], sys.argv[3])
    elif cmd == 'finalize':
        finalize()
    elif cmd == 'write-step2':
        write_step2_markdown()
    else:
        print("Usage: python3 helper.py [merge-batch|aggregate-epics|list-epics-to-count|list-epics-without-names|update-epic-children KEY COUNT|update-epic-name KEY NAME|finalize|write-step2]")
        sys.exit(1)
```

**Главное про этот код:**
- `aggregate_epics()` — новая функция, собирает уникальные эпики из `task.jira.epic`. **Идемпотентна** — повторный вызов не теряет counters.
- `update_epic_children(key, count)` — обновляет один эпик. Принимает аргументы через CLI, не stdin.
- `list_epics_to_count` — выводит JSON-массив эпиков без counter (агент по ним делает jira_search)
- `list_epics_without_names` — выводит эпики без имени (≤15 → значит можно догрузить)
- Колонка в `write_step2_markdown` — **"Поток"**, не "Команда"
- `extract_epic` использует `outward_issue` с подчёркиванием (как в Сбер-MCP)

## 10. Steps — что делает агент в чате

### Step 1. Валидация

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
assert 'excel-parser' in d['metadata']['skills_completed'], 'excel-parser не отработал'
print(f'Tasks: {len(d[\"tasks\"])}')
print('keys:', [t['cr_key'] for t in d['tasks']])
"
```

Если файл не существует или excel-parser не в `skills_completed` — сообщить пользователю запустить предшественника.

### Step 2. Обработать задачи батчами по 5

Для каждых 5 cr_key из `tasks`:

**Step 2.1.** Для каждой задачи в батче — нативный tool call:
```
Tool: jira_get_issue
key = <cr_key>
fields = "summary,issuetype,status,project,created,updated,resolutiondate,reporter,assignee,priority,labels,description,parent,customfield_11400,customfield_22200,issuelinks"
```

Получить JSON, **глазами** извлечь поля (status.name, epic, team, и т.д.) — алгоритм согласно разделу 8 и `extract_jira_fields()` в helper.py.

**Step 2.2.** Сформировать batch:
```json
[
  {"cr_key": "CRSIGMA-26516", "jira": {найденные поля}},
  {"cr_key": "ASFC-58741", "jira": {...}},
  ...
]
```

**Step 2.3.** Передать в helper:
```bash
echo '<batch_json>' | python3 ~/.gigacode/skills/jira-enricher/helper.py merge-batch
```

**Step 2.4.** Прогресс пользователю каждый батч: "Обработано 5/28".

### Step 3. Агрегировать уникальные эпики (НОВОЕ в v3.2)

После того как **все задачи** обработаны (все батчи `merge-batch` выполнены):

```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py aggregate-epics
```

helper:
- Прочитает все `task.jira.epic`
- Соберёт уникальные эпики в `enriched.epics[]`
- В каждом эпике укажет какие задачи плана к нему относятся (`tasks_from_plan`)
- `children_count_total = None` (ещё не посчитано)

**Если этот шаг пропустить — массив `enriched.epics[]` останется пустым, и в финальном отчёте секция эпиков будет пустой.** Это критический шаг, обязательный.

### Step 4 (опционально). Догрузка имён эпиков

Получить список эпиков без имени:
```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py list-epics-without-names
```

Если список **не пустой** (значит их ≤15) — для каждого ключа:
- Tool call `jira_get_issue(key=<epic_key>, fields="summary")`
- Извлечь `fields.summary`
- Обновить: `python3 helper.py update-epic-name <epic_key> "<имя>"`

Если список пустой — пропустить шаг (либо все имена есть, либо эпиков >15).

### Step 5. Подсчёт дочерних задач для каждого эпика

Получить список эпиков для подсчёта:
```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py list-epics-to-count
```

Для каждого `epic_key` из списка:

**Step 5.1.** Tool call:
```
Tool: jira_search
jql = '"Epic Link" = <epic_key>'
fields = "summary,status,issuetype"
maxResults = 100
```

**Step 5.2.** Получить `len(result.issues)` = N.

**Step 5.3.** Обновить counter:
```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py update-epic-children <epic_key> <N>
```

### Step 6. Финализация

```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py finalize
```

helper обновит:
- `metadata.enriched_at`
- `metadata.skills_completed` добавит `"jira-enricher"`
- `metadata.jira_stats` (сводная статистика)

### Step 7. Создать markdown-снимок

```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py write-step2
```

Создаст `pipeline/step-2-after-jira-enricher.md` с колонкой **"Поток"** (не "Команда") и секцией уникальных эпиков с counter дочерних.

### Step 8. Сводка пользователю

В чат:
- Обработано задач: X из Y (M не найдены)
- Уникальных эпиков: N (M с counter, K без)
- Создан `pipeline/step-2-after-jira-enricher.md`
- Следующий шаг: запустите `timing-analyzer` (для факта А/Р/Т) или сразу `report-builder`

## 11. КРИТИЧНО: правильная последовательность шагов

```
для каждых 5 задач: tool calls jira_get_issue → echo batch | helper.py merge-batch
после всех:        helper.py aggregate-epics  ← БЕЗ ЭТОГО ШАГА epics[] ПУСТОЙ
для каждого эпика: tool call jira_search → helper.py update-epic-children <key> <count>
финал:             helper.py finalize → helper.py write-step2
```

**Главный антипаттерн v3.1:** скилл делал tool calls и merge-batch правильно, но **пропускал aggregate-epics** — массив `enriched.epics` оставался пустым. В v3.2 этот шаг явный и обязательный.

## 12. Файлы которые скилл может создавать

| Файл | Назначение |
|------|------------|
| `pipeline/enriched.json` | Перезаписывается с дополнениями (jira-поля + epics массив) |
| `pipeline/step-2-after-jira-enricher.md` | Читаемый snapshot с колонкой "Поток" |
| `helper.py` (в `~/.gigacode/skills/jira-enricher/`) | CLI с подкомандами |

### Запрещённые файлы

Только `helper.py`. Никаких `main.py`, `process.py`, `run_*.py`, `generate_*.py`, `__pycache__`, виртуальных окружений.

## 13. Guardrails

- READ-ONLY для Jira
- Точные `fields=` из раздела 6, не `fields="*"`
- НЕ запрашивать changelog
- НЕ ходить за дочерними задачами эпика дальше counter
- Не падать на одной ошибке Jira
- Колонка в step-2 — **"Поток"**, не "Команда"

## 14. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| `pipeline/enriched.json` не существует | Попросить запустить excel-parser, завершить |
| `excel-parser` не в `skills_completed` | То же |
| Задача 404 в Jira | `jira = {found: false, error: "404"}`, продолжить |
| Jira timeout | Записать error, продолжить со следующей задачей |
| Статус не в mapping | `category = "unknown"`, `phase = null` |
| `customfield_11400` пустой и нет issuelinks "Implement in" | `epic = {key: null, ...}` — эпика нет |
| `customfield_22200` пустой и assignee пустой | `team = {value: null, ...}` |
| `customfield_22200` пришёл как объект, не массив | `extract_team` пытается оба формата |
| Несколько задач указывают на один эпик | `aggregate-epics` соберёт уникальный, `tasks_from_plan` будет с обеими |
| Дочерних у эпика 0 | `children_count_total = 0`, валидно |
| `jira_search` вернул ошибку | НЕ вызывать update-epic-children для этого ключа — пометить в metadata |
| `jira_search` вернул >100 | `update-epic-children <key> 100`, в metadata jira_stats отметить |
| Повторный запуск (idempotency) | Поля перезаписываются, но `aggregate-epics` сохраняет существующие counters |
| `update-epic-children` для несуществующего эпика | helper падает с понятной ошибкой — значит aggregate-epics не вызывали |

## 15. Антипаттерны

### Критические (приводили к багам)

- **Пропустить шаг `aggregate-epics`** — массив `enriched.epics[]` останется пустым → секция эпиков в финальном отчёте будет пустой. ЭТО БЫЛ КОРНЕВОЙ БАГ v3.1.
- **Вызвать MCP из Python** — NameError
- **Использовать `fields="*"`** — раздувает контекст
- **Запрашивать `expand=changelog`** — это работа timing-analyzer
- **Передавать batch через CLI-аргумент** — длина ограничена, только stdin
- **Создавать `main.py`, `process.py`, run_*.py** — только `helper.py`
- **В step-2-after-jira-enricher.md колонка "Команда"** — должно быть "Поток"

### Обычные

- Падать на одной 404 вместо записи `found: false` и продолжения
- Параллельные вызовы MCP — только последовательно
- Перезаписывать `enriched.json` без `ensure_ascii=False`
- Хранить сырой JSON ответа MCP — только извлечённые поля

## 16. Критерий успеха

После запуска:
1. `pipeline/enriched.json` перезаписан, валиден
2. У каждой задачи поле `jira` заполнено
3. **Массив `epics` непустой** (если у задач есть эпики) — ЭТО КЛЮЧЕВАЯ ПРОВЕРКА для v3.2
4. У каждого эпика заполнено `tasks_from_plan` и `children_count_total`
5. `metadata.skills_completed` содержит `["excel-parser", "jira-enricher"]`
6. Создан `pipeline/step-2-after-jira-enricher.md` с колонкой "Поток"
7. В чате сводка с реальными цифрами

**Быстрая проверка после прогона:**
```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
print('Tasks:', len(d['tasks']))
print('Tasks with jira:', sum(1 for t in d['tasks'] if (t.get('jira') or {}).get('found')))
print('Epics:', len(d.get('epics', [])))  # ДОЛЖНО БЫТЬ > 0
for e in d.get('epics', [])[:3]:
    print(f'  {e[\"key\"]}: from_plan={len(e[\"tasks_from_plan\"])} children={e[\"children_count_total\"]}')
"
```

Если `Epics: 0` — значит шаг `aggregate-epics` пропущен. Запустить вручную:
```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py aggregate-epics
python3 ~/.gigacode/skills/jira-enricher/helper.py write-step2
```

## 17. Что отложено

- v4: догрузка плановых дат ИФТ/ПСИ/ПРОМ (отдельный скилл `dates-enricher` после jira-enricher)
- v6: получать **каждую** дочернюю задачу эпика с её планом, не только counter
