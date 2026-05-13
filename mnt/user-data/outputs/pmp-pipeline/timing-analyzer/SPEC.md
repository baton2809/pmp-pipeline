# SPEC: timing-analyzer (v3.1)

> Третий скилл pipeline. Читает `pipeline/enriched.json` после `jira-enricher`. Для **активных** задач (статус не not_started и не finished) делает второй `jira_get_issue` с `expand=changelog`, парсит историю переходов статусов, считает календарные дни в каждой фазе А/Р/Т.
>
> **Архитектурное правило (то же что в `jira-enricher`):** агент делает нативные tool calls в чате, накапливает результаты, передаёт батчи в `helper.py` через bash + stdin. Python никогда не вызывает MCP.

---

## 1. Место в pipeline

```
1. excel-parser
2. jira-enricher              → pipeline/enriched.json с jira-данными
3. timing-analyzer  ◄── (этот скилл)
4. report-builder
```

## 2. Цели

- Прочитать `pipeline/enriched.json`, валидировать что `jira-enricher` отработал
- Отфильтровать **активные** задачи: `task.jira.status_category in ["analysis", "development", "testing"]` и `task.jira.found = true`
- Для каждой такой задачи сделать `jira_get_issue` с `expand=changelog`
- Извлечь историю переходов статусов
- Построить timeline с timestamps
- Применить mapping статус→фаза, сгруппировать интервалы по фазам
- Посчитать `phase_days = {A, R, T, not_started, finished, unknown}`
- Записать `task.timing` для каждой обработанной задачи
- Для не-активных задач — `timing.computed = false`, `phase_days` нули
- Перезаписать `pipeline/enriched.json`
- Создать `pipeline/step-3-after-timing-analyzer.md`

## 3. Анти-цели

- **НЕ** обрабатывать not_started задачи (нет интересной истории)
- **НЕ** обрабатывать finished задачи (для v3 — отложено в v3.2)
- **НЕ** хранить сырой changelog в `enriched.json` — только агрегированные `phase_days`
- **НЕ** генерировать финальный markdown — только step-3 snapshot
- **НЕ** делать `jira_search` — это работа jira-enricher

## 4. Формула вызова MCP — зафиксирована

### Для каждой активной задачи (нативный tool call агента)

```
Tool: jira_get_issue

Параметры:
  key = <cr_key>
  expand = "changelog"
  fields = "summary,status,created,updated,resolutiondate"
```

- `expand=changelog` ОБЯЗАТЕЛЕН
- `fields` минимальный — нам нужны только базовые поля (timestamp создания, имя статуса, дата закрытия)
- Это нативный tool call агента в чате, **не Python-функция**

## 5. Готовый код helper.py — используйте как основу

```python
# helper.py для timing-analyzer

import json
import sys
import re
import os
from datetime import datetime, timezone
from collections import OrderedDict

# === Mapping (то же что в jira-enricher) ===

STATUS_MAP = {
    'backlog': ('not_started', None),
    'to do': ('not_started', None),
    'открыта': ('not_started', None),
    'new': ('analysis', 'A'),
    'need info': ('analysis', 'A'),
    'analysis': ('analysis', 'A'),
    'анализ': ('analysis', 'A'),
    'in progress': ('development', 'R'),
    'разработка': ('development', 'R'),
    'готов к разработке': ('development', 'R'),
    'ready for qa': ('testing', 'T'),
    'готов к тестированию': ('testing', 'T'),
    'начато тестирование': ('testing', 'T'),
    'тестирование': ('testing', 'T'),
    'st': ('testing', 'T'),
    'ift': ('testing', 'T'),
    'uat': ('testing', 'T'),
    'пси': ('testing', 'T'),
    'проверено на ифт/гот': ('testing', 'T'),
    'in discovery': ('testing', 'T'),
    'done': ('finished', None),
    'resolved': ('finished', None),
    'closed': ('finished', None),
    'закрыт': ('finished', None),
    'закрыты': ('finished', None),
    'cancelled': ('finished', None),
}

def map_status(status_name):
    if not status_name:
        return ('unknown', None)
    return STATUS_MAP.get(str(status_name).strip().lower(), ('unknown', None))

# === Парсинг timestamps ===

def parse_iso(s):
    """Парсит ISO 8601 с offset (например '2026-02-13T18:24:19.841+0300')."""
    if not s:
        return None
    if re.search(r'[+-]\d{4}$', s):
        s = s[:-2] + ':' + s[-2:]
    try:
        return datetime.fromisoformat(s)
    except (ValueError, TypeError):
        return None

# === Алгоритм расчёта phase_days ===

def extract_status_transitions(changelog):
    """Извлечь переходы статусов из changelog. Возвращает отсортированный список."""
    transitions = []
    for history in (changelog or {}).get('histories', []) or []:
        created = parse_iso(history.get('created'))
        if not created:
            continue
        for item in history.get('items', []) or []:
            if item.get('field') == 'status':
                transitions.append({
                    'created': created,
                    'from_string': item.get('fromString'),
                    'to_string': item.get('toString'),
                })
    return sorted(transitions, key=lambda t: t['created'])

def build_timeline(transitions, task_created_str, task_resolutiondate_str, now_dt=None):
    """Построить timeline статусов с интервалами.
    
    Возвращает список dict вида {status, from, to}.
    Если переходов нет — возвращает пустой список (timing не считаем)."""
    if not transitions:
        return []

    task_created = parse_iso(task_created_str)
    resolved = parse_iso(task_resolutiondate_str) if task_resolutiondate_str else None
    now = now_dt or datetime.now(timezone.utc).astimezone()

    timeline = []

    # Первый интервал: от created задачи до первого перехода
    first = transitions[0]
    initial_status = first['from_string'] or first['to_string']
    if task_created and first['created'] >= task_created:
        timeline.append({
            'status': initial_status,
            'from': task_created,
            'to': first['created'],
        })

    # Между переходами
    for i in range(len(transitions) - 1):
        timeline.append({
            'status': transitions[i]['to_string'],
            'from': transitions[i]['created'],
            'to': transitions[i+1]['created'],
        })

    # Последний интервал
    last = transitions[-1]
    last_status = last['to_string']
    category, _ = map_status(last_status)
    if category == 'finished' and resolved:
        end = resolved
    else:
        end = now
    if end >= last['created']:
        timeline.append({
            'status': last_status,
            'from': last['created'],
            'to': end,
        })

    return timeline

def aggregate_phase_days(timeline):
    """Сгруппировать интервалы timeline по фазам, вернуть phase_days."""
    phase_days = {'A': 0.0, 'R': 0.0, 'T': 0.0,
                  'not_started': 0.0, 'finished': 0.0, 'unknown': 0.0}

    for interval in timeline:
        category, phase = map_status(interval['status'])
        days = (interval['to'] - interval['from']).total_seconds() / 86400.0
        if days < 0:
            continue  # на всякий случай
        if phase in ('A', 'R', 'T'):
            phase_days[phase] += days
        elif category in ('not_started', 'finished', 'unknown'):
            phase_days[category] += days
        else:
            phase_days['unknown'] += days

    return {k: round(v, 1) for k, v in phase_days.items()}

def compute_timing(jira_response, task_created, task_resolutiondate):
    """Главная функция — принимает ответ jira_get_issue с expand=changelog,
    возвращает структуру для task.timing."""
    if not jira_response:
        return {
            'computed': False,
            'phase_days': {'A': 0, 'R': 0, 'T': 0, 'not_started': 0, 'finished': 0, 'unknown': 0},
            'transitions_count': 0,
            'first_transition': None,
            'last_transition': None,
            'reason': 'no_response',
        }

    changelog = jira_response.get('changelog')
    if not changelog:
        return {
            'computed': False,
            'phase_days': {'A': 0, 'R': 0, 'T': 0, 'not_started': 0, 'finished': 0, 'unknown': 0},
            'transitions_count': 0,
            'first_transition': None,
            'last_transition': None,
            'reason': 'no_changelog',
        }

    transitions = extract_status_transitions(changelog)
    if not transitions:
        return {
            'computed': False,
            'phase_days': {'A': 0, 'R': 0, 'T': 0, 'not_started': 0, 'finished': 0, 'unknown': 0},
            'transitions_count': 0,
            'first_transition': None,
            'last_transition': None,
            'reason': 'no_status_transitions',
        }

    timeline = build_timeline(transitions, task_created, task_resolutiondate)
    phase_days = aggregate_phase_days(timeline)

    return {
        'computed': True,
        'phase_days': phase_days,
        'transitions_count': len(transitions),
        'first_transition': transitions[0]['created'].isoformat() if transitions else None,
        'last_transition': transitions[-1]['created'].isoformat() if transitions else None,
        'computed_at': now_iso(),
    }

def trivial_timing(reason='not_active'):
    """Заглушка для неактивных задач."""
    return {
        'computed': False,
        'phase_days': {'A': 0, 'R': 0, 'T': 0, 'not_started': 0, 'finished': 0, 'unknown': 0},
        'transitions_count': 0,
        'first_transition': None,
        'last_transition': None,
        'reason': reason,
    }

def is_active(task):
    """Активна ли задача? (Нужно ли считать timing через changelog)"""
    jira = task.get('jira')
    if not jira or not jira.get('found'):
        return False
    return jira.get('status_category') in ('analysis', 'development', 'testing')

def now_iso():
    return datetime.now(timezone.utc).astimezone().isoformat()

# === CLI entry-points ===

def list_active(enriched_path='pipeline/enriched.json'):
    """Вывести в stdout список cr_key активных задач (тех для кого нужен changelog).
    Используется так:
        python3 helper.py list-active
    Возвращает JSON-массив на stdout."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    active = [t['cr_key'] for t in enriched['tasks'] if is_active(t)]
    print(json.dumps(active, ensure_ascii=False))

def fill_inactive(enriched_path='pipeline/enriched.json'):
    """Заполнить task.timing для всех НЕ активных задач (тривиальное значение)."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    
    filled = 0
    for task in enriched['tasks']:
        if not is_active(task):
            jira = task.get('jira') or {}
            if not jira.get('found'):
                reason = 'not_found'
            else:
                reason = 'not_active'
            task['timing'] = trivial_timing(reason=reason)
            filled += 1
    
    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)
    
    print(f"Filled trivial timing for {filled} non-active tasks")

def merge_timing_batch(enriched_path='pipeline/enriched.json'):
    """Принять batch результатов timing через stdin, мерджить в enriched.json.
    
    Формат входа:
    [
      {
        "cr_key": "CRSIGMA-23749",
        "timing": {...результат compute_timing...}
      },
      ...
    ]
    
    Использование:
        echo '<batch>' | python3 helper.py merge-batch
    """
    batch_text = sys.stdin.read()
    batch = json.loads(batch_text)

    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    by_key = {item['cr_key']: item['timing'] for item in batch}
    updated = 0
    for task in enriched['tasks']:
        if task['cr_key'] in by_key:
            task['timing'] = by_key[task['cr_key']]
            updated += 1

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)
    
    print(f"Merged timing for {updated} active tasks")

def compute_from_response(enriched_path='pipeline/enriched.json'):
    """Принять через stdin {cr_key, response} — где response это ответ MCP с changelog.
    Посчитать timing через compute_timing() и записать в enriched.json.
    
    Использование:
        echo '{"cr_key": "...", "response": {...полный JSON от MCP...}}' \
            | python3 helper.py compute-from-response
    
    Это удобнее чем agent сам формирует timing — он передаёт сырой response,
    Python применяет алгоритм."""
    payload = json.loads(sys.stdin.read())
    cr_key = payload['cr_key']
    response = payload['response']

    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    
    # Найти задачу и получить created/resolutiondate из jira
    task = None
    for t in enriched['tasks']:
        if t['cr_key'] == cr_key:
            task = t
            break
    
    if not task:
        print(f"ERROR: cr_key {cr_key} not found in enriched.json", file=sys.stderr)
        sys.exit(1)
    
    jira = task.get('jira') or {}
    task_created = jira.get('created')
    task_resolutiondate = jira.get('resolutiondate')
    
    timing = compute_timing(response, task_created, task_resolutiondate)
    task['timing'] = timing
    
    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)
    
    p = timing['phase_days']
    print(f"{cr_key}: A={p['A']} R={p['R']} T={p['T']} (transitions={timing.get('transitions_count', 0)})")

def finalize(enriched_path='pipeline/enriched.json'):
    """Обновить metadata.skills_completed."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    
    enriched['metadata']['timing_at'] = now_iso()
    completed = enriched['metadata'].setdefault('skills_completed', [])
    if 'timing-analyzer' not in completed:
        completed.append('timing-analyzer')
    
    tasks = enriched['tasks']
    stats = {
        'tasks_total': len(tasks),
        'tasks_active_computed': sum(1 for t in tasks 
                                       if (t.get('timing') or {}).get('computed')),
        'tasks_inactive_or_no_changelog': sum(1 for t in tasks 
                                                if t.get('timing') and not t['timing'].get('computed')),
    }
    enriched['metadata']['timing_stats'] = stats
    
    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)
    
    print(json.dumps(stats, ensure_ascii=False))

def write_step3_markdown(enriched_path='pipeline/enriched.json',
                          md_path='pipeline/step-3-after-timing-analyzer.md'):
    """Создать читаемый snapshot после timing-analyzer."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    
    tasks = enriched['tasks']
    computed_tasks = [t for t in tasks if (t.get('timing') or {}).get('computed')]
    
    md = []
    md.append("# Снимок после timing-analyzer\n")
    md.append(f"**Дата:** {enriched['metadata'].get('timing_at')}\n")
    md.append(f"**Активных задач с timing:** {len(computed_tasks)}\n")
    md.append(f"**Без changelog или неактивных:** {len(tasks) - len(computed_tasks)}\n")
    
    # Сортируем по максимальной фазе по убыванию
    def max_phase(t):
        p = (t.get('timing') or {}).get('phase_days') or {}
        return max(p.get('A', 0), p.get('R', 0), p.get('T', 0))
    
    sorted_tasks = sorted(computed_tasks, key=max_phase, reverse=True)
    
    md.append("\n## Топ-10 задач с самыми долгими фазами\n")
    md.append("| CR | Статус | Факт А (д) | Факт Р (д) | Факт Т (д) | План А/Р/Т |")
    md.append("|-----|--------|------------|------------|------------|-------------|")
    for t in sorted_tasks[:10]:
        cr = t['cr_key']
        status = (t.get('jira') or {}).get('status', '—')
        p = t['timing']['phase_days']
        plan = t.get('plan') or {}
        def fmt(v):
            return str(int(v)) if v is not None else '0'
        plan_str = f"{fmt(plan.get('analytics'))}/{fmt(plan.get('development'))}/{fmt(plan.get('testing'))}"
        md.append(f"| {cr} | {status} | {int(p['A'])} | {int(p['R'])} | {int(p['T'])} | {plan_str} |")
    
    md.append("\n## Следующий шаг\n")
    md.append("Запустить `report-builder` — соберёт финальный `report.md`.")
    
    with open(md_path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(md))

# === Main ===

if __name__ == '__main__':
    cmd = sys.argv[1] if len(sys.argv) > 1 else 'help'
    if cmd == 'list-active':
        list_active()
    elif cmd == 'fill-inactive':
        fill_inactive()
    elif cmd == 'compute-from-response':
        compute_from_response()
    elif cmd == 'merge-batch':
        merge_timing_batch()
    elif cmd == 'finalize':
        finalize()
    elif cmd == 'write-step3':
        write_step3_markdown()
    else:
        print("Usage: python3 helper.py [list-active|fill-inactive|compute-from-response|merge-batch|finalize|write-step3]")
        sys.exit(1)
```

**Главное про этот код:**
- `compute_timing(response, created, resolutiondate)` — главная функция расчёта, **алгоритм валидирован**
- `compute-from-response` — самый удобный entry point: агент передаёт сырой ответ MCP, Python сам считает
- Возвраты задачи в один статус суммируются автоматически (это правильно)
- Работа с часовыми поясами через `parse_iso`

## 6. Steps — как агент работает в чате

### Step 1. Валидация

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py list-active
```

helper выведет JSON-массив cr_keys активных задач — те для которых нужен changelog. Например `["CRSIGMA-23749", "ASFC-58741", ...]`.

Если файл не существует или `jira-enricher` не отработал — helper упадёт с понятной ошибкой. Сообщить пользователю запустить предшественника.

### Step 2. Заполнить тривиальный timing для неактивных задач

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py fill-inactive
```

helper пройдёт по всем не активным задачам, запишет `timing.computed = false`, нули в `phase_days`.

### Step 3. Для каждой активной задачи — нативный tool call + compute

Для каждого `cr_key` из списка активных:

**Step 3.1.** Агент в чате делает нативный tool call:

```
Tool: jira_get_issue
key = <cr_key>
expand = "changelog"
fields = "summary,status,created,updated,resolutiondate"
```

**Step 3.2.** Агент видит JSON-ответ в контексте (включая `changelog.histories`).

**Step 3.3.** Передаёт ответ в helper для расчёта через bash:

```bash
echo '<JSON: {"cr_key": "...", "response": <полный JSON от MCP>}>' \
    | python3 ~/.gigacode/skills/timing-analyzer/helper.py compute-from-response
```

helper.py:
- Парсит JSON со stdin
- Извлекает changelog
- Применяет `compute_timing()` — алгоритм построения timeline и агрегации phase_days
- Записывает результат в `pipeline/enriched.json`
- Печатает короткую сводку в stdout (например: "CRSIGMA-23749: A=30 R=240 T=0")

Агент видит вывод, продолжает со следующей задачей.

**Пауза 0.2 сек** между задачами (rate limit).

**Прогресс** каждые 3 задачи: "Обработано 3/14".

### Step 4. Финализация

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py finalize
```

Обновляет `metadata.timing_at`, добавляет `"timing-analyzer"` в `skills_completed`, заполняет `timing_stats`.

### Step 5. Создать markdown-снимок

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py write-step3
```

Создаст `pipeline/step-3-after-timing-analyzer.md` с топ-10 самых долгих фаз.

### Step 6. Сводка пользователю

В чат:
- Активных задач с timing: N
- Без changelog или неактивных: M
- Топ-3 самых долгих по факту: краткий список
- Созданы файлы:
  - `pipeline/enriched.json` (обновлён)
  - `pipeline/step-3-after-timing-analyzer.md`
- Следующий шаг: запустите `report-builder`

## 7. КРИТИЧНО: формат вызова MCP

### ПРАВИЛЬНО

```
Шаг 3.1. Сделать tool call:
  Tool: jira_get_issue
  key = <cr_key>
  expand = "changelog"
  fields = "summary,status,created,updated,resolutiondate"

Получить JSON-ответ. Передать в helper через bash + stdin:

echo '{"cr_key": "<cr_key>", "response": <ответ MCP как есть>}' \
    | python3 ~/.gigacode/skills/timing-analyzer/helper.py compute-from-response
```

### ❌ НЕПРАВИЛЬНО

```python
# MCP нельзя вызвать из Python!
result = mcp__Atlassian__jira_get_issue(key="...", expand="changelog")
# NameError
```

Граница: tool call → агент видит JSON → bash + stdin → Python считает.

## 8. Файлы

| Файл | Назначение |
|------|------------|
| `pipeline/enriched.json` | Читается, перезаписывается с полем `timing` для активных задач |
| `pipeline/step-3-after-timing-analyzer.md` | Читаемый snapshot |
| `helper.py` | CLI с подкомандами `list-active`, `fill-inactive`, `compute-from-response`, `merge-batch`, `finalize`, `write-step3` |

### Запрещённые файлы

Только `helper.py`, никаких `main.py`, `process.py`, `run_*.py`, `generate_*.py`, `__pycache__`, виртуальных окружений.

## 9. Guardrails

- READ-ONLY для Jira (только `jira_get_issue` с `expand=changelog`)
- НЕ обрабатывать неактивные задачи через MCP — экономим контекст
- НЕ хранить сырой changelog в json — только phase_days
- НЕ изменять поле `task.jira` (это работа jira-enricher)
- НЕ выдумывать timing для задач без changelog — `computed: false`

## 10. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| jira-enricher не отработал | helper упадёт с ошибкой на list-active, агент сообщит пользователю |
| 0 активных задач | fill-inactive заполняет всем тривиальный timing, переходим к finalize и step3 |
| changelog без переходов статусов (только labels, sprint) | `computed: false`, `reason: "no_status_transitions"` |
| `from_string` первого перехода пустой | helper использует `to_string` как initial_status |
| `resolutiondate` есть но статус не finished | helper использует resolved как end (странно, но не критично) |
| Возвраты в статус | Алгоритм сам суммирует — это нормально |
| Очень старая задача (798 дней) | changelog может быть большим — это нормально |
| Часовые пояса в timestamps | helper.parse_iso учитывает |
| Задача создана в `In Progress` | from_string первого перехода пустой, обрабатывается |
| Timing уже посчитан (повторный запуск) | Перезаписать |

## 11. Антипаттерны

### Критические

- **Вызывать MCP из Python** (`mcp__Atlassian__jira_get_issue`, `from mcp_atlassian import`) — невозможно, NameError
- **Запросить changelog для всех 28 задач** — лишние ~20 запросов, не нужно
- **Использовать `fields="*"` + `expand=changelog`** — катастрофа по контексту
- **Создавать `main.py`, `process.py`** — только `helper.py`
- **Передавать batch через CLI-аргумент** — только stdin
- **Считать timing "примерно" вместо алгоритма** — расхождения будут существенные

### Обычные

- Параллельные вызовы Jira — только последовательно
- Не сортировать transitions перед построением timeline (но helper.extract_status_transitions сам сортирует)
- Считать дни через `(d2 - d1).days` (отбрасывает часы) вместо `.total_seconds() / 86400`
- Игнорировать часовые пояса

## 12. Критерий успеха

После запуска:
1. `pipeline/enriched.json` валиден, у каждой задачи поле `timing` заполнено
2. У активных задач `timing.computed = true` (если был changelog)
3. У неактивных `timing.computed = false`, `phase_days` нули
4. `metadata.skills_completed` содержит `[..., "timing-analyzer"]`
5. Создан `pipeline/step-3-after-timing-analyzer.md`

## 13. Что отложено

- v3.2: расчёт timing для **finished** задач (для секции "уже закрытые но висели долго")
- v4: timing с разбивкой по sprint (использует Sprint-переходы из changelog)
- v6: расчёт **загрузки команд** во времени
