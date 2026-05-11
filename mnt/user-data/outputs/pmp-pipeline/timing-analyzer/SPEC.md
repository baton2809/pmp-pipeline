# SPEC: timing-analyzer

> Третий скилл pipeline. Читает `.cache/enriched.json` после `jira-enricher`. Для **активных** задач (статус не not_started и не finished) делает второй `jira_get_issue` с `expand=changelog`, парсит историю переходов статусов, считает календарные дни в каждой фазе А/Р/Т. Дополняет файл полем `timing` у этих задач.
>
> Это самый "тяжёлый" скилл по контексту (changelog может быть большим), поэтому **обрабатывает только активные задачи**, не все 28.

## 1. Место в pipeline

```
1. excel-parser
2. jira-enricher              → .cache/enriched.json с jira-данными
3. timing-analyzer  ◄── (этот скилл)
4. report-builder
```

## 2. Цели

- Прочитать `.cache/enriched.json`, валидировать что `jira-enricher` отработал
- Отфильтровать задачи: только те где `task.jira.status_category` в `["analysis", "development", "testing"]` и `task.jira.found = true`
- Для каждой такой задачи сделать `jira_get_issue` с `expand=changelog`
- Извлечь историю переходов статусов (только `items[].field == "status"`)
- Построить timeline статусов с timestamps
- Применить mapping статус→фаза, сгруппировать интервалы по фазам
- Посчитать `phase_days = {A, R, T, not_started, finished, unknown}`
- Записать `task.timing` для каждой обработанной задачи
- Для задач не-активных (not_started/finished/unknown) — `timing.computed = false`, `phase_days` нули
- Перезаписать `.cache/enriched.json`

## 3. Анти-цели

- **НЕ** обрабатывать not_started задачи (Backlog/TO DO) — у них нет интересной истории
- **НЕ** обрабатывать finished задачи — для них факт уже не важен (или важен, но это v3.1)
- **НЕ** хранить сырой changelog в `enriched.json` — только агрегированные `phase_days`
- **НЕ** генерировать markdown — это работа report-builder
- **НЕ** делать `jira_search` — это работа jira-enricher

## 4. Вход и выход

### Вход

`.cache/enriched.json` с `metadata.skills_completed` содержащим `["excel-parser", "jira-enricher"]`. Если нет — попросить запустить предшественников, завершить.

### Выход

Тот же файл с дополнениями:
- `metadata.timing_at = "<now ISO>"`
- `metadata.skills_completed` += `"timing-analyzer"`
- `metadata.timing_stats` — статистика обработки
- У активных задач — заполненное `task.timing` (см. CONTRACT.md "После timing-analyzer")
- У не-активных задач — `task.timing = {computed: false, phase_days: {все нули}}`

## 5. Формула вызова MCP — зафиксирована

### Для каждой активной задачи

```
Tool: jira_get_issue

Параметры:
  key = <cr_key из enriched.tasks>
  expand = "changelog"
  fields = "summary,status,created,updated,resolutiondate"
```

**Минимальный** `fields` — нам нужны только базовые поля + changelog. Остальное уже есть из jira-enricher.

`expand=changelog` ОБЯЗАТЕЛЕН — без него история переходов не вернётся.

## 6. Фильтрация активных задач

```python
def is_active(task):
    jira = task.get('jira')
    if not jira or not jira.get('found'):
        return False
    category = jira.get('status_category')
    return category in ('analysis', 'development', 'testing')
```

Реализуется в helper.py.

Для **не-активных** задач timing записывается тривиально:

```json
"timing": {
  "computed": false,
  "phase_days": {"A": 0, "R": 0, "T": 0, "not_started": 0, "finished": 0, "unknown": 0},
  "transitions_count": 0,
  "first_transition": null,
  "last_transition": null,
  "reason": "not_active" 
  // либо "not_found" если jira.found=false
}
```

## 7. Алгоритм расчёта phase_days

Это самая ответственная часть. Реализуется в helper.py функция `compute_phase_days(changelog, task_created, task_resolutiondate, status_to_phase_fn)`.

### Шаг 7.1. Извлечь переходы статусов

```python
def extract_status_transitions(changelog):
    transitions = []
    for history in changelog.get('histories', []):
        for item in history.get('items', []):
            if item.get('field') == 'status':
                transitions.append({
                    'created': parse_iso(history['created']),
                    'from_string': item.get('fromString'),
                    'to_string': item.get('toString'),
                })
    return sorted(transitions, key=lambda t: t['created'])
```

### Шаг 7.2. Построить timeline

```python
def build_timeline(transitions, task_created, task_resolutiondate, now):
    if not transitions:
        # Нет переходов — задача всё время в текущем статусе
        # Это случай "только что создана" или "никогда не двигалась"
        return []  # без timeline — фазы посчитать нельзя
    
    timeline = []
    
    # Первый интервал: от создания задачи до первого перехода
    # Статус в этом интервале = from_string первого перехода
    first = transitions[0]
    timeline.append({
        'status': first['from_string'] or first['to_string'],
        'from': task_created,
        'to': first['created']
    })
    
    # Между переходами
    for i in range(len(transitions) - 1):
        timeline.append({
            'status': transitions[i]['to_string'],
            'from': transitions[i]['created'],
            'to': transitions[i+1]['created']
        })
    
    # Последний интервал: после последнего перехода
    last = transitions[-1]
    last_status = last['to_string']
    
    # Конец = resolutiondate если задача finished, иначе now
    if task_resolutiondate:
        end = parse_iso(task_resolutiondate)
    else:
        end = now
    
    timeline.append({
        'status': last_status,
        'from': last['created'],
        'to': end
    })
    
    return timeline
```

### Шаг 7.3. Сгруппировать по фазам

```python
def aggregate_phase_days(timeline, status_to_phase_fn):
    phase_days = {'A': 0.0, 'R': 0.0, 'T': 0.0, 
                  'not_started': 0.0, 'finished': 0.0, 'unknown': 0.0}
    
    for interval in timeline:
        category, phase = status_to_phase_fn(interval['status'])
        days = (interval['to'] - interval['from']).total_seconds() / 86400.0
        
        # Куда писать: фаза или категория
        if phase in ('A', 'R', 'T'):
            phase_days[phase] += days
        elif category in ('not_started', 'finished', 'unknown'):
            phase_days[category] += days
        else:
            phase_days['unknown'] += days
    
    # Округление до 1 знака
    return {k: round(v, 1) for k, v in phase_days.items()}
```

### Замечание про возвраты

Если задача проходила через статус несколько раз (возврат из тестирования в разработку) — алгоритм **естественно** суммирует все интервалы. Это правильное поведение для метрики "сколько всего календарных дней работали над этой задачей в фазе Р".

## 8. Steps

### Step 1. Валидация

Прочитать `.cache/enriched.json`. Проверить:
- Файл существует
- `metadata.skills_completed` содержит `"jira-enricher"`
- Массив `tasks` не пустой

Если что-то не так — сообщить и завершить.

### Step 2. Фильтрация задач

Пройти по `tasks`, разделить на:
- `active_tasks` — те для которых нужен changelog (helper.is_active)
- `inactive_tasks` — остальные (получат тривиальный timing)

В чат сообщить: "Активных задач для анализа: N, неактивных (тривиальный timing): M"

### Step 3. Для каждой активной — jira_get_issue с changelog

Последовательно (НЕ параллельно). Для каждой задачи:

1. Tool call `jira_get_issue` с параметрами из раздела 5 — это нативный tool call, **не Python-обёртка**
2. Получить JSON-ответ
3. Если ошибка — записать `task.timing = {computed: false, reason: "fetch_error", error: <текст>}`, продолжить
4. Если changelog пустой (нет histories) — записать `timing = {computed: false, reason: "no_changelog"}`
5. Если changelog есть:
   - Вызвать `helper.extract_status_transitions(changelog)`
   - Вызвать `helper.build_timeline(transitions, created, resolutiondate, now)`
   - Вызвать `helper.aggregate_phase_days(timeline, status_to_phase_fn)`
   - Сформировать `task.timing` согласно CONTRACT.md
6. Прогресс каждые 5 задач
7. Пауза 0.1 сек между вызовами

### Step 4. Для неактивных — тривиальный timing

Пройти по `inactive_tasks`, записать каждой:
```json
"timing": {
  "computed": false,
  "phase_days": {все нули},
  "transitions_count": 0,
  "first_transition": null,
  "last_transition": null,
  "reason": <"not_active" | "not_found">
}
```

### Step 5. Обновить metadata

```json
"metadata": {
  ...,
  "timing_at": "<now>",
  "skills_completed": [..., "timing-analyzer"],
  "timing_stats": {
    "tasks_total": 28,
    "tasks_active_analyzed": 8,
    "tasks_inactive_skipped": 20,
    "tasks_with_changelog": 7,
    "tasks_without_changelog": 1,
    "tasks_fetch_error": 0
  }
}
```

### Step 6. Записать json

Перезаписать `.cache/enriched.json` целиком. `indent=2, ensure_ascii=False`.

### Step 7. Сводка

В чат:
- Активных задач обработано: N
- С реальной историей: K
- Без changelog (только что созданы или нет переходов): L
- Топ-3 самых долгих фаз в данных:
  - `CRSIGMA-23749`: Р = 240 д
  - `ASFC-35817`: Р = 698 д
  - ...
- Следующий шаг: запустите `report-builder`

## 9. КРИТИЧНО: формат вызова MCP

Те же правила. SKILL.md — инструкция для агента в чате, **не** Python-скрипт.

**Правильно:**
```
Шаг 3.1. Для каждой задачи из active_tasks:

Сделать tool call jira_get_issue:
  key = <task.cr_key>
  expand = "changelog"
  fields = "summary,status,created,updated,resolutiondate"

Получить JSON-ответ. Передать в helper.compute_timing(response, task) и сохранить результат в task.timing.
```

**Запрещено:**
- `result = mcp_jira_get_issue(...)` — обёртка
- Заглушки, TODO, симуляции
- `def fetch_issue(...)` с псевдокодом
- Расчёты timing не через helper.py а прямо в SKILL.md как код

## 10. Файлы

| Файл | Назначение |
|------|------------|
| `.cache/enriched.json` | Читается и перезаписывается |
| `helper.py` | Функции `is_active`, `extract_status_transitions`, `build_timeline`, `aggregate_phase_days`, `compute_timing`, `parse_iso`, `status_to_phase` |

Тот же запрет: ТОЛЬКО `helper.py`, никаких других `.py` файлов.

## 11. Guardrails

- READ-ONLY для Jira
- Только `jira_get_issue` с `expand=changelog`, никаких других вызовов
- НЕ обрабатывать неактивные задачи через MCP — экономим контекст
- НЕ хранить сырой changelog в json — только phase_days
- НЕ выдумывать timing для задач без changelog — `computed: false`
- НЕ изменять поле `task.jira` (это работа jira-enricher)

## 12. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| jira-enricher не отработал | Сообщить, попросить запустить, завершить |
| 0 активных задач | Заполнить тривиальный timing для всех, сохранить, выйти |
| changelog без переходов статусов (только лейблы, sprint) | `computed: false`, `reason: "no_status_transitions"` |
| `from_string` первого перехода пустой ("") | Статус первого интервала = `to_string` первого перехода (вычислили из контекста) |
| `resolutiondate` есть но статус не finished | Использовать `resolutiondate` как end (что-то странное в данных, но не критично) |
| Возвраты в статус | Алгоритм сам суммирует — это нормально |
| Очень старая задача (798 дней) | Changelog может быть большим — это нормально, агент справится за один вызов. Если ответ обрезан — пометить `reason: "changelog_truncated"` |
| Часовые пояса в timestamps | Парсить через `parse_iso` который учитывает offset (`+0300`) |
| Задача создана в `In Progress` (без перехода из) | `from_string == ""` или `null` — обрабатываем как описано в Step 7.2 |
| Timing уже посчитан (повторный запуск) | Перезаписать |

## 13. Антипаттерны

### Критические

- Запросить changelog для **всех** 28 задач — лишние ~20 запросов, не нужно
- Хранить сырой changelog в `enriched.json` — раздуем файл, потеряем контекст
- Использовать `fields="*"` вместе с `expand=changelog` — катастрофа по контексту
- Считать timing "примерно" вместо алгоритма из раздела 7 — будут расхождения
- Складывать в `phase_days.A` дни статуса который не в категории analysis — баг
- Заглушки в SKILL.md — обязательно реальные tool calls

### Обычные

- Параллельные вызовы Jira
- Не сортировать transitions по timestamp перед построением timeline
- Считать дни через `(d2 - d1).days` (отбрасывает дробную часть) вместо `.total_seconds() / 86400`
- Игнорировать часовые пояса

## 14. Критерий успеха

После запуска:
1. `.cache/enriched.json` валиден, у каждой задачи поле `timing` заполнено
2. У активных задач `timing.computed = true` (если был changelog)
3. У неактивных `timing.computed = false`, `phase_days` нули
4. Сумма `phase_days.A + R + T + not_started + finished + unknown` для каждой задачи ≈ `lead_time_days` (расхождение допустимо до 1-2 дней из-за округлений)
5. `metadata.skills_completed` содержит `[..., "timing-analyzer"]`

## 15. Что отложено

- v3.1: расчёт timing для **finished** задач (для секции "уже закрытые но висели долго")
- v4: timing с разбивкой по sprint (использует `customfield_10007` и Sprint-переходы из changelog)
- v6: расчёт **загрузки команд** во времени — сколько задач конкретная команда вела параллельно
