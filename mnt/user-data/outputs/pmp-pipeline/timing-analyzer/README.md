# timing-analyzer v3.1 — третий скилл pipeline

Считает **факт А/Р/Т в календарных днях** для активных задач через анализ истории переходов статусов (changelog).

## Изменения v3.1 vs v3.0

- ✅ **Архитектурный фикс:** агент передаёт сырой ответ MCP через stdin в helper, helper применяет алгоритм `compute_timing`. Никаких MCP-вызовов из Python.
- ✅ Папка `pipeline/` (видимая)
- ✅ Создаётся `pipeline/step-3-after-timing-analyzer.md` с топ-10 самых долгих фаз
- ✅ Готовый код `helper.py` в SPEC.md раздел 5 (включая алгоритм `build_timeline` и `aggregate_phase_days`)
- ✅ CLI с подкомандами: `list-active`, `fill-inactive`, `compute-from-response`, `merge-batch`, `finalize`, `write-step3`
- ✅ Алгоритм построения timeline валидирован на mock changelog: для задачи в Backlog 3.9 д → In Progress 2.3 д → Ready for QA 12 д даёт правильные phase_days

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

- В SKILL.md tool call jira_get_issue **С** параметром `expand="changelog"`
- В SKILL.md фильтр `is_active` применяется ДО tool call — изначально получаем список активных задач через `helper.py list-active`
- В helper.py есть функции `extract_status_transitions`, `build_timeline`, `aggregate_phase_days`, `compute_timing`
- Алгоритм построения timeline в `build_timeline` соответствует SPEC.md раздел 5
- В SKILL.md передача через stdin: `echo '...' | python3 helper.py compute-from-response`
- В helper.py **НЕТ** строк типа `mcp__Atlassian__jira_get_issue` — это NameError
- `parse_iso` учитывает часовые пояса (формат `+0300` конвертирует в `+03:00`)
- НЕТ запроса changelog для неактивных задач (для них вызывается `fill-inactive`)

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
