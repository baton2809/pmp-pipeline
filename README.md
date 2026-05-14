# pmp-pipeline v3.2 — 4 скилла для контроля портфеля задач

ETL-pipeline из 4 независимых скиллов для GigaCode CLI. Каждый скилл делает одну вещь, скиллы передают данные через **`pipeline/enriched.json`** в рабочей директории. После каждого шага создаётся **видимый markdown-снимок** для проверки промежуточного результата.

## Изменения v3.2 vs v3.1

**jira-enricher (КРИТИЧНЫЙ ФИКС):**
- В v3.1 массив `enriched.epics[]` оставался пустым → секция "Срез по эпикам" в финальном отчёте была пустая
- Корень: после `merge_batch` агент не вызывал агрегацию эпиков
- В v3.2: новые функции `aggregate_epics`, `update_epic_children`, `list_epics_to_count`
- В SKILL.md явные шаги: после всех `merge-batch` → `aggregate-epics` → цикл `jira_search + update-epic-children`
- Колонка в `step-2-after-jira-enricher.md` — "Поток" (не "Команда")

**timing-analyzer:**
- Структура changelog в Сбер-MCP: **`changelogs`** (множественное), без обёртки `histories`
- Для `field == 'status'` — поля **`fromString`** / **`toString`** (camelCase), не snake_case
- Архитектура передачи: **WriteFile tool агента** + чтение из файла (`compute-from-file`) вместо stdin/echo
- Streaming: обработка по **одной** задаче, не батчами по 5

**report-builder:**
- Колонка "Команда" → "Поток/Проект" (`customfield_22200` это техническая метка типа `PALM.CSP.K7M`, а не имя команды)
- Заголовок секции 4 — "Срез по потокам разработки"
- Пояснение под секцией 4 что это значит

**CONTRACT.md:**
- Описание правильной структуры changelog от MCP
- Описание двух паттернов передачи данных (stdin для маленьких batch, WriteFile+файл для больших с changelog)
- Уточнение про customfield_22200 (поток ≠ команда)

## Pipeline

```
┌───────────────────────────────────────────────────────────────────────┐
│ 1. excel-parser                                                       │
│    Читает: Бэклог и цели.xlsx (жёстко зафиксирован)                   │
│    Создаёт:                                                           │
│       pipeline/enriched.json (план задач)                             │
│       pipeline/step-1-after-excel-parser.md (читаемый снимок)         │
│    Без MCP                                                            │
└───────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│ 2. jira-enricher                                                      │
│    Читает: pipeline/enriched.json                                     │
│    MCP: jira_get_issue (БЕЗ changelog) × 28 + jira_search × ~12      │
│    Архитектура: агент в чате делает tool calls, helper.py через bash │
│                 принимает batch через stdin и мерджит в файл          │
│    Дополняет:                                                         │
│       pipeline/enriched.json — статус, фаза, эпик, команда, lead_time │
│       pipeline/step-2-after-jira-enricher.md                          │
└───────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│ 3. timing-analyzer (опционально)                                      │
│    Читает: pipeline/enriched.json                                     │
│    MCP: jira_get_issue С expand=changelog для active задач            │
│    Архитектура: агент передаёт сырой ответ MCP через stdin в helper,  │
│                 helper.compute_timing считает phase_days              │
│    Дополняет:                                                         │
│       pipeline/enriched.json — phase_days {A, R, T} в кален. днях     │
│       pipeline/step-3-after-timing-analyzer.md                        │
└───────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌───────────────────────────────────────────────────────────────────────┐
│ 4. report-builder                                                     │
│    Читает: pipeline/enriched.json                                     │
│    Создаёт: report.md (в корне рабочей директории, не в pipeline/)    │
│    Без MCP                                                            │
│    Условный рендеринг: с/без колонок Факт А/Р/Т                       │
└───────────────────────────────────────────────────────────────────────┘
```

## Главные архитектурные решения

### 1. Видимая папка `pipeline/` вместо скрытой `.cache/`

В v3.0 была `.cache/` — папка скрыта (точка в начале), пользователь её не видел. **В v3.1** папка называется `pipeline/` — видна в файловом менеджере.

### 2. Per-step markdown снимки

Каждый скилл создаёт `pipeline/step-N-after-<name>.md` — читаемый снимок текущего состояния. Не финальный отчёт, а **промежуточная проверка**. Пользователь может проверить результат каждого шага не открывая JSON.

### 3. Разделение агент ↔ Python через stdin

Это критическое исправление после v3.0 где `jira-enricher` падал с `NameError`.

**Архитектурное правило для скиллов с MCP (jira-enricher, timing-analyzer):**

```
АГЕНТ (GigaCode CLI в чате):
  ✅ Делает НАТИВНЫЕ tool calls (jira_get_issue, jira_search)
  ✅ Видит JSON-ответы в своём контексте
  ✅ Накапливает результаты как текст
  ✅ После каждого батча — передаёт в helper.py через bash + stdin
  
  ❌ НЕ пытается вызвать MCP из Python (это NameError)

PYTHON через bash (helper.py):
  ✅ Принимает данные через stdin
  ✅ Парсит JSON, мерджит, пишет файл
  
  ❌ НЕ делает MCP-вызовов (физически невозможно)
```

**Граница:** `tool call` → агент видит JSON → `echo '...' | python3 helper.py merge-batch`.

### 4. Один helper.py на скилл, никаких лишних файлов

В v3.0 GigaCode создавал 4 Python-файла (`main.py`, `process.py`, `generate_report.py`, `run_jira_v3.py`) и `__pycache__`. **В v3.1** строгое правило: **один** `helper.py`, **никаких** `main.py`, `process.py`, `run_*.py`, `__pycache__`, `requirements.txt`, виртуальных окружений.

### 5. Готовый код helper.py в каждом SPEC.md

Каждый SPEC.md содержит **готовый полный код** `helper.py` — не описание функций, а **рабочий код** который можно скопировать. Это снижает риск что GigaCode что-то напишет криво.

## Структура репозитория

```
pmp-pipeline/
├── CONTRACT.md                   ← структура enriched.json (для всех скиллов)
├── README.md                     ← этот файл
│
├── excel-parser/
│   ├── SPEC.md                   ← 14 разделов, готовый код helper.py в разделе 7
│   ├── PROMPT.md                 ← для one-shot GigaCode
│   ├── README.md                 ← инструкция запуска
│   └── skill/excel-parser/       ← результат one-shot (SKILL.md + helper.py)
│
├── jira-enricher/
│   ├── SPEC.md                   ← готовый код helper.py в разделе 7
│   ├── PROMPT.md
│   ├── README.md
│   └── skill/jira-enricher/
│
├── timing-analyzer/
│   ├── SPEC.md                   ← готовый код helper.py в разделе 5 + алгоритм timeline
│   ├── PROMPT.md
│   ├── README.md
│   └── skill/timing-analyzer/
│
└── report-builder/
    ├── SPEC.md                   ← готовый код helper.py в разделе 5
    ├── PROMPT.md
    ├── README.md
    └── skill/report-builder/
```

## Подготовка окружения

### MCP

В `~/.gigacode/settings.json` для Atlassian:

```json
"includeTools": ["jira_get_issue", "jira_search"]
```

### Файлы в рабочей директории

```
<рабочая_директория>/
├── Бэклог и цели.xlsx       ← файл Натальи (жёстко зафиксированное имя)
├── pipeline/                 ← создаётся первым скиллом
│   ├── enriched.json
│   ├── step-1-after-excel-parser.md
│   ├── step-2-after-jira-enricher.md
│   └── step-3-after-timing-analyzer.md
└── report.md                 ← финальный отчёт от report-builder
```

## Запуск pipeline

### Установка скиллов (one-shot для каждого + перенос в gigacode)

```bash
# Скилл 1
cd excel-parser
gigacode
# вставь PROMPT.md, получи skill/excel-parser/{SKILL.md, helper.py}
# проверь глазами: 2 файла, нет main.py, использован openpyxl, поиск по имени
cp -r skill/excel-parser ~/.gigacode/skills/

# Скилл 2
cd ../jira-enricher
gigacode
# проверь: 2 файла, нет mcp__... в Python, batch через stdin, customfield_22200 как массив
cp -r skill/jira-enricher ~/.gigacode/skills/

# Скилл 3
cd ../timing-analyzer
gigacode
# проверь: 2 файла, нет mcp__... в Python, expand=changelog, нет changelog для неактивных
cp -r skill/timing-analyzer ~/.gigacode/skills/

# Скилл 4
cd ../report-builder
gigacode
# проверь: 2 файла, нет MCP, условный рендеринг
cp -r skill/report-builder ~/.gigacode/skills/
```

### Прогон

В рабочей директории (где `Бэклог и цели.xlsx`):

```bash
gigacode
```

В чате последовательно:

1. `запусти excel-parser` → `pipeline/enriched.json` + `pipeline/step-1-after-excel-parser.md` (28 задач)
2. `запусти jira-enricher` → дополнен Jira-данными, создан `step-2-after-jira-enricher.md`
3. `запусти timing-analyzer` → дополнен фактом А/Р/Т, создан `step-3-after-timing-analyzer.md`
4. `запусти report-builder` → `report.md` в корне

**Время полного pipeline:** 3-5 минут.

После каждого шага можно открыть соответствующий `step-N.md` и проверить промежуточный результат.

## Что показать Наталье

После полного pipeline:

1. Открыть `report.md` в редакторе с поддержкой markdown
2. Главные секции для разговора:
   - **Дисклеймер** "Что в отчёте и чего нет" — проговорить ограничения
   - **Секция 5 "Застрявшие"** (если есть timing) — конкретный список задач для разбора
   - **Секция 3 "Срез по эпикам"** — обзор работы
   - **Секция 4 "Срез по командам"** — кто чем загружен

3. Спросить:
   - Это полезный взгляд?
   - Какие пороги маркера ⚠ лучше (× 2 от плана)?
   - Хочется ли v4 (контроль дат ИФТ/ПСИ/ПРОМ)?

## Roadmap

| Версия | Что добавляется | Куда |
|--------|------------------|------|
| **v3.2** | Точечные правки по фидбеку Натальи | в SPEC.md существующих скиллов |
| **v4** | Контроль плановых дат ИФТ/ПСИ/ПРОМ | новый скилл `dates-enricher` после jira-enricher. customfield известны (24300, 29500, 13700, 22601, 23703) |
| **v5** | Сберчат-сигналы при отклонениях | новый скилл `sberchat-notifier` после report-builder |
| **v6** | Развёрнутые дочерние эпиков + загрузка команд | расширение jira-enricher или новый `epic-expander` |
| **v7** | ML/автооценка БТ | "фантастика" по словам Натальи — не делается |

## Принципы дизайна

1. **Один скилл — одна вещь.** Если хочется добавить функцию — лучше новый скилл, не раздувать существующий.
2. **CONTRACT.md — единый источник истины** для структуры `enriched.json`. Изменения в структуре — сначала в CONTRACT.md, потом в коде.
3. **Только `helper.py`.** Запрещены лишние Python-файлы. SKILL.md — главная точка входа, helper.py — функции.
4. **MCP-вызовы — нативные tool calls агента**, не Python-обёртки. Граница: tool_call → агент → bash + stdin → helper.
5. **Жёстко зафиксированы:** имя Excel-файла, имя листа. Колонки ищутся **по имени заголовка** (case-insensitive подстрока).
6. **Календарные дни ≠ чел-дни** — всегда дисклеймер. Не выдумывать точный факт где его нет.
7. **Идемпотентность.** Любой скилл можно перезапустить. Перезаписываются только свои поля.

## Что делать если что-то сломалось

| Симптом | Диагноз | Действие |
|---------|---------|----------|
| Скилл создал `main.py` или другие лишние `.py` | Антипаттерн критический | Перегенерировать с явной ссылкой на запрет в SPEC.md |
| `NameError: mcp__Atlassian__jira_get_issue` | Скилл пытается вызвать MCP из Python | Перегенерировать с упором на раздел "КРИТИЧЕСКАЯ АРХИТЕКТУРА" |
| Контекст переполнился | `fields="*"` или changelog для всех задач | Проверить SPEC раздел 5 (формула вызова) |
| Колонка "Эпик" пустая | customfield_11400 не в fields= или нет fallback на issuelinks | Проверить SPEC jira-enricher раздел 7 |
| Колонка "Команда" показывает странное | customfield_22200 обрабатывается не как массив | Проверить extract_team в helper.py |
| Факт А/Р/Т кривой | Баг в build_timeline или aggregate_phase_days | Сравнить helper.py с готовым кодом в SPEC.md раздел 5 |
| Маркеры выглядят странно | Пороги не подходят | Обсудить с Натальей, скорректировать SPEC report-builder |
| `pipeline/enriched.json` пустой | Скилл не дошёл до записи | Проверить логи где упал |
| Скилл нашёл 3 задачи вместо 28 | Ручной XML-парсинг вместо openpyxl | Перегенерировать с упором на раздел "Антипаттерны критические" |
