# jira-enricher v3.1 — второй скилл pipeline

Читает `pipeline/enriched.json`, для каждой задачи запрашивает данные из Jira, перезаписывает файл с дополнениями и создаёт читаемый снимок.

## Изменения v3.1 vs v3.0

- ✅ **Архитектурный фикс:** строгое разделение "tool calls делает агент в чате" vs "Python через bash работает только с данными которые ему передали через stdin". В v3.0 GigaCode пытался вызвать MCP из Python, получал `NameError`.
- ✅ Batching по 5 задач — после каждого батча агент передаёт результаты в helper через `echo '...' | python3 helper.py merge-batch`
- ✅ Папка `pipeline/` (видимая)
- ✅ Создаётся `pipeline/step-2-after-jira-enricher.md`
- ✅ Готовый код `helper.py` в SPEC.md раздел 7 (включая CLI с подкомандами: `merge-batch`, `merge-epics`, `finalize`, `write-step2`)

## Место в pipeline

```
1. excel-parser       → pipeline/enriched.json
2. jira-enricher      ← вы здесь
3. timing-analyzer
4. report-builder
```

## Подготовка

В `~/.gigacode/settings.json`:

```json
"includeTools": ["jira_get_issue", "jira_search"]
```

Перед запуском должен отработать `excel-parser`.

## Запуск

### Шаг 1. Сгенерировать

```bash
cd jira-enricher
gigacode
```

Вставить промпт из `PROMPT.md`.

### Шаг 2. Проверить перед установкой

В `skill/jira-enricher/` должно быть **ровно 2 файла**:

**Главные проверки (это и был корень провала в v3.0):**

- В SKILL.md tool call описан **на естественном языке** ("сделать tool call jira_get_issue с параметрами..."), не как Python-функция
- В helper.py **НЕТ** строк типа `mcp__Atlassian__jira_get_issue(...)` или `from mcp_atlassian import ...`
- В SKILL.md есть упоминание batching по 5 задач
- В SKILL.md есть передача через stdin: `echo '...' | python3 helper.py merge-batch`
- helper.py содержит CLI entry-points: `merge-batch`, `merge-epics`, `finalize`, `write-step2`
- `extract_team` в helper.py обрабатывает `customfield_22200` как **массив строк** `["PALM.CSP.K7M"]`
- `fields=` в tool call — точный список из 15 полей, **не** `"*"`
- **Нет** `expand="changelog"` (это работа следующего скилла)

### Шаг 3. Установить и запустить

```bash
cp -r skill/jira-enricher ~/.gigacode/skills/
gigacode
```

В чате: "запусти jira-enricher".

## Сколько времени занимает

- 28 задач × `jira_get_issue` = ~30 сек (с паузами по 0.1с)
- + ~10-15 эпиков × `jira_get_issue` для имён = ~15 сек
- + ~10-15 эпиков × `jira_search` для дочерних = ~15 сек

Итого ~1 минута.

## Проверка результата

```bash
ls -la pipeline/
# Должны быть enriched.json + step-1.md + step-2.md
```

```bash
cat pipeline/step-2-after-jira-enricher.md | head -30
```

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
print('Skills:', d['metadata']['skills_completed'])
print('Tasks:', len(d['tasks']))
print('Tasks found:', sum(1 for t in d['tasks'] if t.get('jira', {}).get('found')))
print('Epics:', len(d.get('epics', [])))
"
```

Должно вывести:
```
Skills: ['excel-parser', 'jira-enricher']
Tasks: 28
Tasks found: ~26-28
Epics: ~10-15
```

## Если что-то пошло не так

| Симптом | Действие |
|---------|----------|
| `NameError: mcp__Atlassian__jira_get_issue` | Скилл сгенерил вызов MCP в Python — перегенерировать с упором на раздел "КРИТИЧЕСКАЯ АРХИТЕКТУРА" в SPEC.md |
| Колонка "Команда" пустая или странная | Проверить `extract_team` в helper.py — должна обрабатывать массив строк |
| Колонка "Эпик" пустая для ASFC-задач | Проверить что `customfield_11400` в `fields=` |
| Колонка "Эпик" пустая для CRSIGMA | Проверить fallback на `issuelinks "Implement in"` в `extract_epic` |
| Часть задач не найдены | Это нормально если CR-ключ изменился. Они попадут в секцию "Не найдено" в финальном отчёте |
| Контекст переполнился | `fields="*"` вместо точного списка. Перегенерировать. |

## Следующий шаг

- Запустить `timing-analyzer` — добавит факт А/Р/Т из changelog для активных задач
- Или сразу `report-builder` если факт не нужен (отчёт без колонок Факт А/Р/Т)
