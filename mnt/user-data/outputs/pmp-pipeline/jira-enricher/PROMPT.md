# PROMPT для GigaCode CLI — jira-enricher (v3.1)

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. jira-enricher — второй из 4 скиллов pipeline pmp-vs-jira.

Сгенерируй РОВНО два файла:

1. skill/jira-enricher/SKILL.md
2. skill/jira-enricher/helper.py

Что делает скилл:
- Читает pipeline/enriched.json (валидирует что excel-parser отработал)
- Для каждой задачи делает jira_get_issue БЕЗ changelog с фиксированным fields=
- Извлекает: статус (+ category/phase через mapping), эпик (customfield_11400 или issuelinks "Implement in"), команду (customfield_22200 как МАССИВ СТРОК), lead_time
- Делает jira_search для подсчёта дочерних каждого уникального эпика
- Перезаписывает pipeline/enriched.json
- Создаёт pipeline/step-2-after-jira-enricher.md

КРИТИЧЕСКАЯ АРХИТЕКТУРА (читать ВНИМАТЕЛЬНО — это причина провала прошлой версии):

Скилл работает как ДИАЛОГ между двумя сущностями:

АГЕНТ (в чате GigaCode CLI):
- Делает НАТИВНЫЕ tool calls jira_get_issue и jira_search
- Получает JSON-ответы в свой контекст
- Извлекает поля (status.name, customfield_22200[0], и т.д.) глазами
- Накапливает результаты как текстовый JSON-батч
- После 5 задач — передаёт батч в helper.py через bash:

  echo '<JSON-batch>' | python3 ~/.gigacode/skills/jira-enricher/helper.py merge-batch

PYTHON через bash (helper.py):
- НЕ делает MCP-вызовов (это невозможно!)
- Принимает JSON через stdin
- Парсит, мерджит в pipeline/enriched.json

ЗАПРЕЩЕНО:
- mcp__Atlassian__jira_get_issue(...) — NameError, такой функции нет в Python
- from mcp_atlassian import jira_get_issue — модуля не существует
- def fetch_issue(...) с псевдокодом MCP-вызова внутри
- Передавать большие batch через CLI-аргумент (длина ограничена) — только через stdin

Это причина провала прошлого прогона: GigaCode попытался засунуть tool call в Python-скрипт, получил NameError, скилл "зависал" между нативным контекстом агента и Python-окружением.

Критически важно:

1. РАЗДЕЛ 7 SPEC.md СОДЕРЖИТ ГОТОВЫЙ КОД helper.py. Использовать его как основу. Все функции (map_status, extract_epic, extract_team, compute_lead_time, extract_jira_fields, merge_batch, merge_epics, finalize, write_step2_markdown) — уже прописаны.

2. Раздел 5 SPEC.md — формула вызова MCP: ТОЧНЫЙ список fields, БЕЗ expand=changelog. Не использовать fields="*".

3. customfield_22200 (Team) — это МАССИВ СТРОК типа ["PALM.CSP.K7M"]. Не объект. См. extract_team в разделе 7.

4. customfield_11400 (Epic Link) — для ASFC задач строка типа "ASFC-65543", для CRSIGMA часто null (тогда эпик через issuelinks).

5. issuelinks тип "Implement in" — fallback для эпика когда customfield_11400 пуст.

6. Mapping статусов на phase/category — берётся из раздела 6 SPEC.md и CONTRACT.md.

7. Раздел 13 SPEC.md (антипаттерны): ТОЛЬКО helper.py. Никаких main.py, process.py, run_*.py, generate_*.py, __pycache__.

8. Batching по 5 задач — это для того чтобы при сбое не потерять прогресс и не переполнить контекст.

9. Создаётся ДВА файла на выходе:
   - pipeline/enriched.json (обновлён)
   - pipeline/step-2-after-jira-enricher.md (читаемый снимок)

10. Папка pipeline/ (без точки — видимая).

Когда сгенерируешь — выведи в чат:
- Файлы созданы (ровно 2)
- Список функций в helper.py
- Один пример Step из SKILL.md где описан вызов jira_get_issue (показать что нет mcp__... обёртки в Python)
- Подтверди что MCP вызывается НАТИВНО агентом, не из Python
- Подтверди что batching по 5, передача через stdin (echo ... | python3 ...)
- Подтверди что fields= точный список, не "*"
- Подтверди что нигде нет expand="changelog"
- Подтверди что customfield_22200 обрабатывается как массив строк
```

## Ожидаемый результат

После запуска `jira-enricher` на реальном файле плана:
- 28 задач обработаны (некоторые могут быть not_found)
- Найдено уникальных эпиков: ~10-12
- Заполнен `task.jira` у каждой задачи
- Заполнен массив `epics` со счётчиком дочерних
- Созданы файлы:
  - `pipeline/enriched.json` (обновлён, добавлена секция `jira` и `epics`)
  - `pipeline/step-2-after-jira-enricher.md` (читаемый снимок)
- В чате сводка: количество найдено/не найдено, эпиков, источников команды
