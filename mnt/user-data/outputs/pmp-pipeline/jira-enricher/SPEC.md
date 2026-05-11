# SPEC: jira-enricher

> Второй скилл pipeline. Читает `.cache/enriched.json`, для каждой задачи делает `jira_get_issue` БЕЗ changelog, дополняет данными из Jira (статус, эпик, команда, lead time). Дополнительно делает `jira_search` для подсчёта дочерних задач каждого уникального эпика.
>
> Это самый частый узкое место по времени (28 + ~10 вызовов MCP), но **не самый тяжёлый по контексту** — changelog не запрашивается.

## 1. Место в pipeline

```
1. excel-parser              → создал .cache/enriched.json
2. jira-enricher  ◄── (этот скилл)
3. timing-analyzer
4. report-builder
```

## 2. Цели

- Прочитать `.cache/enriched.json`, валидировать что предыдущий скилл отработал
- Для каждой задачи с `cr_key` сделать **один** `jira_get_issue` БЕЗ `expand=changelog`
- Извлечь поля согласно CONTRACT.md секция "После jira-enricher"
- Применить mapping статус → category → phase
- Извлечь эпик через `customfield_11400` или `issuelinks "Implement in"`
- Извлечь команду через `customfield_22200` или fallback на assignee
- Посчитать `lead_time_days`
- Собрать уникальные эпики, для каждого сделать `jira_search('"Epic Link" = <key>')` для подсчёта дочерних
- Перезаписать `.cache/enriched.json` с заполненной секцией `jira` и массивом `epics`

## 3. Анти-цели

- **НЕ** запрашивать changelog — это работа `timing-analyzer`
- **НЕ** считать `phase_days` — у нас нет changelog
- **НЕ** генерировать markdown-отчёт
- **НЕ** читать Excel — только enriched.json

## 4. Вход и выход

### Вход

`.cache/enriched.json` в рабочей директории, прошедший `excel-parser`. Валидация:
- `metadata.skills_completed` содержит `"excel-parser"`
- `tasks` массив не пустой

Если валидация не прошла — сообщить пользователю запустить `excel-parser` сначала, завершить.

### Выход

Тот же файл `.cache/enriched.json`, **перезаписанный** с дополнениями:
- `metadata.enriched_at = "<now ISO 8601>"`
- `metadata.skills_completed` += `"jira-enricher"`
- Для каждой задачи заполнено поле `jira`
- Массив `epics` заполнен уникальными эпиками с `children_count_total`

Структура — CONTRACT.md секция "После jira-enricher".

## 5. Формула вызова MCP — зафиксирована

### Для каждой задачи

```
Tool: jira_get_issue

Параметры:
  key = <cr_key из enriched.json>
  expand = (НЕ передавать — без changelog)
  fields = "summary,issuetype,status,project,created,updated,resolutiondate,
            reporter,assignee,priority,labels,description,parent,
            customfield_11400,customfield_22200,issuelinks"
```

Поля строго ограничены. **НЕ** использовать `fields="*"` (раздувает контекст). **НЕ** добавлять `expand="changelog"` (это работа следующего скилла).

### Для каждого уникального эпика

После прохода по всем задачам — собрать уникальные `epic.key` (не пустые), для каждого:

```
Tool: jira_search

Параметры:
  jql = '"Epic Link" = <epic_key>'
  fields = "summary,status,issuetype"
  maxResults = 100
```

Результат — `len(issues)` записывается как `children_count_total`. Сами задачи **не** сохраняем (это для будущих версий v6).

## 6. Mapping статуса → category → phase

Используется единый mapping из CONTRACT.md секция "Маппинг статуса → category → phase".

Извлекается из `helper.py` функция `map_status(status_name) -> (category, phase)`.

## 7. Извлечение полей из ответа Jira

### Эпик (приоритет важен)

```python
# псевдокод в helper.py
def extract_epic(jira_response):
    # Способ A: прямой Epic Link через customfield_11400
    epic_key = jira_response['fields'].get('customfield_11400')
    if epic_key and isinstance(epic_key, str) and re.match(r'^[A-Z]+-\d+$', epic_key):
        return {'key': epic_key, 'name': None, 'source': 'customfield_11400'}
    
    # Способ B: через issuelinks "Implement in"
    for link in jira_response['fields'].get('issuelinks', []):
        link_type = link.get('type', {}).get('outward', '')
        if link_type == 'Implement in' and 'outwardIssue' in link:
            outward = link['outwardIssue']
            return {
                'key': outward['key'],
                'name': outward.get('fields', {}).get('summary'),
                'source': 'issuelinks.Implement_in'
            }
    
    return {'key': None, 'name': None, 'source': None}
```

`name` эпика заполняется когда придёт через `issuelinks`. Если эпик получен через `customfield_11400` — имя пока null, агент может **опционально** сделать второй `jira_get_issue` для эпика чтобы получить имя. Это решение принимается скиллом: если эпиков мало (≤15) — запросить имена, иначе оставить null.

### Команда

```python
def extract_team(jira_response):
    cf = jira_response['fields'].get('customfield_22200')
    if isinstance(cf, list) and len(cf) > 0:
        return {'value': str(cf[0]), 'source': 'customfield_22200'}
    
    assignee = jira_response['fields'].get('assignee')
    if assignee and assignee.get('displayName'):
        return {'value': assignee['displayName'], 'source': 'assignee_fallback'}
    
    return {'value': None, 'source': None}
```

### Lead time

```python
def compute_lead_time(jira_response):
    from datetime import datetime, timezone
    created = parse_iso(jira_response['fields']['created'])
    resolved = jira_response['fields'].get('resolutiondate')
    end = parse_iso(resolved) if resolved else datetime.now(timezone.utc)
    return (end - created).days
```

## 8. Steps

### Step 1. Валидация enriched.json

Прочитать `.cache/enriched.json`. Если не существует — попросить пользователя запустить `excel-parser`. Завершить.

Проверить `"excel-parser" in metadata.skills_completed`. Если нет — то же самое.

Если `tasks` пуст — сообщить, завершить.

### Step 2. Для каждой задачи — jira_get_issue

Последовательно (не параллельно). Для каждой `task` с `cr_key`:

1. Сделать tool call `jira_get_issue` с параметрами из раздела 5 — это **нативный tool call агента**, не Python-обёртка
2. Получить JSON-ответ
3. Если ошибка 404 / "issue not found":
   ```json
   "jira": {
     "found": false,
     "error": "404 not found"
   }
   ```
   Продолжить со следующей задачей
4. Если другая ошибка (timeout, 500) — записать с error, продолжить
5. Если ответ корректный — извлечь поля через helper.py функции:
   - `extract_basic(response)` — summary, status, issue_type, project, priority, labels, created, updated, resolutiondate, assignee, reporter
   - `map_status(status_name)` → category, phase
   - `extract_epic(response)` → epic dict
   - `extract_team(response)` → team dict
   - `compute_lead_time(response)` → number
6. Собрать в объект `task.jira` согласно CONTRACT.md
7. Прогресс каждые 5 задач: "Обработано 5/28"
8. Пауза 0.1 сек между вызовами

### Step 3. Опционально — догрузить имена эпиков

После Step 2 у нас есть массив задач с `task.jira.epic.key`. Часть эпиков имеет `name = null` (те которые пришли через `customfield_11400`).

Собрать уникальные эпики без имени. Если их ≤15 — сделать `jira_get_issue` для каждого с `fields="summary"`, заполнить имя. Если их больше — оставить null (можно догрузить в timing-analyzer или вручную).

### Step 4. Подсчитать дочерние для каждого эпика

Собрать массив уникальных эпиков из всех задач. Для каждого:

1. Tool call `jira_search` с jql `'"Epic Link" = <epic_key>'`, maxResults=100
2. Записать `children_count_total = len(issues)`
3. Записать `tasks_from_plan = [список cr_key которые в плане ссылаются на этот эпик]`
4. Имя эпика — берём из первого встретившегося `task.jira.epic.name`, если есть

Сформировать массив `epics`.

### Step 5. Обновить metadata

```json
"metadata": {
  ...existing fields...,
  "enriched_at": "<now ISO 8601>",
  "skills_completed": ["excel-parser", "jira-enricher"],
  "jira_stats": {
    "tasks_total": 28,
    "tasks_found": 26,
    "tasks_not_found": 2,
    "epics_unique": 12,
    "epics_with_names": 10,
    "epics_without_names": 2
  }
}
```

### Step 6. Записать обновлённый json

Перезаписать `.cache/enriched.json` целиком с обновлёнными полями. Сохранять `indent=2, ensure_ascii=False`.

### Step 7. Сводка в чат

- Обработано задач: X из Y (M не найдены)
- Уникальных эпиков: N
- Команды найдены: K через customfield_22200, L через assignee_fallback, P без команды
- Следующий шаг: запустите `timing-analyzer` (для расчёта факта А/Р/Т) или сразу `report-builder` если факт не нужен

## 9. КРИТИЧНО: формат вызова MCP

Те же правила что в pipeline-предшественниках. SKILL.md — инструкция для агента в чате, **не** Python-скрипт.

**Правильно** в SKILL.md:
```
Шаг 2.1. Для каждой задачи из enriched.tasks, у которой cr_key не пустой:
  Сделать tool call jira_get_issue со следующими параметрами:
    key = <cr_key текущей задачи>
    fields = "summary,issuetype,status,project,created,updated,resolutiondate,reporter,assignee,priority,labels,description,parent,customfield_11400,customfield_22200,issuelinks"
  
  Получить JSON-ответ. Если ответ содержит ошибку — записать task.jira = {found: false, error: <текст>}.
  Иначе передать ответ в функцию helper.extract_jira_fields(response) и сохранить результат в task.jira.
```

**Запрещено** в SKILL.md:
- `result = mcp_jira_get_issue(...)` — обёртка
- `# здесь будет вызов MCP`, `# TODO`
- `def fetch_issue(...)` с псевдокодом
- `from mcp_atlassian import ...`

Расчёты, парсинг JSON, mapping — через `helper.py`. Сами вызовы MCP — нативные tool calls агента.

## 10. Файлы

| Файл | Назначение |
|------|------------|
| `.cache/enriched.json` | Читается и перезаписывается |
| `helper.py` (в `~/.gigacode/skills/jira-enricher/`) | Функции `extract_basic`, `map_status`, `extract_epic`, `extract_team`, `compute_lead_time`, `extract_jira_fields` (агрегирующая) |

### Запрещённые файлы

То же что в `excel-parser`: только `helper.py`, никаких `main.py`, `process.py`, `run_*.py`, `generate_*.py`, `__pycache__`, виртуальных окружений.

## 11. Guardrails

- READ-ONLY для Jira (только `jira_get_issue` и `jira_search`, никаких write-операций)
- Точные `fields=` из раздела 5, не `fields="*"`
- НЕ запрашивать changelog
- НЕ ходить за дочерними задачами эпика дальше counter
- Не падать на одной ошибке Jira — продолжать
- Не выдумывать поля если их нет в ответе — писать null

## 12. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| `.cache/enriched.json` не существует | Попросить запустить excel-parser, завершить |
| `metadata.skills_completed` не содержит `excel-parser` | То же |
| Задача 404 в Jira | `task.jira = {found: false, error: "404"}`, продолжить |
| Jira timeout | Записать error, продолжить со следующей задачей |
| Статус не в mapping | `category = "unknown"`, `phase = null`, в `jira_stats.statuses_unknown` инкрементировать |
| `customfield_11400` пустой и `issuelinks` без "Implement in" | `epic = {key: null, name: null, source: null}` |
| `customfield_22200` пустой и `assignee` пустой | `team = {value: null, source: null}` |
| Дочерних у эпика 0 (пустой результат `jira_search`) | `children_count_total = 0`, валидно |
| `jira_search` вернул ошибку | Записать `children_count_total = null`, в metadata jira_stats отметить |
| `jira_search` вернул >100 (limit) | Записать `children_count_total = 100`, в metadata пометка "limit hit для <epic_key>" |
| Повторный запуск (idempotency) | Перезаписать поля jira и epics, не дублировать `skills_completed` |

## 13. Антипаттерны

### Критические

- Использовать `fields="*"` — раздувает контекст. Точный список.
- Запрашивать `expand=changelog` — это работа timing-analyzer.
- Создавать `main.py`, `process.py` и подобные — только `helper.py`.
- Заглушки `# здесь будет вызов MCP` — реальные tool calls сразу.
- Хранить сырой JSON ответа MCP в `enriched.json` — только извлечённые поля.

### Обычные

- Падать на одной 404 вместо продолжения
- Параллельные вызовы MCP — только последовательно
- Перезаписывать `.cache/enriched.json` без `ensure_ascii=False` (кириллица превратится в `\u...`)
- Игнорировать `metadata.skipped_rows` из excel-parser

## 14. Критерий успеха

После запуска:
1. `.cache/enriched.json` перезаписан, валиден
2. У каждой задачи поле `jira` заполнено (либо `found: true` с данными, либо `found: false`)
3. Массив `epics` непустой (если у задач есть эпики)
4. `metadata.skills_completed` содержит `["excel-parser", "jira-enricher"]`
5. В чате сводка с реальными цифрами

## 15. Что отложено

- v4: догрузка плановых дат ИФТ/ПСИ/ПРОМ из customfields 24300/29500/13700/22601/23703 (это будет в отдельном скилле `dates-enricher`, добавится в pipeline после jira-enricher)
- v6: расширенный поиск дочерних задач эпиков — получать **каждую** дочернюю задачу с её планом, не только counter
