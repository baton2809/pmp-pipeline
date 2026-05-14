# jira-enricher v3.2 — второй скилл pipeline

Читает `pipeline/enriched.json`, для каждой задачи запрашивает данные из Jira, агрегирует уникальные эпики и считает их дочерние задачи. Перезаписывает файл с дополнениями и создаёт читаемый снимок.

## Изменения v3.2 vs v3.1

**Что пошло не так в v3.1:**
- 28/28 задач корректно обрабатывались (`task.jira` заполнялся)
- Но массив `enriched.epics[]` оставался **пустым**
- Из-за этого секция "Срез по эпикам" в финальном `report.md` показывала `*Эпики не найдены.*`
- Корневая причина: после цикла `merge_batch` агент не вызывал агрегацию эпиков

**Что починено в v3.2:**
- ✅ Новая функция `aggregate_epics()` — собирает уникальные эпики из `task.jira.epic`
- ✅ Новая функция `update_epic_children(key, count)` — обновляет counter одного эпика
- ✅ Новая функция `list_epics_to_count` — выводит JSON эпиков без counter (для следующего шага)
- ✅ Явный шаг в SKILL.md "после всех `merge-batch` — `aggregate-epics`"
- ✅ Колонка в `step-2-after-jira-enricher.md` — **"Поток"** (не "Команда"), консистентность с финальным отчётом v3.2
- ✅ `extract_epic` использует `outward_issue` с подчёркиванием (как реально в Сбер-MCP)
- ✅ Все функции протестированы: идемпотентность, корректность агрегации, сохранение counters при повторном вызове

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

**Главные проверки (после v3.1 был корень провала):**
- В helper.py есть **3 новые функции**: `aggregate_epics`, `update_epic_children`, `list_epics_to_count`
- CLI entry-points включают: `aggregate-epics`, `update-epic-children`, `list-epics-to-count`
- В SKILL.md есть **ЯВНЫЙ Step 3** "после всех `merge-batch` — `aggregate-epics`"
- В SKILL.md есть **ЯВНЫЙ Step 5** "для каждого эпика — `jira_search` + `update-epic-children`"
- В `write_step2_markdown` колонка таблицы называется **"Поток"** (не "Команда")

**Архитектурные (не сломаны с v3.1):**
- В SKILL.md tool call описан на естественном языке
- В helper.py **НЕТ** `mcp__Atlassian__jira_get_issue` или `from mcp_atlassian import`
- Batching по 5 задач: `echo '...' | python3 helper.py merge-batch`
- `extract_team` обрабатывает `customfield_22200` как массив строк `["PALM.CSP.K7M"]`
- `extract_epic` ищет `outward_issue` (с подчёркиванием!) для CRSIGMA-задач
- `fields=` точный список (не `"*"`), нет `expand="changelog"`

### Шаг 3. Установить и запустить

```bash
cp -r skill/jira-enricher ~/.gigacode/skills/
gigacode
```

В чате: "запусти jira-enricher".

## Сколько времени занимает

- 28 задач × `jira_get_issue` + паузы = ~30 сек (батчи по 5)
- + до 15 эпиков × `jira_get_issue` для имён = ~10 сек
- + ~10-15 эпиков × `jira_search` для дочерних = ~15 сек

Итого ~1 минута.

## Проверка результата

```bash
cat pipeline/step-2-after-jira-enricher.md | head -50
```

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
print('Skills:', d['metadata']['skills_completed'])
print('Tasks:', len(d['tasks']))
print('Tasks with jira:', sum(1 for t in d['tasks'] if (t.get('jira') or {}).get('found')))
print('Epics:', len(d.get('epics', [])))  # ГЛАВНОЕ — должно быть > 0
for e in d.get('epics', [])[:5]:
    cnt = e.get('children_count_total')
    print(f'  {e[\"key\"]}: from_plan={len(e[\"tasks_from_plan\"])} children={cnt}')
"
```

Должно вывести:
```
Skills: ['excel-parser', 'jira-enricher']
Tasks: 28
Tasks with jira: ~26-28
Epics: ~10-15        ← ЭТО ГЛАВНОЕ. Если 0 — баг.
  ASFC-57216: from_plan=2 children=38
  ASFC-65543: from_plan=6 children=15
  ...
```

## Если массив эпиков пустой

**Корневой баг v3.1.** Если после прогона `Epics: 0`, значит шаг `aggregate-epics` пропущен в SKILL.md.

**Быстрое решение** (без перегенерации скилла):
```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py aggregate-epics
python3 ~/.gigacode/skills/jira-enricher/helper.py write-step2
```

Это **идемпотентно** — можно запускать сколько угодно раз. Дальше для каждого эпика нужно сделать `jira_search` руками (или попросить агента) + `update-epic-children`.

**Правильное решение:** перегенерировать SKILL.md по `PROMPT.md` v3.2 с явным упором на разделы 10 и 15 SPEC.md.

## Если что-то ещё пошло не так

| Симптом | Действие |
|---------|----------|
| `NameError: mcp__Atlassian__jira_get_issue` | Скилл сгенерил вызов MCP в Python — перегенерировать |
| Колонка "Поток" пустая или странная | Проверить `extract_team` — должна обрабатывать массив строк |
| Колонка "Эпик" пустая для ASFC | Проверить что `customfield_11400` в `fields=` |
| Колонка "Эпик" пустая для CRSIGMA | Проверить fallback на `issuelinks "Implement in"` с `outward_issue` (подчёркивание!) |
| `Epics: 0` в проверке | Пропущен шаг `aggregate-epics` — см. секцию выше |
| `update-epic-children` падает с "epic not in enriched.epics" | Пропущен шаг `aggregate-epics` ДО update-epic-children |
| В таблице step-2 колонка "Команда" | Не обновлён `write_step2_markdown` — перегенерировать |
| Контекст переполнился | `fields="*"` вместо точного списка. Перегенерировать. |

## Следующий шаг

- Запустить `timing-analyzer` — добавит факт А/Р/Т из changelog для активных задач
- Или сразу `report-builder` если факт не нужен
