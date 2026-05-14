# PROMPT для GigaCode CLI — timing-analyzer (v3.2)

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. timing-analyzer — третий из 4 скиллов pipeline pmp-vs-jira.

В прошлой попытке (v3.1) скилл провалился: все 24 активные задачи получили computed=false. Главные причины:
- Скилл читал ключ 'changelog' (единственное число), а в Сбер-MCP он называется 'changelogs' (множественное)
- Скилл читал 'from_string'/'to_string' (snake_case), а реально приходит 'fromString'/'toString' (camelCase) для field='status'
- Скилл пытался передавать большой changelog через stdin/echo — кавычки и длина команды ломались
- Контекст агента переполнялся при 24 задачах подряд

Сгенерируй РОВНО два файла:

1. skill/timing-analyzer/SKILL.md
2. skill/timing-analyzer/helper.py

Что делает скилл:
- Читает pipeline/enriched.json (валидирует что jira-enricher отработал)
- Получает список active задач через helper.py list-active
- Для НЕАКТИВНЫХ — fill-inactive (тривиальный timing)
- Для активных — STREAMING-цикл (по ОДНОЙ задаче):
  1. Tool call jira_get_issue с expand="changelog"
  2. Сразу WriteFile JSON-ответа в pipeline/tmp/<cr_key>.json
  3. Shell: python3 helper.py compute-from-file pipeline/tmp/<cr_key>.json
  4. helper парсит файл, считает, мерджит в enriched.json
- НЕ копит ответы в контексте
- Создаёт pipeline/step-3-after-timing-analyzer.md

КРИТИЧЕСКАЯ АРХИТЕКТУРА (это причина провала v3.1):

1. Запись JSON ТОЛЬКО через встроенный WriteFile tool агента
   - НЕ через echo '...' > file (кавычки сломают большой JSON)
   - НЕ через python3 -c (то же)
   - НЕ в ~/.gigacode/tmp/ (Filesystem Guard блокирует)
   - ТОЛЬКО WriteFile + рабочая директория проекта (pipeline/tmp/)

2. STREAMING — обработка по одной задаче, не батчами
   Цикл строго: tool call → WriteFile → Shell-helper → следующая задача
   После каждого compute-from-file JSON выпадает из контекста (он на диске)

3. MCP вызывается ТОЛЬКО как нативный tool call агента в чате
   В helper.py НИКОГДА не должно быть mcp__Atlassian__... или from mcp_atlassian
   Это NameError, helper не имеет доступа к MCP

СТРУКТУРА changelog (зафиксировано на реальных ответах от Сбер-MCP):

{
  'changelogs': [                             # ← МНОЖЕСТВЕННОЕ число
    {
      'created': '2025-08-14T13:32:23.287+0300',
      'items': [
        {
          'field': 'status',
          'fromString': 'New',                # ← camelCase для status
          'toString': 'In Progress',
        },
        {
          'field': 'Link',
          'to_string': '...',                 # ← snake_case для других полей
        },
      ]
    }
  ]
}

extract_status_transitions ДОЛЖНА:
- Читать поле 'changelogs' (множественное число)
- Перебирать массив напрямую (НЕТ обёртки 'histories'!)
- Фильтровать ТОЛЬКО элементы где item['field'] == 'status'
- Для них читать 'fromString' и 'toString' (camelCase)

Критически важно:

1. РАЗДЕЛ 7 SPEC.md СОДЕРЖИТ ГОТОВЫЙ КОД helper.py — целиком. Использовать его как основу. Все функции (map_status, parse_iso, extract_status_transitions, build_timeline, aggregate_phase_days, compute_timing, is_active, list_active, fill_inactive, compute_from_file, cleanup_tmp, finalize, write_step3_markdown) — уже прописаны и протестированы.

2. Раздел 4 SPEC.md — формула MCP-вызова: expand="changelog" + минимальный fields.

3. Раздел 5 SPEC.md — точная структура ответа MCP. Соблюдать дословно.

4. Раздел 6 SPEC.md — архитектура передачи. WriteFile tool, не bash echo.

5. Раздел 8 SPEC.md — последовательность Steps. Шаги 4.1 → 4.2 → 4.3 для каждой задачи по очереди (streaming).

6. Раздел 13 SPEC.md (антипаттерны):
   - НЕ создавать main.py, process_timing.py, run_timing.sh, batch_timing.json или любые другие .py/.sh/.json кроме разрешённых
   - НЕ копить tool-ответы в контексте
   - НЕ читать ключ 'changelog' (только 'changelogs')
   - НЕ использовать from_string/to_string для status (только fromString/toString)
   - НЕ писать в ~/.gigacode/tmp/

7. Создаётся ДВА файла на выходе:
   - pipeline/enriched.json (обновлён, добавлено task.timing)
   - pipeline/step-3-after-timing-analyzer.md

Временные файлы (pipeline/tmp/<cr_key>.json) создаются по одному во время работы и удаляются в Step 7 (cleanup-tmp).

Когда сгенерируешь — выведи в чат:
- Файлы созданы (ровно 2: SKILL.md и helper.py)
- Функции в helper.py
- Один пример шага Step 4.1-4.3 из SKILL.md (tool call → WriteFile → Shell)
- Подтверди что используется ключ 'changelogs' (множественное)
- Подтверди что для status извлекаются 'fromString'/'toString' (camelCase)
- Подтверди что non-status элементы (Link, description) ОТФИЛЬТРОВАНЫ
- Подтверди что используется WriteFile tool, не bash echo
- Подтверди что цикл streaming (по одной задаче), не batch (по 5)
- Подтверди что НЕТ создания process_timing.py / run_timing.sh / batch_timing.json
```

## Ожидаемый результат

После запуска `timing-analyzer`:
- Активных задач (для которых считаем): 8-14 из 28
- У каждой active заполнен `timing.computed = true` с реальным `phase_days`
- У неактивных `timing.computed = false`, нули
- Топ-3 самых долгих фаз: видно в step-3.md
- Созданы файлы:
  - `pipeline/enriched.json` (обновлён)
  - `pipeline/step-3-after-timing-analyzer.md`
- `pipeline/tmp/` пуст (или удалён) после cleanup

## Валидация результата

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
computed = [t for t in d['tasks'] if (t.get('timing') or {}).get('computed')]
print('Active with timing:', len(computed))
for t in computed[:3]:
    p = t['timing']['phase_days']
    print(f\"  {t['cr_key']}: A={p['A']} R={p['R']} T={p['T']}\")
"
```

Должно быть `Active with timing: > 0` (если в плане есть активные задачи).
