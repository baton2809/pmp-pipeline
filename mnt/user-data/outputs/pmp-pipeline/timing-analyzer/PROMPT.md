# PROMPT для GigaCode CLI — timing-analyzer (v3.1)

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. timing-analyzer — третий из 4 скиллов pipeline pmp-vs-jira.

Сгенерируй РОВНО два файла:

1. skill/timing-analyzer/SKILL.md
2. skill/timing-analyzer/helper.py

Что делает скилл:
- Читает pipeline/enriched.json (валидирует что jira-enricher отработал)
- Фильтрует задачи: только active (status_category in analysis/development/testing)
- Для каждой active задачи делает jira_get_issue С expand="changelog" и минимальным fields=
- Парсит историю переходов статусов из changelog
- Строит timeline статусов с timestamps (точный алгоритм в разделе 5 SPEC.md)
- Группирует интервалы по фазам через mapping
- Считает phase_days = {A, R, T, not_started, finished, unknown} в КАЛЕНДАРНЫХ ДНЯХ
- Записывает task.timing для каждой обработанной задачи
- Для неактивных — тривиальный timing с нулями
- Перезаписывает pipeline/enriched.json
- Создаёт pipeline/step-3-after-timing-analyzer.md

КРИТИЧЕСКАЯ АРХИТЕКТУРА (та же что в jira-enricher):

АГЕНТ в чате:
- Делает НАТИВНЫЙ tool call jira_get_issue с expand="changelog"
- Получает JSON-ответ
- Передаёт ответ в helper.py через bash + stdin:

  echo '{"cr_key": "...", "response": <полный JSON от MCP>}' | python3 helper.py compute-from-response

PYTHON через bash (helper.py):
- НЕ делает MCP-вызовов
- Принимает JSON через stdin
- compute_timing() применяет алгоритм построения timeline
- Записывает результат в pipeline/enriched.json

ЗАПРЕЩЕНО:
- mcp__Atlassian__jira_get_issue(...) в Python — NameError
- def fetch_issue(...) с псевдокодом
- Запрашивать changelog для НЕ активных задач — экономим контекст

Критически важно:

1. РАЗДЕЛ 5 SPEC.md СОДЕРЖИТ ГОТОВЫЙ КОД helper.py. Использовать его как основу. Все функции (map_status, parse_iso, extract_status_transitions, build_timeline, aggregate_phase_days, compute_timing, is_active, list_active, fill_inactive, compute_from_response, finalize, write_step3_markdown) — уже прописаны и протестированы.

2. Раздел 4 SPEC.md — формула вызова MCP. expand="changelog" ОБЯЗАТЕЛЕН. fields = минимальный список (summary, status, created, updated, resolutiondate).

3. Алгоритм построения timeline (раздел 5 SPEC.md в коде build_timeline):
   - Сортировать transitions по timestamp
   - Первый интервал: от created задачи до первого перехода (статус = from_string первого перехода или to_string если from_string пуст)
   - Между переходами: использовать to_string предыдущего как имя статуса
   - Последний интервал: до resolutiondate (если задача finished) или до now

4. Возвраты задачи в один статус — алгоритм СУММИРУЕТ интервалы (это правильно).

5. Учитывать часовые пояса в timestamps (формат "2026-02-13T18:24:19.841+0300"). parse_iso уже умеет.

6. НЕ обрабатывать неактивные задачи через MCP. Им — тривиальный timing через fill_inactive.

7. НЕ хранить сырой changelog в enriched.json — только агрегированные phase_days.

8. Раздел 11 SPEC.md (антипаттерны): ТОЛЬКО helper.py. Никаких main.py, process.py, run_*.py.

9. Создаётся ДВА файла на выходе:
   - pipeline/enriched.json (обновлён, добавлено task.timing)
   - pipeline/step-3-after-timing-analyzer.md (читаемый снимок)

10. Папка pipeline/ (без точки — видимая).

Когда сгенерируешь — выведи в чат:
- Файлы созданы (ровно 2)
- Функции в helper.py
- Один пример шага вызова jira_get_issue из SKILL.md (показать что нет mcp__... в Python)
- Алгоритм построения timeline (короткое описание из build_timeline)
- Подтверди что НЕТ запроса changelog для неактивных задач (используется list_active + fill_inactive)
- Подтверди что данные передаются через stdin (echo ... | python3 helper.py compute-from-response)
- Подтверди что parse_iso учитывает часовые пояса
```

## Ожидаемый результат

После запуска `timing-analyzer`:
- Активных задач (для которых считаем): 8-12 из 28
- У каждой active заполнен `timing.computed = true` с реальным `phase_days`
- У неактивных `timing.computed = false`, нули
- Топ-3 самых долгих фаз: видно в step-3.md
- Созданы файлы:
  - `pipeline/enriched.json` (обновлён, добавлено task.timing)
  - `pipeline/step-3-after-timing-analyzer.md`

## Валидация результата

После запуска полезно проверить одну задачу:

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
for t in d['tasks']:
    if t['cr_key'] == 'CRSIGMA-23749':  # известная задача
        print('jira status:', t['jira']['status'])
        print('timing:', t['timing'])
        break
"
```
