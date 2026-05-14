# timing-analyzer v3.2 — третий скилл pipeline

Считает **факт А/Р/Т в календарных днях** для активных задач через анализ истории переходов статусов (changelog).

## Изменения v3.2 vs v3.1 (после провала первого прогона)

**Что пошло не так в v3.1:**
- Все 24 активные задачи получили `computed=false`
- Скилл читал ключ `changelog` (единственное число), а в Сбер-MCP он называется `changelogs` (множественное)
- Скилл читал `from_string`/`to_string` (snake_case), а реально приходит `fromString`/`toString` (camelCase) для `field='status'`
- Передача changelog через stdin/echo не работала — кавычки и длина команды
- Контекст агента переполнялся при 24 задачах подряд

**Что починено в v3.2:**
- ✅ Правильное чтение `changelogs` (множественное), без обёртки `histories`
- ✅ Для `field == 'status'` читаем `fromString`/`toString` (camelCase)
- ✅ Фильтр — только `field == 'status'`, игнорируем Link/description и другие
- ✅ Архитектура передачи: **WriteFile tool агента** записывает JSON в файл → `python3 helper.py compute-from-file <path>`
- ✅ Streaming: обработка по **одной** задаче с записью в файл — контекст не переполняется
- ✅ Запрет `~/.gigacode/tmp/` (Filesystem Guard) — только `pipeline/tmp/` в рабочей директории
- ✅ Готовый код `helper.py` валидирован на mock changelog с правильной структурой
- ✅ CLI с подкомандами: `list-active`, `fill-inactive`, `compute-from-file`, `cleanup-tmp`, `finalize`, `write-step3`

## Место в pipeline

```
1. excel-parser
2. jira-enricher
3. timing-analyzer    ← вы здесь
4. report-builder
```

## Главное про этот скилл

Это **единственный скилл который реально считает факт работы** над задачей. Работает только для **активных** задач (статус не Done, не Backlog) — для них факт интересен.

Что значит "факт А/Р/Т в календарных днях":
- Это **не чел-дни.** Это календарное время в статусах соответствующей фазы.
- Включает выходные, ожидания, переключения разработчика.
- Сравнение факт vs план — это **сигнал** "стоит проверить", не точная метрика.

Эти оговорки попадут в дисклеймер `report.md`.

## Подготовка

В `~/.gigacode/settings.json`:

```json
"includeTools": ["jira_get_issue"]
```

(`jira_search` уже не нужен — это была работа `jira-enricher`.)

Должны отработать `excel-parser` и `jira-enricher`.

## Запуск

### Шаг 1. Сгенерировать

```bash
cd timing-analyzer
gigacode
```

Вставить промпт из `PROMPT.md`.

### Шаг 2. Проверить перед установкой

В `skill/timing-analyzer/` — ровно 2 файла:

**Главное (после v3.1 это был корень провала):**
- В helper.py `compute_timing` читает поле **`changelogs`** (множественное), НЕ `changelog`
- В helper.py `extract_status_transitions` фильтрует **только** `item['field'] == 'status'`
- Для status-переходов читает **`fromString`** и **`toString`** (camelCase, не snake_case)
- Нет обёртки `histories` — массив changelogs обрабатывается напрямую

**Архитектура передачи:**
- В SKILL.md tool call `jira_get_issue` **С** параметром `expand="changelog"`
- В SKILL.md **WriteFile tool** записывает ответ в `pipeline/tmp/<cr_key>.json` (НЕ через `echo`, НЕ через bash, НЕ в `.gigacode/`)
- В SKILL.md после каждой задачи — `python3 helper.py compute-from-file pipeline/tmp/<cr_key>.json`
- Streaming: цикл по ОДНОЙ задаче (`tool call → WriteFile → Shell`), не batch
- В SKILL.md фильтр `is_active` применяется ДО tool call — получаем список через `helper.py list-active`

**Антипаттерны:**
- В helper.py **НЕТ** строк типа `mcp__Atlassian__jira_get_issue` — это NameError
- НЕТ создания `process_timing.py`, `run_timing.sh`, `batch_timing.json` — только `helper.py`
- НЕТ запроса changelog для неактивных задач (для них `fill-inactive`)
- `parse_iso` учитывает часовые пояса (формат `+0300` конвертирует в `+03:00`)

### Шаг 3. Установить и запустить

```bash
cp -r skill/timing-analyzer ~/.gigacode/skills/
gigacode
```

В чате: "запусти timing-analyzer".

## Сколько времени занимает

- Из 28 задач Натальи активных обычно 8-12
- Каждая = 1 jira_get_issue с changelog + пауза 0.2 сек = ~5-10 сек
- Расчёт через helper мгновенный

Итого 15-30 секунд.

## Проверка результата

```bash
cat pipeline/step-3-after-timing-analyzer.md | head -20
```

```bash
python3 -c "
import json
d = json.load(open('pipeline/enriched.json'))
active = [t for t in d['tasks'] if (t.get('timing') or {}).get('computed')]
print('Активных с timing:', len(active))
print()
for t in active[:5]:
    p = t['timing']['phase_days']
    print(f\"{t['cr_key']:20s} A={p['A']:6.1f}  R={p['R']:6.1f}  T={p['T']:6.1f}\")
"
```

Должно показать что-то вроде:
```
Активных с timing: 9

CRSIGMA-23749         A=  30.5  R= 240.2  T=   0.0
ASFC-35817            A=  98.0  R= 700.5  T=   0.0
...
```

## Если что-то пошло не так

| Симптом | Действие |
|---------|----------|
| `NameError: mcp__Atlassian__jira_get_issue` | Скилл вызывает MCP из Python — перегенерировать |
| Все timing = 0 / null | Алгоритм не работает, проверить `build_timeline` в helper.py |
| Запрашивает changelog для всех 28 задач (долго) | Не использует `is_active` фильтр — перегенерировать |
| Phase_days содержат отрицательные числа | Сортировка transitions не работает — проверить `extract_status_transitions` |
| Часовые пояса игнорируются | `parse_iso` не учитывает offset — проверить регулярку |

## Следующий шаг

Запустить `report-builder` — построит итоговый markdown отчёт со всеми колонками.
