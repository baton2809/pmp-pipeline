# PROMPT для GigaCode CLI — jira-enricher (v3.2)

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. jira-enricher — второй из 4 скиллов pipeline pmp-vs-jira.

В прошлой попытке (v3.1) скилл провалился частично: 28/28 задач обрабатывались корректно, но массив enriched.epics[] оставался ПУСТЫМ. Из-за этого секция "Срез по эпикам" в финальном отчёте показывала "Эпики не найдены."

Причина — после успешного цикла merge_batch агент ПРОПУСКАЛ шаг агрегации эпиков. Функция merge_epics существовала, но никто её не вызывал с данными.

В v3.2 решение: добавлены ЯВНЫЕ шаги aggregate-epics + update-epic-children. Без них pipeline не доходит до финального отчёта корректно.

Сгенерируй РОВНО два файла:

1. skill/jira-enricher/SKILL.md
2. skill/jira-enricher/helper.py

Что делает скилл:
- Читает pipeline/enriched.json (валидирует что excel-parser отработал)
- Цикл по задачам батчами по 5:
  * jira_get_issue (БЕЗ changelog) для каждой
  * После каждых 5 — echo batch | python3 helper.py merge-batch
- ВАЖНО: после ВСЕХ задач — python3 helper.py aggregate-epics
  * Это собирает уникальные эпики из task.jira.epic в enriched.epics[]
  * БЕЗ ЭТОГО ШАГА массив будет ПУСТОЙ
- (опц.) Догрузка имён эпиков через list-epics-without-names + jira_get_issue
- Для каждого эпика — jira_search для подсчёта дочерних
  * После каждого — python3 helper.py update-epic-children <key> <count>
- finalize + write-step2

КРИТИЧЕСКАЯ АРХИТЕКТУРА (без изменений с v3.1):

АГЕНТ в чате:
- Делает НАТИВНЫЕ tool calls jira_get_issue и jira_search
- Получает JSON в свой контекст
- Извлекает поля глазами
- Передаёт batch в helper через bash + stdin

PYTHON через bash (helper.py):
- НЕ делает MCP-вызовов (NameError)
- Принимает данные через stdin или CLI-аргументы
- Парсит, мерджит, пишет в pipeline/enriched.json

Критически важно:

1. РАЗДЕЛ 9 SPEC.md СОДЕРЖИТ ГОТОВЫЙ КОД helper.py. Использовать его как основу. Все функции уже прописаны и протестированы:
   * merge_batch — через stdin
   * aggregate_epics — НОВАЯ функция, без аргументов
   * update_epic_children KEY COUNT — через CLI args
   * list_epics_to_count — выводит JSON-массив эпиков без counter
   * list_epics_without_names — выводит JSON эпиков без имени (≤15)
   * update_epic_name KEY NAME — через CLI args
   * finalize
   * write_step2_markdown — с колонкой "Поток", не "Команда"

2. ОБЯЗАТЕЛЬНАЯ ПОСЛЕДОВАТЕЛЬНОСТЬ ШАГОВ В SKILL.md (раздел 10 SPEC.md):
   Step 2 — цикл batched по 5 задач (merge-batch)
   Step 3 — aggregate-epics (ОДИН РАЗ после всех батчей)
   Step 4 — (опц.) догрузка имён эпиков
   Step 5 — цикл по эпикам: jira_search + update-epic-children
   Step 6 — finalize
   Step 7 — write-step2

   Шаг 3 КРИТИЧЕСКИЙ — без него секция эпиков останется пустой.

3. customfield_22200 (Поток разработки) — это МАССИВ СТРОК ["PALM.CSP.K7M"]. В step-2-after-jira-enricher.md колонка называется "Поток" (НЕ "Команда") — это техническая метка потока, не имя команды.

4. customfield_11400 (Epic Link) — для ASFC-задач строка типа "ASFC-65543". Для CRSIGMA — null, тогда эпик через issuelinks тип "Implement in" → outward_issue.key (С ПОДЧЁРКИВАНИЕМ, не outwardIssue!).

5. fields= в jira_get_issue — точный список из раздела 6 SPEC.md, НЕ "*".

6. НЕТ expand="changelog" — это работа timing-analyzer.

7. Раздел 15 SPEC.md (антипаттерны):
   * ГЛАВНЫЙ: не пропустить aggregate-epics. Это корневой баг v3.1.
   * НЕ создавать main.py, process.py, run_*.py — только helper.py
   * НЕ передавать batch через CLI-аргумент — только stdin
   * НЕ обращаться к MCP из Python

8. Раздел 16 SPEC.md (критерий успеха):
   * Главная проверка: len(enriched['epics']) > 0
   * Каждый эпик имеет tasks_from_plan и children_count_total

Когда сгенерируешь — выведи в чат:
- Файлы созданы (ровно 2)
- Список функций в helper.py
- Один пример Step из SKILL.md где описан aggregate-epics
- Подтверди что есть ЯВНЫЙ шаг aggregate-epics после цикла merge-batch
- Подтверди что есть цикл по эпикам с update-epic-children после каждого jira_search
- Подтверди что в write_step2_markdown колонка "Поток" (не "Команда")
- Подтверди что extract_epic использует outward_issue с подчёркиванием
- Подтверди что extract_team обрабатывает customfield_22200 как массив строк
- Подтверди что нет main.py / process.py / run_*.py
```

## Ожидаемый результат

После запуска `jira-enricher`:
- 28 задач обработаны
- **Массив `epics` НЕ пустой** (главное отличие от v3.1)
- У каждого эпика: `key`, `name`, `tasks_from_plan`, `children_count_total`
- В чате сводка: количество найдено/не найдено, эпиков, потоков

## Валидация результата

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

**Главное число: `Epics: > 0`.** Если 0 — значит шаг `aggregate-epics` пропущен.

## Если массив эпиков пустой после прогона

Можно запустить вручную (это идемпотентно):

```bash
python3 ~/.gigacode/skills/jira-enricher/helper.py aggregate-epics
python3 ~/.gigacode/skills/jira-enricher/helper.py write-step2
```

Дальше — `jira_search` руками для каждого эпика, потом `update-epic-children`. Но лучше перегенерировать SKILL.md.
