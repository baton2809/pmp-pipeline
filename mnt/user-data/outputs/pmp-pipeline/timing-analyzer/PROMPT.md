# PROMPT для GigaCode CLI — timing-analyzer

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. timing-analyzer — третий из 4 скиллов pipeline pmp-vs-jira.

Сгенерируй два файла:

1. skill/timing-analyzer/SKILL.md
2. skill/timing-analyzer/helper.py

Что делает скилл:
- Читает .cache/enriched.json (валидирует что jira-enricher отработал)
- Фильтрует задачи: только active (status_category in analysis/development/testing)
- Для каждой active задачи делает jira_get_issue с expand="changelog" и минимальным fields=
- Парсит историю переходов статусов из changelog
- Строит timeline статусов с timestamps (см. раздел 7.2 SPEC.md — точный алгоритм)
- Группирует интервалы по фазам через mapping (см. CONTRACT.md)
- Считает phase_days = {A, R, T, not_started, finished, unknown} в календарных днях
- Записывает task.timing для каждой обработанной задачи
- Для неактивных — тривиальный timing с нулями
- Перезаписывает .cache/enriched.json

Критически важно:

1. Раздел 7 SPEC.md содержит ТОЧНЫЙ алгоритм расчёта phase_days в трёх шагах с псевдокодом. Реализовать в helper.py функции, не примерно.

2. Раздел 5 SPEC.md — формула MCP-вызова. expand="changelog" ОБЯЗАТЕЛЕН. fields = минимальный список из 5 полей.

3. Раздел 6 SPEC.md — фильтрация active задач. Неактивные через MCP НЕ обрабатываем (экономим контекст). Им пишем тривиальный timing с computed=false.

4. Раздел 9 SPEC.md (формат MCP): SKILL.md — инструкция для агента на естественном языке. Никаких result = mcp_jira_get_issue, никаких заглушек.

5. Раздел 10 SPEC.md (файлы): ТОЛЬКО helper.py. Никаких main.py, process.py, run_*.py, generate_*.py, __pycache__.

6. НЕ хранить сырой changelog в enriched.json — только агрегированные phase_days.

7. НЕ обрабатывать неактивные задачи через MCP — это лишние ~20 вызовов на ~28 задачах.

8. Учитывать часовые пояса при парсинге ISO timestamps (формат "2026-02-13T18:24:19.841+0300").

9. Возвраты задачи в один и тот же статус — алгоритм естественно суммирует интервалы (это правильно).

Когда сгенерируешь — выведи в чат:
- Файлы которые создал
- Функции в helper.py (должно быть: is_active, parse_iso, extract_status_transitions, build_timeline, aggregate_phase_days, status_to_phase, compute_timing)
- Пример шага вызова jira_get_issue из SKILL.md (проверка что нет заглушек)
- Один пример как helper строит timeline (псевдокод или реальная функция)
- Подтверди что НЕТ запроса changelog для неактивных задач
```
