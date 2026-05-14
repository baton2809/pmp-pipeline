# CONTRACT.md — структура данных pipeline

> Все 4 скилла (`excel-parser`, `jira-enricher`, `timing-analyzer`, `report-builder`) разделяют **один файл** `pipeline/enriched.json` в рабочей директории. Каждый следующий скилл читает текущее состояние, дополняет своими полями, перезаписывает обратно.
>
> Дополнительно каждый скилл создаёт **видимый markdown-снимок** `pipeline/step-N-after-<skill>.md` чтобы пользователь мог сразу проверить результат каждого шага не открывая json.
>
> Этот документ — единственный источник истины о структуре данных. Любые изменения в `enriched.json` начинаются с обновления здесь.

## Расположение файлов

```
<рабочая_директория>/
├── Бэклог и цели.xlsx              ← вход (файл Натальи)
├── pipeline/                       ← создаётся первым скиллом, ВИДИМАЯ папка
│   ├── enriched.json               ← канонические данные (передаются между скиллами)
│   ├── step-1-after-excel-parser.md      ← снимок после excel-parser
│   ├── step-2-after-jira-enricher.md     ← снимок после jira-enricher
│   ├── step-3-after-timing-analyzer.md   ← снимок после timing-analyzer (опционально)
└── report.md                       ← финальный отчёт от report-builder
```

**Важно:** папка называется `pipeline/`, **не** `.cache/`. Точка в начале делает её скрытой в Linux/Mac — это неудобно для пользователя.

## Зачем оба формата (json + md)?

- **`enriched.json`** — для **скиллов**. Структурированные данные, легко парсятся, не теряют типы.
- **`step-N-after-*.md`** — для **пользователя**. Сразу видно что наработал каждый скилл, можно проверить таблицу без открытия json.

Скиллы передают данные **через json**, не через markdown. Markdown — это side-effect для прозрачности, никто его не парсит.

## Полная структура enriched.json

```json
{
  "metadata": {
    "source_file": "Бэклог и цели.xlsx",
    "sheet": "Q2_26_оценки_new_name",
    "parsed_at": "2026-05-12T18:00:00",
    "enriched_at": null,
    "timing_at": null,
    "report_generated_at": null,
    "scope_version": "v3.1",
    "skills_completed": ["excel-parser"]
  },
  "tasks": [
    {
      "cr_key": "CRSIGMA-26516",
      "task_name": "Доработка УКИ. Добавление драг. металлов",
      "initiative": "Реализована ДК по повышению уровня автономности ААВ",
      "customer": "Иванов И.И.",
      "plan": {
        "analytics": 23,
        "development": 23,
        "testing": 15,
        "total": 61,
        "source_version": "v2"
      },

      "jira": null,
      "timing": null
    }
  ],
  "epics": []
}
```

### После `excel-parser` (обязательные поля)

Заполнены `metadata.parsed_at`, `metadata.skills_completed = ["excel-parser"]` и массив `tasks` с базовыми полями плана. `jira: null` и `timing: null` для каждой задачи.

| Поле | Тип | Описание | Источник |
|------|-----|----------|----------|
| `cr_key` | string | Ключ задачи в Jira (`CRSIGMA-26516`, `ASFC-58741`) | Excel колонка `cr` |
| `task_name` | string | Название задачи | Excel колонка `задача` |
| `initiative` | string | Инициатива | Excel колонка `инициатива` |
| `customer` | string \| null | Заказчик | Excel колонка `заказчик` |
| `plan.analytics` | number \| null | План аналитики (чд) | вторая позиция колонок `аналитика` |
| `plan.development` | number \| null | План разработки (чд) | вторая позиция колонок `разработка` |
| `plan.testing` | number \| null | План тестирования (чд) | вторая позиция колонок `тестирование` |
| `plan.total` | number \| null | Сумма А+Р+Т | computed |
| `plan.source_version` | string | `"v1"` (первая позиция) или `"v2"` (вторая) | вычисляется |

### После `jira-enricher`

Заполняется поле `jira` для каждой задачи (если найдена), и массив `epics` уникальными эпиками.

```json
{
  "metadata": {
    ...
    "enriched_at": "2026-05-12T18:05:00",
    "skills_completed": ["excel-parser", "jira-enricher"]
  },
  "tasks": [
    {
      "cr_key": "CRSIGMA-26516",
      ...
      "jira": {
        "found": true,
        "summary": "ЦКП. Доработка УКИ (металлические счета)",
        "status": "New",
        "status_category": "analysis",
        "phase": "A",
        "issue_type": "Change Request",
        "project": "CRSIGMA",
        "priority": "Critical",
        "labels": ["ПКАП.PALM", "Прайсинг", "ЦКП"],
        "created": "2026-02-13T18:24:19.841+0300",
        "updated": "2026-05-11T11:02:30.763+0300",
        "resolutiondate": null,
        "assignee": "Гриднева Ирина Ринатовна",
        "reporter": "Никиткина Екатерина Борисовна",
        "epic": {
          "key": "ASFC-57216",
          "name": "ЦКП.ПГ-1 универсальная задача",
          "source": "issuelinks.Implement_in"
        },
        "team": {
          "value": "PALM.CSP.K7M",
          "source": "customfield_22200"
        },
        "lead_time_days": 87,
        "fetched_at": "2026-05-12T18:05:00"
      }
    }
  ],
  "epics": [
    {
      "key": "ASFC-57216",
      "name": "ЦКП.ПГ-1 универсальная задача",
      "tasks_from_plan": ["CRSIGMA-26516", "ASFC-63820"],
      "children_count_total": 38,
      "fetched_at": "2026-05-12T18:05:30"
    }
  ]
}
```

| Поле в `task.jira` | Тип | Описание |
|---------------------|-----|----------|
| `found` | bool | `false` если задача не найдена в Jira (404) — остальные поля null |
| `summary` | string | Название из Jira |
| `status` | string | Точное имя статуса (`New`, `In Progress`, `Done`, `Analysis`, ...) |
| `status_category` | string | Одно из: `not_started`, `analysis`, `development`, `testing`, `finished`, `unknown` |
| `phase` | string \| null | Одно из: `A`, `R`, `T`, или `null` для not_started/finished |
| `issue_type` | string | `Change Request`, `Task`, `Story`, `Bug`, ... |
| `project` | string | `CRSIGMA`, `ASFC`, `TIBDS` |
| `priority` | string | `Critical`, `Major`, `Minor`, ... |
| `labels` | array[string] | Метки задачи |
| `created` | string ISO 8601 | Дата создания |
| `updated` | string ISO 8601 | Дата последнего обновления |
| `resolutiondate` | string \| null | Дата закрытия (для finished) |
| `assignee` | string \| null | Имя исполнителя |
| `reporter` | string \| null | Имя автора |
| `epic.key` | string \| null | Ключ эпика |
| `epic.name` | string \| null | Имя эпика |
| `epic.source` | string \| null | `"customfield_11400"` или `"issuelinks.Implement_in"` |
| `team.value` | string \| null | Поток разработки (например `PALM.CSP.K7M` — техническая метка из customfield_22200, не "команда" типа "Пальмира") |
| `team.source` | string | `"customfield_22200"` или `"assignee_fallback"` |
| `lead_time_days` | number | Дни от created до resolutiondate (если закрыта) или до now |

**Важно про `customfield_22200`:** это **массив строк** типа `["PALM.CSP.K7M"]`, не объект. Берём первый элемент массива. Это **техническая метка потока разработки** в Сбер-Jira (`PALM.*` означает Palmira-стек), а не "команда" в обычном понимании. Поэтому в отчёте колонку называем "Поток/Проект", не "Команда".

**Важно про `customfield_11400`:** для **ASFC-задач** обычно содержит ключ эпика как строку. Для **CRSIGMA-задач** часто `null` — эпик находится через `issuelinks` тип `"Implement in"`.

### Маппинг статуса → category → phase

Используется единый для всех скиллов:

| Статус (case-insensitive) | category | phase |
|---------------------------|----------|-------|
| `Backlog`, `TO DO`, `Открыта` | `not_started` | `null` |
| `New` | `analysis` | `A` |
| `Need Info` | `analysis` | `A` |
| `Analysis`, `АНАЛИЗ` | `analysis` | `A` |
| `In Progress`, `РАЗРАБОТКА`, `ГОТОВ К РАЗРАБОТКЕ` | `development` | `R` |
| `Ready for QA`, `ГОТОВ К ТЕСТИРОВАНИЮ`, `НАЧАТО ТЕСТИРОВАНИЕ`, `Тестирование`, `ST`, `IFT`, `UAT`, `ПСИ`, `ПРОВЕРЕНО НА ИФТ/ГОТ`, `In Discovery` | `testing` | `T` |
| `Done`, `Resolved`, `Closed`, `Закрыт`, `ЗАКРЫТЫ`, `Cancelled` | `finished` | `null` |
| (всё остальное) | `unknown` | `null` |

### После `timing-analyzer`

Заполняется поле `timing` для **активных** задач (статус не finished и не not_started).

```json
{
  "metadata": {
    ...
    "timing_at": "2026-05-12T18:10:00",
    "skills_completed": ["excel-parser", "jira-enricher", "timing-analyzer"]
  },
  "tasks": [
    {
      "cr_key": "CRSIGMA-23749",
      ...
      "jira": { ... },
      "timing": {
        "computed": true,
        "phase_days": {
          "A": 30.5,
          "R": 240.2,
          "T": 0.0,
          "not_started": 0.0,
          "finished": 0.0,
          "unknown": 0.0
        },
        "transitions_count": 5,
        "first_transition": "2024-08-15T09:30:00+0300",
        "last_transition": "2024-12-10T11:00:00+0300",
        "computed_at": "2026-05-12T18:10:00"
      }
    }
  ]
}
```

Если задача в `not_started` или `finished` — `timing.computed = false`, `phase_days` все нули.

### Структура ответа MCP с changelog (для timing-analyzer)

Важно для скилла `timing-analyzer` который запрашивает changelog. Реальная структура ответа Сбер-MCP:

```python
{
    'key': 'ASFC-67203',
    'fields': {
        'created': '2026-04-23T12:40:49+0300',
        'status': {'name': 'Ready for QA'},
        'resolutiondate': None,
    },
    'changelogs': [                           # ← МНОЖЕСТВЕННОЕ число!
        {
            'created': '2025-08-14T13:32:23.287+0300',
            'items': [
                {
                    'field': 'status',
                    'fromString': 'New',      # ← camelCase для status!
                    'toString': 'In Progress',
                },
                {
                    'field': 'Link',
                    'to_string': '...',       # ← snake_case для не-status
                }
            ]
        }
    ]
}
```

**Что критично:**
1. Ключ верхнего уровня — **`changelogs`** (множественное число). НЕ `changelog`.
2. **Нет** обёртки `histories` — массив сразу содержит элементы.
3. Для `field == 'status'` — поля **`fromString`** / **`toString`** (camelCase).
4. Для других полей (`Link`, `description`, ...) — могут быть другие имена. Эти переходы фильтруем.

Скилл `timing-analyzer` использует функцию `extract_status_transitions(changelog_list)` которая фильтрует только status-переходы.

### После `report-builder`

`enriched.json` обновляется минимально (только `metadata.report_generated_at`). Создаётся `report.md` в **корне** рабочей директории (не в `pipeline/`).

## Структура промежуточных markdown-снимков

Каждый скилл создаёт `pipeline/step-N-after-<skill>.md` после своей работы. Это **снимок текущего состояния enriched.json в человекочитаемом виде**, не финальный отчёт.

### `step-1-after-excel-parser.md`

```markdown
# Снимок после excel-parser

**Дата:** 2026-05-12 18:00
**Источник:** Бэклог и цели.xlsx, лист Q2_26_оценки_new_name
**Задач извлечено:** 28
**Пропущено строк (без CR):** 0

## Задачи плана

| # | CR | Название | План А/Р/Т (Σ) | Версия плана |
|---|-----|----------|-----------------|---------------|
| 1 | CRSIGMA-26516 | Доработка УКИ. Добавление драг. металлов | 23/23/15 (61) | v2 |
| 2 | CRSIGMA-23749 | Расчёт в ин. валютах по кредитам | 23/23/15 (61) | v2 |
| ... | ... | ... | ... | ... |

## Следующий шаг

Запустить `jira-enricher` — добавит статусы, эпики, команды из Jira.
```

### `step-2-after-jira-enricher.md`

```markdown
# Снимок после jira-enricher

**Дата:** 2026-05-12 18:05
**Задач из плана:** 28
**Найдено в Jira:** 28
**Не найдено:** 0

## Сводка по статусам

| Категория | Количество |
|-----------|------------|
| not_started | 13 |
| analysis (А) | 8 |
| development (Р) | 5 |
| testing (Т) | 1 |
| finished | 1 |

## Задачи

| # | CR | Статус | Фаза | Эпик | Поток/Проект | Lead time |
|---|-----|--------|------|------|---------|-----------|
| 1 | CRSIGMA-26516 | New | А | ASFC-57216 ЦКП.ПГ-1 | PALM.CSP.K7M | 87 д |
| ... | ... | ... | ... | ... | ... | ... |

## Уникальные эпики (12)

| Эпик | Имя | Задач из плана | Всего дочерних |
|------|-----|----------------|-----------------|
| ASFC-57216 | ЦКП.ПГ-1 универсальная задача | 2 | 38 |
| ... | ... | ... | ... |

## Следующий шаг

Запустить `timing-analyzer` — добавит факт А/Р/Т для активных задач (опционально).
Или сразу `report-builder` если timing не нужен.
```

### `step-3-after-timing-analyzer.md`

```markdown
# Снимок после timing-analyzer

**Дата:** 2026-05-12 18:10
**Активных задач:** 14
**С реальным timing:** 12
**Без changelog:** 2

## Топ-5 задач с самыми долгими фазами

| CR | Статус | Факт А (д) | Факт Р (д) | Факт Т (д) | План А/Р/Т |
|-----|--------|------------|------------|------------|-------------|
| ASFC-35817 | In Progress | 98 | 700 | 0 | 38/23/15 |
| ... | ... | ... | ... | ... | ... |

## Следующий шаг

Запустить `report-builder` — соберёт финальный report.md.
```

## Валидация между скиллами

Каждый скилл при запуске **проверяет**:

1. Файл `pipeline/enriched.json` существует (кроме `excel-parser` который его создаёт)
2. `metadata.skills_completed` содержит все предшественники:
   - `jira-enricher` требует `["excel-parser"]`
   - `timing-analyzer` требует `["excel-parser", "jira-enricher"]`
   - `report-builder` требует `["excel-parser", "jira-enricher"]` (timing-analyzer опционален)
3. `metadata.source_file` соответствует ожидаемому
4. `tasks` массив не пустой

Если валидация не прошла — скилл сообщает пользователю что нужно запустить предшественник, и не пытается работать частично.

## Идемпотентность

**Любой скилл можно запустить повторно.** Он перезапишет свои поля и свой step-N.md. Это полезно когда:
- Изменился Excel → перезапустить весь pipeline
- Хочется обновить статусы Jira → перезапустить `jira-enricher`
- Хочется пересчитать тайминги → перезапустить `timing-analyzer`

Сам по себе `enriched.json` после повторного запуска перезаписывается **только в той части** которую обрабатывает данный скилл. Поля предыдущих скиллов сохраняются.

## Архитектурный принцип — где живут данные

**Критически важно для скиллов которые делают MCP-вызовы (`jira-enricher`, `timing-analyzer`):**

```
Агент (GigaCode CLI):
  - Делает нативные tool calls (jira_get_issue, jira_search)
  - Видит JSON-ответы в своём контексте
  - Извлекает нужные поля (явно, через чтение JSON)
  - Передаёт данные в helper.py через bash

Python через bash (helper.py):
  - НЕ делает MCP-вызовов (это невозможно из Python в окружении GigaCode)
  - Принимает данные через stdin (для маленьких batch) или читает из файла на диске (для больших)
  - Парсит, валидирует, мерджит в enriched.json
  - Записывает файл, обновляет step-N-after-*.md
```

**Запрещено** для агента:

```python
# Это НЕ работает в Python-окружении GigaCode!
from mcp_atlassian import jira_get_issue  # такого модуля нет
result = mcp__Atlassian__jira_get_issue(key="...")  # NameError
```

### Два паттерна передачи данных агент → Python

**Паттерн A: stdin через echo (для маленьких ответов).**

Подходит для `jira-enricher` где каждый ответ компактный (без changelog ~2-3 KB), но **батчами по 5 задач** ответы помещаются в командной строке через `echo`:

```
1. Агент делает 5 tool calls: jira_get_issue(key=...)
2. Видит JSON в контексте
3. Извлекает поля из каждого ответа, накапливает batch
4. После 5 задач:
   echo '<batch_json>' | python3 ~/.gigacode/skills/jira-enricher/helper.py merge-batch
5. helper.py читает stdin, мерджит в pipeline/enriched.json
```

**Паттерн B: WriteFile + чтение из файла (для больших ответов).**

Обязателен для `timing-analyzer` где каждый ответ с `expand=changelog` весит 5-10 KB. Передача через `echo` ломается на длине команды и кавычках. Архитектура streaming:

```
Для каждой active задачи (по ОДНОЙ, не batch):
  1. Tool call: jira_get_issue(key=..., expand="changelog")
     → агент получает большой JSON в контекст
  2. Tool: WriteFile (встроенный в GigaCode CLI)
     path = "pipeline/tmp/<cr_key>.json"
     content = <сырой JSON-ответ>
     → JSON на диске
  3. Shell: python3 helper.py compute-from-file pipeline/tmp/<cr_key>.json
     → helper читает с диска, считает, мерджит, выводит сводку
  4. После этого JSON-ответ "выпадает" из контекста — он уже обработан
  5. Следующая задача начинается с пустого контекста по JSON-ответам
```

**Запрещено для Паттерна B:**
- `echo '...' > file.json` через bash — длина команды и кавычки ломают большой JSON
- `python3 -c "..."` с JSON в коде — то же
- Запись в `~/.gigacode/tmp/` или любые пути содержащие `.gigacode/` — Filesystem Guard блокирует

Только **WriteFile tool агента + рабочая директория проекта** (`pipeline/tmp/`).

## Ограничения

- Максимум задач в плане: ~100. При большем количестве `enriched.json` может стать большим, скиллы могут потерять контекст.
- Не хранится сырой ответ MCP — только извлечённые поля. Это сознательно (экономия места и контекста).
- Не хранится changelog — только агрегированные `phase_days`.
