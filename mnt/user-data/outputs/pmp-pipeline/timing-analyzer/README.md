# timing-analyzer — третий скилл pipeline

Считает **факт А/Р/Т в календарных днях** для активных задач через анализ истории переходов статусов (changelog).

## Место в pipeline

```
1. excel-parser       → .cache/enriched.json (план)
2. jira-enricher      → .cache/enriched.json (+статус, +эпик, +команда)
3. timing-analyzer    ← вы здесь (+факт А/Р/Т)
4. report-builder
```

## Что нового в этом скилле

В отличие от `jira-enricher`, этот скилл запрашивает у Jira **changelog** (историю изменений). Из него парсит **переходы статусов** с timestamps, строит timeline, агрегирует календарные дни по фазам А/Р/Т.

**Главное:** обрабатывает **только активные** задачи (статус не Done, не Backlog) — для них факт интересен. Не активные получают тривиальный пустой timing.

## Подготовка

В `~/.gigacode/settings.json`:

```json
"includeTools": ["jira_get_issue"]
```

`jira_search` уже не нужен — это была работа предыдущего скилла.

Должны отработать `excel-parser` и `jira-enricher`. Файл `.cache/enriched.json` должен содержать `["excel-parser", "jira-enricher"]` в `metadata.skills_completed`.

## Запуск

### Шаг 1. Сгенерировать

```bash
cd timing-analyzer
gigacode
```

Вставить промпт из `PROMPT.md`. Получить `skill/timing-analyzer/SKILL.md` + `helper.py`.

Перед установкой проверить:
- 2 файла в `skill/timing-analyzer/`
- В SKILL.md вызов jira_get_issue **С** параметром `expand="changelog"`
- В SKILL.md нет запроса changelog для неактивных задач (фильтр через helper.is_active)
- helper.py содержит функции `extract_status_transitions`, `build_timeline`, `aggregate_phase_days`
- Алгоритм построения timeline в helper.py соответствует разделу 7 SPEC.md

### Шаг 2. Установить

```bash
cp -r skill/timing-analyzer ~/.gigacode/skills/
```

### Шаг 3. Запустить

```bash
gigacode
```

В чате: "запусти timing-analyzer".

## Сколько времени занимает

Зависит от количества активных задач:
- Из 28 задач Натальи активных обычно 8-12
- Каждая = 1 jira_get_issue с changelog + 0.1 сек пауза = ~10-15 сек
- Расчёт timing локально через helper — мгновенно

Итого 15-30 секунд.

## Проверка результата

```bash
python3 -c "
import json
d = json.load(open('.cache/enriched.json'))
active = [t for t in d['tasks'] if t.get('timing', {}).get('computed')]
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
ASFC-51561            A= 363.2  R=   0.0  T=   0.0
...
```

## Что значит "факт А/Р/Т в календарных днях"

**Это НЕ чел-дни.** Это **календарное время** проведённое задачей в статусах соответствующей фазы.

Например, `CRSIGMA-23749` с факт R = 240 дней означает: задача суммарно провела 240 календарных дней в статусе `In Progress` (с момента первого перехода туда до сейчас или до выхода в другую фазу). Это включает выходные, ожидания, переключения разработчика на другие задачи.

Сравнение факт vs план тогда — это **сигнал** "стоит проверить эту задачу", а не точная метрика переработки.

Все эти оговорки попадут в дисклеймер итогового отчёта (это работа `report-builder`).

## Следующий шаг

Запустить `report-builder` — построит итоговый markdown отчёт со всеми колонками.
