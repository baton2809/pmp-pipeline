# excel-parser v3.1 — первый скилл pipeline

Читает `Бэклог и цели.xlsx` и создаёт `pipeline/enriched.json` + `pipeline/step-1-after-excel-parser.md`.

## Изменения v3.1 vs v3.0

- ✅ Поиск колонок **по имени заголовка** (case-insensitive подстрока), не по фиксированной букве — это вернуло **28 задач** вместо 3
- ✅ Только `openpyxl`, явный запрет на ручной XML-парсинг
- ✅ Папка `pipeline/` (видимая) вместо `.cache/` (скрытая)
- ✅ Дополнительно создаётся `pipeline/step-1-after-excel-parser.md` — читаемый снимок
- ✅ Готовый код `helper.py` прямо в SPEC.md раздел 7 — снижает риск кривой реализации

## Место в pipeline

```
1. excel-parser       ← вы здесь
2. jira-enricher
3. timing-analyzer
4. report-builder
```

## Подготовка

В рабочей директории должен быть `Бэклог и цели.xlsx`. MCP не нужен.

## Запуск

### Шаг 1. Сгенерировать скилл

```bash
cd excel-parser
gigacode
```

Вставить промпт из `PROMPT.md`. Получить:
- `skill/excel-parser/SKILL.md`
- `skill/excel-parser/helper.py`

### Шаг 2. Проверить перед установкой

В `skill/excel-parser/` должно быть **ровно 2 файла**:
- В SKILL.md нет заглушек `# TODO`, нет полных Python-программ
- helper.py использует `openpyxl.load_workbook` (не парсит XML вручную)
- `find_header_row` ищет 'cr' в строках 1-5
- `detect_columns` ищет по подстроке (`'cr'`, `'задача'`, `'аналитика'` и т.д.)
- `parse_cr_key` использует `re.search`, **не** `re.match`/`re.fullmatch`
- Нет других `.py` файлов кроме `helper.py`

### Шаг 3. Установить

```bash
cp -r skill/excel-parser ~/.gigacode/skills/
```

### Шаг 4. Запустить

В рабочей директории (где `Бэклог и цели.xlsx`):

```bash
gigacode
```

В чате: "запусти excel-parser".

## Проверка результата

После прогона будет два файла в `pipeline/`:
- `enriched.json` — данные для следующих скиллов
- `step-1-after-excel-parser.md` — читаемый снимок

```bash
ls -la pipeline/
cat pipeline/step-1-after-excel-parser.md | head -20
```

Также можно проверить через Python:

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
print('Skills:', d['metadata']['skills_completed'])
print('Tasks:', len(d['tasks']))
print('First task:', d['tasks'][0]['cr_key'])
"
```

Должно быть `Tasks: 28`.

## Следующий шаг

Запустить `jira-enricher` — добавит данные из Jira в тот же `enriched.json`.
