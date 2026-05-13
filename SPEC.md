# SPEC: excel-parser

> Первый скилл pipeline pmp-vs-jira. Читает `Бэклог и цели.xlsx`, лист `Q2_26_оценки_new_name`, извлекает план задач квартала и сохраняет в `pipeline/enriched.json`.
>
> Никаких вызовов MCP. Только Excel. Это база для следующих скиллов pipeline.

## 1. Контекст и место в pipeline

```
[Бэклог и цели.xlsx]
        │
        ▼
   excel-parser ◄── (этот скилл)
        │
        ▼
   pipeline/enriched.json
        │
        ▼
   jira-enricher → timing-analyzer → report-builder
```

## 2. Цели

- Прочитать жёстко зафиксированный файл `Бэклог и цели.xlsx`
- Найти жёстко зафиксированный лист `Q2_26_оценки_new_name`
- Извлечь 28 задач квартала с CR-ключами, названиями, инициативами, планом А/Р/Т
- Создать структуру `enriched.json` согласно CONTRACT.md, секция "После excel-parser"
- Сохранить в `pipeline/enriched.json` рабочей директории

## 3. Анти-цели

- **НЕ** ходить в Jira (это делает следующий скилл)
- **НЕ** генерировать markdown-отчёт (это делает `report-builder`)
- **НЕ** создавать Python-проект — только `helper.py` если необходим
- **НЕ** работать с другими листами/файлами кроме зафиксированных

## 4. Вход и выход

### Вход

- Файл `Бэклог и цели.xlsx` в текущей рабочей директории (жёстко зафиксировано)
- Лист `Q2_26_оценки_new_name` (жёстко зафиксировано)

Если файл отсутствует — сообщить пользователю с точным именем файла, не пытаться найти альтернативы. Если лист отсутствует — то же самое.

### Выход

`pipeline/enriched.json` в текущей рабочей директории. Структура — см. CONTRACT.md секция "После excel-parser".

Папку `pipeline/` создать если не существует.

## 5. Структура Excel — поиск колонок по имени заголовка

**КРИТИЧНО:** колонки находятся **по имени в строке заголовков** (case-insensitive, по подстроке), **НЕ по фиксированной букве колонки**. Это решение проверено в v2 (`pmp-vs-jira-light`) — там нашли все 28 задач именно через поиск по имени.

**Почему НЕ по позиции букв (B, C, F, G, ...):** в xlsx XML пустые ячейки физически пропускаются. Если ваш парсер не использует `openpyxl` (который сам это нормализует), а парсит XML вручную — индексы колонок поедут. Поиск по имени защищает от этой проблемы.

### Имена колонок которые ищем

Все имена case-insensitive, поиск по подстроке в значении ячейки заголовка:

| Логическое имя | Поиск по подстроке | Что хранит |
|----------------|---------------------|------------|
| `cr` | `'cr'` | CR-ключ Jira (URL или строка) |
| `task` | `'задача'` | Название задачи |
| `initiative` | `'инициатива'` | Инициатива (для группировки) |
| `customer` | `'заказчик'` | Заказчик (опционально) |
| `analytics` | `'аналитика'` | Оценка А (чд) — встречается ДВА раза (v1 и v2) |
| `development` | `'разработка'` | Оценка Р (чд) — встречается ДВА раза |
| `testing` | `'тестирование'` | Оценка Т (чд) — встречается ДВА раза |

**Важно про дубликаты колонок:** в файле Натальи колонки `Аналитика`, `Разработка`, `Тестирование` встречаются **дважды** — для первой и второй версии оценок. При поиске собираем **список всех позиций** для каждого имени.

### Логика чтения файла

1. **Открыть файл** через `openpyxl.load_workbook(path, data_only=True)`. **НЕ парсить xlsx как XML вручную** — см. антипаттерны.
2. **Найти лист** `Q2_26_оценки_new_name`.
3. **Найти строку заголовков** — первая строка где есть ячейка со значением содержащим `'cr'` (case-insensitive). Обычно это строка 1.
4. **Распознать позиции колонок** — пройти по ячейкам строки заголовков, для каждого имени из таблицы выше собрать список индексов.
5. **Найти первую строку данных** — первая строка ПОСЛЕ заголовков где в колонке `cr` есть непустое значение похожее на CR-ключ (regex поиск).

### Выбор версии плана

Колонки `аналитика`, `разработка`, `тестирование` встречаются дважды — это две версии плана:
- **Первая позиция (раньше по индексу)** — первая версия оценок (J/K/L в файле Натальи)
- **Вторая позиция (позже)** — вторая, актуальная версия (R/S/T)

Для каждой задачи и каждой фазы (А/Р/Т) — **брать значение из последней непустой позиции**. То есть если v2 заполнена — берём v2, иначе v1.

`source_version`:
- `"v2"` — если хотя бы одна из фаз взята из второй версии
- `"v1"` — если все фазы из первой версии
- `"none"` — если ни одна не заполнена. Задача всё равно попадает в `tasks`, просто без оценок.

### Парсинг CR-ключа

Из значения колонки `cr`:
1. `str(value).strip()` — преобразовать в строку и убрать пробелы
2. Если пусто или `None` → задача без CR, **в `tasks` НЕ добавляется**, регистрируется в `metadata.skipped_rows`
3. Применить regex `r'(ASFC|CRSIGMA|OCRED|TIBDS|ASFS)-\d+'` через `re.search` (НЕ `re.match` — нужен поиск подстроки, потому что CR может быть в URL)
4. Если найдено — `match.group(0)` это ключ
5. Если не найдено — в `skipped_rows` с пометкой "некорректный формат CR в строке N"

**Примеры что должно работать:**
- `' https://jira.delta.sbrf.ru/browse/CRSIGMA-26516'` → `CRSIGMA-26516` ✓
- `'https://jira.delta.sbrf.ru/browse/CRSIGMA-23749'` → `CRSIGMA-23749` ✓
- `'ASFC-58741'` → `ASFC-58741` ✓
- `'TIBDS-8245'` → `TIBDS-8245` ✓
- `'Какой-то текст'` → пропуск, в skipped_rows
- `None` или `''` → пропуск, в skipped_rows

## 6. Steps

### Step 1. Проверить файл

Проверить что `Бэклог и цели.xlsx` существует в текущей директории. Если нет — сообщение пользователю:

> Файл `Бэклог и цели.xlsx` не найден в текущей директории. Убедитесь что вы запустили GigaCode в правильной папке.

Завершить.

### Step 2. Открыть лист

Открыть лист `Q2_26_оценки_new_name`. Если листа нет — перечислить доступные листы, попросить пользователя проверить. Завершить.

Реализация — через `helper.py` функция `open_sheet()` использующая `openpyxl.load_workbook(..., data_only=True)`.

### Step 3. Найти строку заголовков и распознать колонки

Через `helper.find_headers(worksheet)`:

1. Идти по строкам от 1 до 5 (заголовки обычно в начале)
2. Для каждой строки проверить — есть ли в ней ячейка со значением содержащим `'cr'` (case-insensitive)
3. Первая такая строка — строка заголовков
4. Запомнить её номер как `header_row`

Через `helper.detect_columns(worksheet, header_row)`:

5. Пройти по всем ячейкам строки `header_row` (от колонки 1 до `worksheet.max_column`)
6. Для каждой ячейки взять значение, привести к строке, lowercase, strip
7. Если значение содержит `'cr'` — добавить индекс в `columns['cr']`
8. Если содержит `'задача'` — `columns['task']`
9. Аналогично для `'инициатива'`, `'заказчик'`, `'аналитика'`, `'разработка'`, `'тестирование'`
10. Вернуть словарь `columns: Dict[str, List[int]]` — для каждого имени список найденных индексов колонок

**Валидация:**
- Если `columns['cr']` пуст — сообщить "не найдена колонка CR в строке заголовков", завершить
- Если `columns['task']` пуст — продолжить, в отчёте у задач `task_name = ''`, в `metadata` пометка
- Если `columns['analytics']` имеет 0 элементов — продолжить, план будет null

### Step 4. Найти первую строку данных

Через `helper.find_first_data_row(worksheet, header_row, cr_col_indices)`:

1. Идти со строки `header_row + 1` до `worksheet.max_row`
2. Для каждой строки проверить **все** колонки из `columns['cr']` (может быть несколько)
3. Первая строка где **хотя бы в одной из cr-колонок** есть значение содержащее regex `(ASFC|CRSIGMA|OCRED|TIBDS|ASFS)-\d+` — это начало данных
4. Запомнить как `first_data_row`

### Step 5. Пройти по строкам, извлечь задачи

Для каждой строки от `first_data_row` до `worksheet.max_row`:

1. **Извлечь CR-ключ:** пройти по `columns['cr']`, для каждой колонки получить значение, применить парсинг (см. раздел 5 "Парсинг CR-ключа"). Взять **первый успешно распарсенный** ключ.
2. Если CR не найден — добавить в `metadata.skipped_rows`, продолжить со следующей строки
3. **Извлечь название** — из первой непустой колонки `columns['task']`, применить `helper.clean_text`
4. **Извлечь инициативу** — из `columns['initiative']`, очистить
5. **Извлечь заказчика** — из `columns['customer']` (опционально)
6. **Извлечь план А/Р/Т** — для каждой фазы пройти по `columns['analytics']` / `['development']` / `['testing']`, **взять последнее непустое числовое значение** (это вторая, актуальная версия)
   - Если все значения пусты — `null`
   - Считать `plan.total = (analytics or 0) + (development or 0) + (testing or 0)` если хотя бы одно не null, иначе `null`
   - Определить `source_version`:
     - `"v2"` — если хотя бы для одной фазы взято значение НЕ из первой позиции (т.е. была вторая колонка с числом)
     - `"v1"` — если для всех фаз взято из первой позиции
     - `"none"` — если все null
7. **Создать объект `task`** согласно CONTRACT.md секция "После excel-parser":
   ```json
   {
     "cr_key": "<распарсенный ключ>",
     "task_name": "<название>",
     "initiative": "<инициатива>",
     "customer": "<заказчик или null>",
     "plan": {
       "analytics": <число или null>,
       "development": <число или null>,
       "testing": <число или null>,
       "total": <сумма или null>,
       "source_version": "v1" | "v2" | "none"
     },
     "jira": null,
     "timing": null
   }
   ```
8. Добавить в `tasks` массив

### Step 6. Сохранить json и markdown-снимок

1. Создать папку `pipeline/` если её нет (`os.makedirs('pipeline', exist_ok=True)`)
2. Сформировать полную структуру согласно CONTRACT.md секция "После excel-parser":
   ```python
   enriched = {
     "metadata": {
       "source_file": "Бэклог и цели.xlsx",
       "sheet": "Q2_26_оценки_new_name",
       "parsed_at": "<now ISO 8601>",
       "enriched_at": None,
       "timing_at": None,
       "report_generated_at": None,
       "scope_version": "v3.1",
       "skills_completed": ["excel-parser"],
       "skipped_rows": [...],
       "tasks_count": len(tasks)
     },
     "tasks": [...],
     "epics": []
   }
   ```
3. Записать в `pipeline/enriched.json`:
   ```python
   with open('pipeline/enriched.json', 'w', encoding='utf-8') as f:
     json.dump(enriched, f, indent=2, ensure_ascii=False)
   ```
4. **Создать markdown-снимок** `pipeline/step-1-after-excel-parser.md` — это **видимый** артефакт для пользователя, чтобы он мог сразу проверить результат не открывая json.

   Структура (см. CONTRACT.md секция "step-1-after-excel-parser.md"):
   - Заголовок: `# Снимок после excel-parser`
   - Дата, источник, число задач
   - Таблица всех задач: `#`, `CR`, `Название` (обрезано до 50 символов), `План А/Р/Т (Σ)`, `Версия плана`
   - Подсказка "Следующий шаг: запустите `jira-enricher`"
   
   Реализация — `helper.write_step1_markdown(enriched)`.

### Step 7. Сообщить пользователю

В чат вывести краткую сводку:
- Обработан файл: `Бэклог и цели.xlsx`, лист `Q2_26_оценки_new_name`
- Строка заголовков: N
- Первая строка данных: M
- Распознано колонок: `cr` (K позиций), `task`, `initiative`, `аналитика` (2 позиции), и т.д.
- **Извлечено задач: N** ← главное число, должно быть **28** для текущего файла Натальи
- Пропущено строк (без CR): M
- Созданы файлы:
  - `pipeline/enriched.json` (данные для следующих скиллов)
  - `pipeline/step-1-after-excel-parser.md` (читаемый снимок)
- Следующий шаг: запустите `jira-enricher`

## 7. Файлы которые скилл может создавать

| Файл | Назначение |
|------|------------|
| `pipeline/enriched.json` | Основной выход, согласно CONTRACT.md |
| `helper.py` (рядом со SKILL.md в `~/.gigacode/skills/excel-parser/`) | Вспомогательные функции |

### Минимальный набор функций которые должны быть в `helper.py`

```python
# helper.py — функции для excel-parser

import openpyxl
import re
import json
import os
from datetime import datetime, timezone

CR_PATTERN = re.compile(r'(ASFC|CRSIGMA|OCRED|TIBDS|ASFS)-\d+')

def open_workbook(path: str):
    """Открыть xlsx через openpyxl. ТОЛЬКО openpyxl, не ручной XML."""
    return openpyxl.load_workbook(path, data_only=True)

def get_sheet(workbook, sheet_name: str):
    """Получить лист по имени."""
    if sheet_name not in workbook.sheetnames:
        return None
    return workbook[sheet_name]

def find_header_row(worksheet, max_search_rows: int = 5) -> int | None:
    """Найти строку заголовков (первая строка где есть ячейка содержащая 'cr' case-insensitive).
    Возвращает 1-based индекс строки или None."""
    for row_idx in range(1, max_search_rows + 1):
        for col_idx in range(1, worksheet.max_column + 1):
            value = worksheet.cell(row_idx, col_idx).value
            if value and 'cr' in str(value).lower():
                return row_idx
    return None

def detect_columns(worksheet, header_row: int) -> dict[str, list[int]]:
    """Распознать позиции колонок по подстроке в заголовке.
    Возвращает dict где значения — списки индексов (может быть несколько колонок с одинаковым именем)."""
    name_patterns = {
        'cr': 'cr',
        'task': 'задача',
        'initiative': 'инициатива',
        'customer': 'заказчик',
        'analytics': 'аналитика',
        'development': 'разработка',
        'testing': 'тестирование',
    }
    columns = {key: [] for key in name_patterns}
    for col_idx in range(1, worksheet.max_column + 1):
        cell_value = worksheet.cell(header_row, col_idx).value
        if cell_value is None:
            continue
        value_lower = str(cell_value).lower().strip()
        for key, pattern in name_patterns.items():
            if pattern in value_lower:
                columns[key].append(col_idx)
    return columns

def parse_cr_key(value) -> str | None:
    """Извлечь CR-ключ из значения ячейки. None если не удалось."""
    if value is None:
        return None
    text = str(value).strip()
    if not text:
        return None
    match = CR_PATTERN.search(text)  # ВАЖНО: search, не match!
    return match.group(0) if match else None

def clean_text(value) -> str:
    """Очистить текст от \\n, \\r, \\t, свернуть пробелы, strip."""
    if value is None:
        return ''
    text = str(value)
    text = text.replace('\n', ' ').replace('\r', ' ').replace('\t', ' ')
    text = re.sub(r'\s+', ' ', text).strip()
    return text

def extract_plan(row_idx: int, worksheet, columns: dict[str, list[int]]) -> dict:
    """Извлечь план А/Р/Т для строки. Берёт последнее непустое значение для каждой фазы.
    Определяет source_version: v2 если хоть одно значение взято не из первой позиции, иначе v1, иначе none."""
    phases = {'analytics': None, 'development': None, 'testing': None}
    used_v2_for_any_phase = False
    
    for phase_key in phases:
        positions = columns.get(phase_key, [])
        # Идём по позициям, последняя непустая — наше значение
        for i, col_idx in enumerate(positions):
            cell_value = worksheet.cell(row_idx, col_idx).value
            if cell_value is not None and cell_value != '':
                try:
                    phases[phase_key] = float(cell_value)
                    if i > 0:  # это не первая позиция, значит взяли из v2 или позже
                        used_v2_for_any_phase = True
                except (ValueError, TypeError):
                    pass  # не число — игнорируем
    
    total = None
    any_filled = any(v is not None for v in phases.values())
    if any_filled:
        total = sum((v or 0) for v in phases.values())
    
    if not any_filled:
        source_version = 'none'
    elif used_v2_for_any_phase:
        source_version = 'v2'
    else:
        source_version = 'v1'
    
    return {
        'analytics': phases['analytics'],
        'development': phases['development'],
        'testing': phases['testing'],
        'total': total,
        'source_version': source_version,
    }

def save_enriched(data: dict, path: str = 'pipeline/enriched.json'):
    """Сохранить JSON в файл, создавая pipeline/ если её нет."""
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, 'w', encoding='utf-8') as f:
        json.dump(data, f, indent=2, ensure_ascii=False)

def write_step1_markdown(enriched: dict, path: str = 'pipeline/step-1-after-excel-parser.md'):
    """Сохранить читаемый markdown-снимок после excel-parser."""
    md = []
    md.append("# Снимок после excel-parser\n")
    md.append(f"**Дата:** {enriched['metadata']['parsed_at']}\n")
    md.append(f"**Источник:** {enriched['metadata']['source_file']}, лист {enriched['metadata']['sheet']}\n")
    md.append(f"**Задач извлечено:** {len(enriched['tasks'])}\n")
    skipped = len(enriched['metadata'].get('skipped_rows', []))
    md.append(f"**Пропущено строк (без CR):** {skipped}\n")
    md.append("\n## Задачи плана\n")
    md.append("| # | CR | Название | План А/Р/Т (Σ) | Версия плана |")
    md.append("|---|-----|----------|-----------------|---------------|")
    for i, task in enumerate(enriched['tasks'], 1):
        name = (task.get('task_name') or '')[:50]
        if len(task.get('task_name') or '') > 50:
            name = name.rsplit(' ', 1)[0] + '…'
        p = task.get('plan') or {}
        a = p.get('analytics')
        r = p.get('development')
        t = p.get('testing')
        total = p.get('total')
        if total is None:
            plan_str = '—'
        else:
            def fmt(v):
                return str(int(v)) if v is not None else '0'
            plan_str = f"{fmt(a)}/{fmt(r)}/{fmt(t)} ({int(total)})"
        sv = p.get('source_version', 'none')
        md.append(f"| {i} | {task['cr_key']} | {name} | {plan_str} | {sv} |")
    md.append("\n## Следующий шаг\n")
    md.append("Запустить `jira-enricher` — добавит статусы, эпики, команды из Jira.")
    with open(path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(md))

def now_iso() -> str:
    """Текущее время в ISO 8601 с offset."""
    return datetime.now(timezone.utc).astimezone().isoformat()
```

**Главное про эти функции:**
- Используют **только** `openpyxl`, никакого ручного XML-парсинга
- Поиск колонок по **имени** (case-insensitive подстрока)
- `parse_cr_key` использует **`re.search`** (не `match`/`fullmatch`)
- Возвращают чистые значения, готовые к записи в json

### Что ЗАПРЕЩЕНО

- `main.py`, `process.py`, `run_*.py`, `generate_*.py`
- `__pycache__/`, `requirements.txt`, `pyproject.toml`, виртуальные окружения
- Любые `.py` файлы кроме одного `helper.py`
- Запуск скилла как Python-приложения — SKILL.md остаётся главной точкой входа, `helper.py` это набор функций

## 8. КРИТИЧНО: формат SKILL.md

SKILL.md — это **инструкция для агента**, а не Python-скрипт. Каждый Step описан на естественном языке. Когда нужны вычисления — агент либо пишет inline `python3 -c '...'` через bash, либо импортирует функцию из `helper.py`.

Пример **правильного** шага:
```
Шаг 2. Открыть Excel-файл.

Импортировать `helper.py` (он рядом со SKILL.md в скиллах) и вызвать:
  workbook = helper.open_workbook("Бэклог и цели.xlsx")
  sheet = helper.get_sheet(workbook, "Q2_26_оценки_new_name")

Если функция вернула ошибку — вывести сообщение пользователю и завершить.
```

**Запрещено** в SKILL.md:
- Полные Python-программы внутри инструкции
- `# TODO`, `# здесь будет код`, заглушки
- Описания вроде "напишите скрипт который..." — скилл сам описывает что делать

## 9. Guardrails

- READ-ONLY для Excel: только читаем, не модифицируем
- Никаких MCP-вызовов
- Только один python-файл: `helper.py`
- Жёстко зафиксированы имя файла и лист
- Не создаём `report.md` — это работа другого скилла
- Не интерпретируем данные — только извлекаем

## 10. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| Файл `Бэклог и цели.xlsx` не найден | Сообщить точное имя, завершить |
| Лист `Q2_26_оценки_new_name` не найден | Перечислить доступные листы, завершить |
| В строке заголовков нет колонки `cr` | Сообщить "не найдена колонка с заголовком 'CR'", показать какие заголовки нашли, завершить |
| Ни в одной строке нет CR-ключа | Сообщить "не найдены CR-ключи в файле", завершить |
| Строка с пустой колонкой `cr` | Пропустить, **НЕ** добавлять в skipped_rows (нормальный случай) |
| Строка с мусором в колонке `cr` (например "TBD" или "?") | skipped_rows с пометкой "не удалось распознать CR в строке N" |
| Строка с OCRED-ключом но без CR-ключа в колонке `cr` | Пропустить — OCRED это отдельный трекер, не Jira которую обрабатываем. Не в skipped_rows. |
| Дубликат CR-ключа в плане (как `CRSIGMA-22127` в Q2) | Обе строки добавляются как отдельные задачи. В `metadata.duplicates` отметка |
| Колонки v2 (вторая позиция А/Р/Т) пустые но v1 (первая позиция) заполнены | Используем v1 версию плана, `source_version = "v1"` |
| Все версии плана пустые | `plan` с null полями, `source_version = "none"`, задача всё равно в tasks |
| Excel содержит несколько листов | Используем только `Q2_26_оценки_new_name`, остальные игнорируем |
| Заголовки в строке 2, а не 1 | Логика поиска по 'cr' справится — header_row будет = 2 |

## 11. Антипаттерны

### Критические (приводили к багам в прошлых итерациях)

- **Ручной парсинг xlsx как XML.** В прошлой попытке (v3 pmp-vs-jira) GigaCode распаковал xlsx как zip и парсил `xl/worksheets/sheet*.xml` вручную через `findall('{http://schemas.openxmlformats.org/spreadsheetml/2006/main}c')`. **Результат:** колонки поехали (пустые ячейки в XML пропускаются, индексы сдвигаются). Из 28 задач нашлось 3. **Использовать ТОЛЬКО `openpyxl.load_workbook()`** — он сам нормализует пропуски через атрибут `r="B3"`.

- **Жёсткая привязка к буквам колонок (B, C, F, G, ...).** При ручном XML-парсинге это особенно опасно. Даже с openpyxl — менее надёжно чем поиск по имени заголовка. **Всегда искать по имени** (case-insensitive, по подстроке) — это проверено в v2 `pmp-vs-jira-light` и дало 28 задач.

- **Создание лишних .py файлов** (`main.py`, `process.py`, `generate_*.py`, `run_*.py`) — в прошлой v3 GigaCode создал 4 файла и pipeline поломался. **Только** `helper.py`. Никаких `__pycache__`, `requirements.txt`, виртуальных окружений.

- **Заглушки в SKILL.md** (`# здесь будет парсинг`, `# TODO`) — SKILL.md должен быть готов к работе сразу, без TODO.

### Обычные

- Записывать в `enriched.json` сырой ответ openpyxl без очистки текста (`\n`, `\r`)
- Падать на одной плохой строке вместо `skipped_rows`
- Не создавать `pipeline/` если её нет (нужно `os.makedirs(..., exist_ok=True)`)
- Использовать `re.match` или `re.fullmatch` для CR-ключа — нужен `re.search` (потому что ключ может быть в URL, не в начале строки)
- Сохранять JSON без `ensure_ascii=False` — кириллица превратится в `\u...`
- Считать source_version как "v2" если только R/S/T пусты — нужно проверять что **хотя бы одна v2-позиция была не пуста**

## 12. Критерий успеха

После запуска:
1. Файл `pipeline/enriched.json` создан и валиден (структура совпадает с CONTRACT.md)
2. Количество задач в массиве `tasks` соответствует количеству CR-ключей в Excel
3. У каждой задачи заполнены `cr_key`, `task_name`, `initiative`, `plan`
4. `metadata.skills_completed = ["excel-parser"]`
5. В чате выведена сводка
6. Никаких лишних файлов в рабочей директории

## 13. Что отложено на следующие версии

- v3.1: парсить лист `Q1_26_PMP` (если попросят)
- v4: парсить дополнительные колонки с плановыми датами ИФТ/ПСИ/ПРОМ из Excel (если они там есть)
