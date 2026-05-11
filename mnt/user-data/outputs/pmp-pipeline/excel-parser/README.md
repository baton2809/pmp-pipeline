# excel-parser — первый скилл pipeline

Читает `Бэклог и цели.xlsx` и создаёт `.cache/enriched.json` для последующих скиллов pipeline.

## Место в pipeline

```
1. excel-parser       ← вы здесь
2. jira-enricher
3. timing-analyzer
4. report-builder
```

## Подготовка

В рабочей директории должен быть `Бэклог и цели.xlsx`. MCP не нужен (этот скилл не ходит в Jira).

## Запуск

### Шаг 1. Сгенерировать скилл

```bash
cd excel-parser
gigacode
```

Вставить промпт из `PROMPT.md`. Получить:
- `skill/excel-parser/SKILL.md`
- `skill/excel-parser/helper.py`

Перед установкой проверить:
- В `skill/excel-parser/` ровно 2 файла (SKILL.md и helper.py)
- В SKILL.md нет заглушек `# TODO`, нет полных Python-программ
- helper.py содержит только функции, не main()-блока с прогонкой

### Шаг 2. Установить

```bash
cp -r skill/excel-parser ~/.gigacode/skills/
```

### Шаг 3. Запустить

В рабочей директории (где лежит `Бэклог и цели.xlsx`):

```bash
gigacode
```

В чате: "запусти excel-parser".

После прогона будет `.cache/enriched.json` с распарсенным планом.

## Проверка результата

```bash
cat .cache/enriched.json | python3 -m json.tool | head -50
```

Должны быть:
- `metadata.skills_completed = ["excel-parser"]`
- `tasks` массив с 28 задачами (примерно)
- У каждой задачи заполнено `cr_key`, `task_name`, `plan.analytics/development/testing`
- `jira: null`, `timing: null` (для следующих скиллов)

## Следующий шаг

Запустить `jira-enricher` — он добавит данные из Jira в тот же `enriched.json`.
