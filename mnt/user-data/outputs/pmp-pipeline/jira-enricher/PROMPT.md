# PROMPT для GigaCode CLI — jira-enricher

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. jira-enricher — второй из 4 скиллов pipeline pmp-vs-jira.

Сгенерируй два файла:

1. skill/jira-enricher/SKILL.md
2. skill/jira-enricher/helper.py

Что делает скилл:
- Читает .cache/enriched.json (валидирует что excel-parser отработал)
- Для каждой задачи делает jira_get_issue БЕЗ changelog с фиксированным fields=
- Извлекает: статус (+ category/phase через mapping), эпик (customfield_11400 или issuelinks "Implement in"), команду (customfield_22200 или assignee), lead_time
- Делает jira_search для подсчёта дочерних каждого уникального эпика
- Перезаписывает .cache/enriched.json с дополнениями
- НЕ запрашивает changelog, это работа следующего скилла timing-analyzer

Критически важно:

1. Раздел 5 SPEC.md — формула вызова jira_get_issue: ТОЧНЫЙ список fields, БЕЗ expand=changelog. Не использовать fields="*".

2. Раздел 9 SPEC.md (формат MCP-вызова): SKILL.md инструкция на естественном языке, не Python-обёртки. Никаких result = mcp_jira_get_issue, никаких def fetch_issue с псевдокодом.

3. Раздел 6 SPEC.md (mapping статусов): использовать единый mapping из CONTRACT.md.

4. Раздел 7 SPEC.md (извлечение полей): epic через customfield_11400 или issuelinks "Implement in", team через customfield_22200 или assignee. Псевдокод в SPEC.md — реализовать в helper.py.

5. Раздел 10 SPEC.md (файлы): ТОЛЬКО helper.py. Никаких main.py, process.py, run_*.py, generate_*.py, __pycache__.

6. CONTRACT.md секция "После jira-enricher" — строгая структура json.

7. Валидация на входе: проверить что "excel-parser" в metadata.skills_completed. Без этого не работаем.

Когда сгенерируешь — выведи в чат:
- Файлы которые создал
- Список функций в helper.py
- Один пример Step из SKILL.md где описан вызов jira_get_issue (чтобы я мог проверить что нет заглушек)
- Проверь что fields= содержит точный список, не "*"
- Проверь что нигде нет expand="changelog"
```
