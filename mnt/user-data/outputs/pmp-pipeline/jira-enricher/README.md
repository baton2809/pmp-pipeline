# jira-enricher — второй скилл pipeline

Читает `.cache/enriched.json` (созданный `excel-parser`), для каждой задачи запрашивает данные из Jira (статус, эпик, команда), перезаписывает тот же файл с дополнениями.

## Место в pipeline

```
1. excel-parser       → .cache/enriched.json
2. jira-enricher      ← вы здесь
3. timing-analyzer
4. report-builder
```

## Подготовка

В `~/.gigacode/settings.json` для Atlassian MCP:

```json
"includeTools": ["jira_get_issue", "jira_search"]
```

Перед запуском должен отработать `excel-parser` — файл `.cache/enriched.json` должен существовать.

## Запуск

### Шаг 1. Сгенерировать скилл

```bash
cd jira-enricher
gigacode
```

Вставить промпт из `PROMPT.md`. Получить `skill/jira-enricher/SKILL.md` и `helper.py`.

Перед установкой проверить:
- Ровно 2 файла в `skill/jira-enricher/`
- В SKILL.md в шагах вызова MCP — естественный язык, не `result = mcp_jira_get_issue(...)`
- Параметр `fields=` содержит точный список (раздел 5 SPEC.md), не `"*"`
- Нет `expand="changelog"` — это работа следующего скилла

### Шаг 2. Установить

```bash
cp -r skill/jira-enricher ~/.gigacode/skills/
```

### Шаг 3. Запустить

В рабочей директории (где `Бэклог и цели.xlsx` и `.cache/enriched.json`):

```bash
gigacode
```

В чате: "запусти jira-enricher".

После прогона:
- `.cache/enriched.json` перезаписан с заполненной секцией `jira` у каждой задачи и массивом `epics`
- В чате сводка: сколько задач найдено, сколько уникальных эпиков, сколько команд резолвилось через customfield vs assignee

## Сколько времени занимает

- 28 задач × `jira_get_issue` + 0.1 сек паузы = ~30 сек
- + 10-15 уникальных эпиков × `jira_get_issue` для имён = ~15 сек
- + 10-15 эпиков × `jira_search` для дочерних = ~15 сек

Итого ~1 минута.

## Проверка результата

```bash
python3 -c "
import json
d = json.load(open('.cache/enriched.json'))
print('Skills:', d['metadata']['skills_completed'])
print('Tasks:', len(d['tasks']))
print('Tasks with jira:', sum(1 for t in d['tasks'] if t.get('jira', {}).get('found')))
print('Epics:', len(d.get('epics', [])))
"
```

Должно вывести:
```
Skills: ['excel-parser', 'jira-enricher']
Tasks: 28
Tasks with jira: ~26-28
Epics: ~10-15
```

## Следующий шаг

- Запустить `timing-analyzer` — добавит факт А/Р/Т из changelog для активных задач
- Или сразу `report-builder` если факт не нужен (получите отчёт без колонок "Факт А/Р/Т")
