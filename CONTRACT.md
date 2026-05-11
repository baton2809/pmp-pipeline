# CONTRACT.md — структура данных pipeline

> Все 4 скилла (`excel-parser`, `jira-enricher`, `timing-analyzer`, `report-builder`) разделяют **один файл** `.cache/enriched.json` в рабочей директории. Каждый следующий скилл читает текущее состояние, дополняет своими полями, перезаписывает обратно.
>
> Этот документ — единственный источник истины о структуре данных. Любые изменения в `enriched.json` начинаются с обновления здесь.

## Расположение файла

```
<рабочая_директория>/.cache/enriched.json
```

`<рабочая_директория>` — папка где лежит `Бэклог и цели.xlsx` и куда запускается GigaCode CLI. Папка `.cache/` создаётся автоматически первым скиллом (`excel-parser`).

## Полная структура

```json
{
  "metadata": {
    "source_file": "Бэклог и цели.xlsx",
    "sheet": "Q2_26_оценки_new_name",
    "parsed_at": "2026-05-11T18:00:00",
    "enriched_at": null,
    "timing_at": null,
    "scope_version": "v3.0",
    "skills_completed": ["excel-parser"]
  },
  "tasks": [
    {
      "cr_key": "CRSIGMA-26516",
      "task_name": "Доработка УКИ. Добавление драг. металлов - золото, серебро",
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

Заполнены только `metadata.parsed_at`, `metadata.skills_completed = ["excel-parser"]` и массив `tasks` с базовыми полями плана.

| Поле | Тип | Описание | Источник |
|------|-----|----------|----------|
| `cr_key` | string | Ключ задачи в Jira (`CRSIGMA-26516`, `ASFC-58741`) | Excel колонка B (URL или строка), парсится regex |
| `task_name` | string | Название задачи | Excel колонка G (без `\n`, `\r`, `\t`) |
| `initiative` | string | Инициатива (для группировки) | Excel колонка F |
| `customer` | string \| null | Заказчик | Excel колонка C |
| `plan.analytics` | number \| null | План аналитики (чд) | Excel колонка R (вторая версия), fallback на J |
| `plan.development` | number \| null | План разработки (чд) | Excel колонка S, fallback на K |
| `plan.testing` | number \| null | План тестирования (чд) | Excel колонка T, fallback на L |
| `plan.total` | number \| null | Сумма А+Р+Т | computed |
| `plan.source_version` | string | `"v1"` (J/K/L) или `"v2"` (R/S/T) | вычисляется |

Если задача без CR-ключа в Excel — не включается в `tasks`. Регистрируется отдельно в `metadata.skipped_rows`.

### После `jira-enricher`

Заполняется поле `jira` для каждой задачи (если найдена), и массив `epics` уникальными эпиками.

```json
{
  "metadata": {
    ...
    "enriched_at": "2026-05-11T18:05:00",
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
        "status_category": "not_started",
        "phase": null,
        "issue_type": "Change Request",
        "project": "CRSIGMA",
        "priority": "Critical",
        "labels": ["ПКАП.PALM", "Прайсинг", "ЦКП"],
        "created": "2024-07-31T14:44:00+0300",
        "updated": "2025-05-11T11:02:30+0300",
        "resolutiondate": null,
        "assignee": "Гриднева Ирина Ринатовна",
        "reporter": "Никиткина Екатерина Борисовна",
        "epic": {
          "key": "ASFC-65543",
          "name": "ЦКП.ПГ-1 универсальная задача КИ",
          "source": "customfield_11400"
        },
        "team": {
          "value": "PALM.CSP.K7M",
          "source": "customfield_22200"
        },
        "lead_time_days": 287,
        "fetched_at": "2026-05-11T18:05:00"
      }
    }
  ],
  "epics": [
    {
      "key": "ASFC-65543",
      "name": "ЦКП.ПГ-1 универсальная задача КИ",
      "tasks_from_plan": ["CRSIGMA-26516", "ASFC-63820"],
      "children_count_total": 38,
      "fetched_at": "2026-05-11T18:05:30"
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
| `team.value` | string \| null | Команда (например `PALM.CSP.K7M`) |
| `team.source` | string | `"customfield_22200"` или `"assignee_fallback"` |
| `lead_time_days` | number | Дни от created до resolutiondate (если закрыта) или до now |

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
    "timing_at": "2026-05-11T18:10:00",
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
        "computed_at": "2026-05-11T18:10:00"
      }
    }
  ]
}
```

| Поле в `task.timing` | Тип | Описание |
|----------------------|-----|----------|
| `computed` | bool | `false` если у задачи не было changelog (например, только что создана) |
| `phase_days.A` | number | Календарные дни в фазе А (сумма всех интервалов в статусах А) |
| `phase_days.R` | number | Календарные дни в фазе Р |
| `phase_days.T` | number | Календарные дни в фазе Т |
| `phase_days.not_started` | number | Дни в not_started статусах (Backlog) |
| `phase_days.finished` | number | Дни после перехода в финальный статус |
| `phase_days.unknown` | number | Дни в нестандартных статусах |
| `transitions_count` | number | Количество переходов статуса |
| `first_transition` | string \| null | Timestamp первого перехода |
| `last_transition` | string \| null | Timestamp последнего перехода |

Если задача в `not_started` или `finished` — `timing.computed = false`, `phase_days` все нули.

### После `report-builder`

Файл `enriched.json` **не изменяется**. Создаётся `report.md` в рабочей директории.

```json
{
  "metadata": {
    ...
    "skills_completed": ["excel-parser", "jira-enricher", "timing-analyzer", "report-builder"],
    "report_generated_at": "2026-05-11T18:15:00"
  }
}
```

## Валидация между скиллами

Каждый скилл при запуске **проверяет**:

1. Файл `.cache/enriched.json` существует (кроме `excel-parser` который его создаёт)
2. `metadata.skills_completed` содержит все предшественники:
   - `jira-enricher` требует `["excel-parser"]`
   - `timing-analyzer` требует `["excel-parser", "jira-enricher"]`
   - `report-builder` требует `["excel-parser", "jira-enricher"]` (timing-analyzer опционален)
3. `metadata.source_file` соответствует ожидаемому
4. `tasks` массив не пустой

Если валидация не прошла — скилл сообщает пользователю что нужно запустить предшественник, и не пытается работать частично.

## Идемпотентность

**Любой скилл можно запустить повторно.** Он перезапишет свои поля. Это полезно когда:
- Изменился Excel → перезапустить весь pipeline
- Хочется обновить статусы Jira → перезапустить `jira-enricher`
- Хочется пересчитать тайминги → перезапустить `timing-analyzer`

Сам по себе `enriched.json` после повторного запуска перезаписывается **только в той части** которую обрабатывает данный скилл. Поля предыдущих скиллов сохраняются.

## Ограничения

- Максимум задач в плане: ~100. При большем количестве `enriched.json` может стать большим (~500 KB), скиллы могут потерять контекст. Если планов больше — обсуждать архитектуру отдельно.
- Не хранится сырой ответ MCP — только извлечённые поля. Это сознательно (экономия места и контекста).
- Не хранится changelog — только агрегированные `phase_days`. Если нужна история переходов конкретной задачи — отдельный скилл `task-details <key>` (вне scope текущего pipeline).
