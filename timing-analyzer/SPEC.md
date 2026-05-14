# SPEC: timing-analyzer (v3.2)

> Третий скилл pipeline. Читает `pipeline/enriched.json` после `jira-enricher`. Для **активных** задач делает второй `jira_get_issue` с `expand=changelog`, парсит историю переходов статусов, считает календарные дни в каждой фазе А/Р/Т.
>
> **Главные изменения v3.2 vs v3.1:**
> 1. Структура changelog в ответе MCP — `changelogs` (множественное число), без обёртки `histories`
> 2. Поля переходов — `fromString`/`toString` (camelCase) только для `field == 'status'`
> 3. Архитектура передачи: **WriteFile tool агента** в рабочую директорию, **не** stdin/echo. Filesystem Guard блокирует `.gigacode/tmp/`.
> 4. Streaming: после **каждой** задачи записываем JSON в файл и сразу вызываем helper. НЕ копим в контексте.

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
- Получить список **активных** задач: `task.jira.status_category in ["analysis", "development", "testing"]` и `task.jira.found = true`
- Для каждой активной задачи (по одной):
  1. Tool call `jira_get_issue(key, expand="changelog")`
  2. Сразу **WriteFile** ответа в `pipeline/tmp/<cr_key>.json` в рабочей директории
  3. Вызвать `python3 helper.py compute-from-file pipeline/tmp/<cr_key>.json`
  4. helper.py парсит файл, считает timing, мерджит в enriched.json
- Для не-активных задач — заполнить тривиальный timing через `fill-inactive`
- Создать `pipeline/step-3-after-timing-analyzer.md`
- Очистить `pipeline/tmp/` (опционально)

## 3. Анти-цели

- **НЕ** запрашивать changelog для not_started / finished задач
- **НЕ** хранить JSON-ответы MCP в контексте агента — переполняется при 24 задачах
- **НЕ** пытаться передать changelog через stdin/echo — слишком большой
- **НЕ** записывать в `~/.gigacode/tmp/` — Filesystem Guard блокирует
- **НЕ** создавать `main.py`, `process_timing.py`, `run_timing.sh`, `batch_timing.json` или любые другие файлы кроме `helper.py`, `enriched.json`, `step-3.md`, `pipeline/tmp/<key>.json`

## 4. Формула вызова MCP

### Для каждой активной задачи (нативный tool call агента)

```
Tool: jira_get_issue

Параметры:
  key = <cr_key>
  expand = "changelog"
  fields = "summary,status,created,updated,resolutiondate"
```

- `expand=changelog` ОБЯЗАТЕЛЕН
- `fields` — минимальный (5 полей)
- Это нативный tool call агента

## 5. Структура ответа MCP — зафиксированная

**Подтверждено на реальных вызовах** в GigaCode CLI к Сбер-MCP:

```python
{
    'key': 'ASFC-67203',
    'fields': {
        'summary': '...',
        'status': {'name': 'Ready for QA'},
        'created': '2026-04-23T12:40:49+0300',
        'updated': '...',
        'resolutiondate': None,
    },
    'changelogs': [                              # ← МНОЖЕСТВЕННОЕ ЧИСЛО
        {
            'created': '2025-08-14T13:32:23.287+0300',
            'items': [
                {
                    'field': 'status',
                    'fromString': 'New',         # ← camelCase для status
                    'toString': 'In Progress',
                },
                {
                    'field': 'Link',
                    'to_string': '...',          # ← другие поля могут быть snake_case
                },
            ]
        },
    ]
}
```

**Важно про структуру:**
1. Ключ верхнего уровня — **`changelogs`** (множественное число). НЕ `changelog`.
2. **Нет** промежуточной обёртки `histories` — массив сразу содержит элементы.
3. Для `item.field == 'status'` — поля **`fromString`** и **`toString`** (camelCase)
4. Для других значений `field` (`Link`, и т.д.) — могут быть другие имена полей. Эти переходы нам **не нужны** — фильтруем только `field == 'status'`.

### Граничный случай

Если у задачи **нет переходов статуса** — `changelogs` пустой или содержит только non-status элементы. Тогда `timing.computed = false` с причиной `no_status_transitions`.

## 6. Архитектура передачи данных — критическая часть

### Главное правило

Контекст агента переполняется при 24 задачах с changelog (каждый ответ 600-800 строк JSON). Поэтому:

```
ПЛОХО (переполнит контекст):
  1. 24 tool calls подряд
  2. Копим JSON-ответы в контексте агента
  3. В конце вызываем helper

ХОРОШО (streaming, обрабатываем по одной):
  Для каждой active задачи:
    1. Tool call jira_get_issue (получаем JSON в контекст)
    2. WriteFile JSON в pipeline/tmp/<cr_key>.json
    3. Shell: python3 helper.py compute-from-file pipeline/tmp/<cr_key>.json
    4. helper.py прочитал, посчитал, записал в enriched.json
    5. JSON-ответ выпадает из контекста — он уже на диске
  Следующая задача начинается с пустого контекста по JSON
```

### Tool который записывает на диск

GigaCode CLI имеет встроенный **WriteFile tool** (это не bash и не python). Использовать **только его** для записи JSON-ответов MCP.

```
Tool: WriteFile
path = "pipeline/tmp/CRSIGMA-26516.json"
content = <сырой JSON-ответ от jira_get_issue>
```

### Что запрещено

- **`echo '...' > file.json` через bash** — длина команды и кавычки ломают большие JSON
- **`python3 -c "..."` с JSON в коде** — то же
- **Запись в `~/.gigacode/tmp/`** — Filesystem Guard блокирует (`Filesystem Guard denied`)
- **Любые пути содержащие `.gigacode/`** — заблокировано

Только **рабочая директория проекта** через **WriteFile tool агента**.

## 7. Готовый код helper.py — используйте как основу

```python
# helper.py для timing-analyzer

import json
import sys
import re
import os
import glob
from datetime import datetime, timezone

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

def parse_iso(s):
    """Парсит ISO 8601 с offset (например '2026-02-13T18:24:19.841+0300')."""
    if not s:
        return None
    s = str(s)
    if re.search(r'[+-]\d{4}$', s):
        s = s[:-2] + ':' + s[-2:]
    try:
        return datetime.fromisoformat(s)
    except (ValueError, TypeError):
        return None

# === Алгоритм расчёта phase_days ===

def extract_status_transitions(changelog_list):
    """Извлечь переходы статусов из ответа MCP.
    
    changelog_list — значение поля 'changelogs' из ответа (МНОЖЕСТВЕННОЕ число).
    Плоский список без обёртки 'histories'.
    
    Возвращает отсортированный по timestamp список переходов статуса.
    Фильтрует ТОЛЬКО элементы с field == 'status'.
    """
    transitions = []
    for entry in (changelog_list or []):
        created = parse_iso(entry.get('created'))
        if not created:
            continue
        for item in entry.get('items', []) or []:
            if item.get('field') != 'status':
                continue
            # Для статуса MCP возвращает camelCase: fromString, toString
            transitions.append({
                'created': created,
                'from_string': item.get('fromString'),
                'to_string': item.get('toString'),
            })
    return sorted(transitions, key=lambda t: t['created'])

def build_timeline(transitions, task_created_str, task_resolutiondate_str, now_dt=None):
    """Построить timeline статусов с интервалами."""
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
    """Сгруппировать интервалы timeline по фазам."""
    phase_days = {'A': 0.0, 'R': 0.0, 'T': 0.0,
                  'not_started': 0.0, 'finished': 0.0, 'unknown': 0.0}

    for interval in timeline:
        category, phase = map_status(interval['status'])
        days = (interval['to'] - interval['from']).total_seconds() / 86400.0
        if days < 0:
            continue
        if phase in ('A', 'R', 'T'):
            phase_days[phase] += days
        elif category in ('not_started', 'finished', 'unknown'):
            phase_days[category] += days
        else:
            phase_days['unknown'] += days

    return {k: round(v, 1) for k, v in phase_days.items()}

def _trivial(reason='not_active'):
    """Тривиальный timing — все нули."""
    return {
        'computed': False,
        'phase_days': {'A': 0, 'R': 0, 'T': 0, 'not_started': 0, 'finished': 0, 'unknown': 0},
        'transitions_count': 0,
        'first_transition': None,
        'last_transition': None,
        'reason': reason,
    }

def compute_timing(jira_response, task_created, task_resolutiondate):
    """Принимает ПОЛНЫЙ ответ jira_get_issue с expand=changelog."""
    if not jira_response:
        return _trivial('no_response')

    # КРИТИЧНО: ключ 'changelogs' (множественное число)
    changelog_list = jira_response.get('changelogs')
    if not changelog_list:
        return _trivial('no_changelog')

    transitions = extract_status_transitions(changelog_list)
    if not transitions:
        # changelog есть, но переходов status нет
        return _trivial('no_status_transitions')

    timeline = build_timeline(transitions, task_created, task_resolutiondate)
    if not timeline:
        return _trivial('empty_timeline')

    phase_days = aggregate_phase_days(timeline)

    return {
        'computed': True,
        'phase_days': phase_days,
        'transitions_count': len(transitions),
        'first_transition': transitions[0]['created'].isoformat(),
        'last_transition': transitions[-1]['created'].isoformat(),
        'computed_at': now_iso(),
    }

def is_active(task):
    """Активна ли задача? (Нужен ли changelog)"""
    jira = task.get('jira')
    if not jira or not jira.get('found'):
        return False
    return jira.get('status_category') in ('analysis', 'development', 'testing')

def now_iso():
    return datetime.now(timezone.utc).astimezone().isoformat()

# === CLI entry-points ===

def list_active(enriched_path='pipeline/enriched.json'):
    """Вывести JSON-массив cr_key активных задач."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    active = [t['cr_key'] for t in enriched['tasks'] if is_active(t)]
    print(json.dumps(active, ensure_ascii=False))

def fill_inactive(enriched_path='pipeline/enriched.json'):
    """Заполнить task.timing для всех НЕ активных задач."""
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

    filled = 0
    for task in enriched['tasks']:
        if not is_active(task):
            jira = task.get('jira') or {}
            reason = 'not_found' if not jira.get('found') else 'not_active'
            task['timing'] = _trivial(reason=reason)
            filled += 1

    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)

    print(f"Filled trivial timing for {filled} non-active tasks")

def compute_from_file(file_path, enriched_path='pipeline/enriched.json'):
    """Прочитать сырой ответ MCP из файла, посчитать timing, мерджить.
    
    ОСНОВНОЙ ENTRY POINT.
    
    Использование:
        python3 helper.py compute-from-file pipeline/tmp/CRSIGMA-26516.json
    
    Файл должен содержать полный JSON-ответ jira_get_issue с expand=changelog,
    записанный агентом через WriteFile tool сразу после tool call.
    """
    with open(file_path, 'r', encoding='utf-8') as f:
        response = json.load(f)

    cr_key = response.get('key')
    if not cr_key:
        cr_key = os.path.basename(file_path).replace('.json', '')

    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)

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
    mark = '✓' if timing.get('computed') else '✗'
    reason = timing.get('reason', '')
    reason_str = f" reason={reason}" if reason else ''
    print(f"{mark} {cr_key}: A={p['A']} R={p['R']} T={p['T']} "
          f"(transitions={timing.get('transitions_count', 0)}){reason_str}")

def cleanup_tmp(tmp_dir='pipeline/tmp'):
    """Удалить файлы из pipeline/tmp/."""
    if not os.path.isdir(tmp_dir):
        print(f"Directory {tmp_dir} does not exist, nothing to clean")
        return
    files = glob.glob(os.path.join(tmp_dir, '*.json'))
    for f in files:
        os.remove(f)
    print(f"Cleaned {len(files)} files from {tmp_dir}")

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

    print(f"Created {md_path}")

# === Main ===

if __name__ == '__main__':
    cmd = sys.argv[1] if len(sys.argv) > 1 else 'help'
    if cmd == 'list-active':
        list_active()
    elif cmd == 'fill-inactive':
        fill_inactive()
    elif cmd == 'compute-from-file':
        if len(sys.argv) < 3:
            print("Usage: python3 helper.py compute-from-file <path>", file=sys.stderr)
            sys.exit(1)
        compute_from_file(sys.argv[2])
    elif cmd == 'cleanup-tmp':
        cleanup_tmp()
    elif cmd == 'finalize':
        finalize()
    elif cmd == 'write-step3':
        write_step3_markdown()
    else:
        print("Usage: python3 helper.py [list-active|fill-inactive|compute-from-file <path>|cleanup-tmp|finalize|write-step3]")
        sys.exit(1)
```

**Главное про этот код:**
- `extract_status_transitions(changelog_list)` принимает значение поля **`changelogs`** напрямую — плоский список без `histories`
- Фильтрует **только** `field == 'status'` (игнорирует Link и другие)
- Для status-переходов читает поля **`fromString`/`toString`** (camelCase)
- `compute_from_file` — основной entry-point. Читает файл записанный агентом через WriteFile.
- НЕ читает stdin (раньше пытался — не работало для больших ответов)

## 8. Steps — что делает агент в чате

### Step 1. Получить список активных задач

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py list-active
```

helper выведет JSON-массив cr_keys активных задач. Например `["CRSIGMA-26516", "ASFC-58741", ...]`.

Сохранить этот список — итерируем по нему в Step 4.

### Step 2. Заполнить тривиальный timing для не-активных

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py fill-inactive
```

helper пройдёт по всем не-активным задачам, запишет `timing.computed = false`, нули.

### Step 3. Создать временную директорию

Через Shell:

```bash
mkdir -p pipeline/tmp
```

**Только** рабочая директория. **Не** `~/.gigacode/tmp/`.

### Step 4. Streaming-цикл по активным задачам

**Главная часть скилла.** Для каждой `cr_key` из списка активных (по одной, последовательно):

#### Step 4.1. Tool call с changelog

Нативный tool call агента:

```
Tool: jira_get_issue
key = <cr_key>
expand = "changelog"
fields = "summary,status,created,updated,resolutiondate"
```

#### Step 4.2. WriteFile ответа в файл

Сразу после получения ответа — через **встроенный WriteFile tool агента**:

```
Tool: WriteFile
path = "pipeline/tmp/<cr_key>.json"
content = <сырой JSON-ответ от jira_get_issue, без модификаций>
```

**Запрещено:**
- `echo '...' > file.json` через bash — длина команды и кавычки сломают
- `python3 -c "..."` с JSON в коде — то же
- Запись в `~/.gigacode/tmp/` — Filesystem Guard блокирует

**Только WriteFile tool + рабочая директория.**

#### Step 4.3. Compute через helper

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py compute-from-file pipeline/tmp/<cr_key>.json
```

helper.py:
- Читает JSON-файл с диска
- Извлекает `changelogs` (множественное число)
- Применяет `compute_timing` — строит timeline, агрегирует phase_days
- Записывает результат в `pipeline/enriched.json`
- Печатает короткую сводку: `✓ CRSIGMA-26516: A=87.0 R=0.0 T=0.0 (transitions=3)`

#### Step 4.4. Следующая задача

Прогресс пользователю каждые 3 задачи: "Обработано 3/24".

**КРИТИЧНО:** не накапливать tool-результаты в контексте. Строго: tool call → WriteFile → Shell-helper → следующая задача. После Step 4.3 JSON-ответ MCP **больше не нужен в контексте** — он на диске и обработан.

### Step 5. Финализация

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py finalize
```

Обновляет metadata, печатает JSON со статистикой.

### Step 6. Создать markdown-снимок

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py write-step3
```

### Step 7. Очистка tmp (опционально)

```bash
python3 ~/.gigacode/skills/timing-analyzer/helper.py cleanup-tmp
```

Удалит файлы из `pipeline/tmp/`. Для дебага можно пропустить.

### Step 8. Сводка пользователю

В чат:
- Активных задач с timing: N из M
- Без changelog: K
- Топ-3 самых долгих по факту
- Созданы файлы: `pipeline/enriched.json` (обновлён), `pipeline/step-3-after-timing-analyzer.md`
- Следующий шаг: запустите `report-builder`

## 9. КРИТИЧНО: архитектура передачи данных

### ПРАВИЛЬНО

```
Step 4.1. Tool call: jira_get_issue(key, expand="changelog", fields="...")
   → агент получает JSON в контекст

Step 4.2. Tool: WriteFile
   path = "pipeline/tmp/<cr_key>.json"
   content = <JSON-ответ>
   → JSON на диске в рабочей директории

Step 4.3. Shell:
   python3 ~/.gigacode/skills/timing-analyzer/helper.py compute-from-file pipeline/tmp/<cr_key>.json
   → helper прочитал, посчитал, обновил enriched.json
```

### ❌ НЕПРАВИЛЬНО

```python
# Нельзя вызвать MCP из Python
result = mcp__Atlassian__jira_get_issue(key="...")  # NameError
```

```bash
# Нельзя передавать большой JSON через echo — кавычки и длина команды
echo '{...огромный JSON...}' | python3 helper.py
```

```
# Нельзя писать в .gigacode/
WriteFile path="~/.gigacode/tmp/file.json"  # Filesystem Guard denied
```

```
# Нельзя копить ответы в контексте
Шаг 1: 24 tool calls подряд
Шаг 2: потом обрабатываем все
# Контекст переполнится после 5-7 ответов
```

## 10. Файлы

| Файл | Назначение |
|------|------------|
| `pipeline/enriched.json` | Читается, перезаписывается с полем `timing` |
| `pipeline/step-3-after-timing-analyzer.md` | Читаемый snapshot |
| `pipeline/tmp/<cr_key>.json` | Временные файлы с сырыми ответами MCP. Удаляются в Step 7. |
| `helper.py` | CLI с подкомандами |

### Запрещённые файлы

- `main.py`, `process.py`, `run_*.py`, `generate_*.py` — только helper.py
- **Конкретно запрещены:** `process_timing.py`, `run_timing.sh`, `batch_timing.json` — GigaCode создавал их в прошлых попытках
- `__pycache__`, `requirements.txt`, виртуальные окружения

## 11. Guardrails

- READ-ONLY для Jira (только `jira_get_issue` с `expand=changelog`)
- НЕ обрабатывать неактивные задачи через MCP — для них `fill-inactive`
- НЕ хранить сырой changelog в `enriched.json` — только агрегированные `phase_days`
- НЕ копить tool-ответы в контексте — streaming через файл
- НЕ изменять поле `task.jira`
- Запись JSON ТОЛЬКО через WriteFile в рабочую директорию

## 12. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| jira-enricher не отработал | helper упадёт на list-active, агент сообщит пользователю |
| 0 активных задач | fill-inactive заполняет всем тривиальный timing, переходим к finalize и step3 |
| `changelogs` отсутствует в ответе | `computed: false`, `reason: "no_changelog"` |
| `changelogs` есть, но без переходов status | `computed: false`, `reason: "no_status_transitions"` |
| `from_string` первого перехода пустой | helper использует `to_string` как initial_status |
| `resolutiondate` есть но статус не finished | helper использует resolved как end |
| Возвраты в один статус | Алгоритм суммирует — это правильно |
| Очень старая задача (798 дней) | changelog большой — нормально, файл на диске, не в контексте |
| Часовые пояса в timestamps | `parse_iso` учитывает |
| Задача создана в `In Progress` | from_string первого перехода пустой, обрабатывается |
| Timing уже посчитан (повторный запуск) | Перезаписывается |
| Файл `pipeline/tmp/<key>.json` существует от предыдущего прогона | WriteFile перезаписывает, или сначала cleanup-tmp |

## 13. Антипаттерны

### Критические

- **Вызывать MCP из Python** — NameError
- **Читать ключ `changelog`** (единственное число) — в Сбер-MCP **`changelogs`** (множественное)
- **Читать `from_string`/`to_string`** для status — в Сбер-MCP **`fromString`/`toString`** (camelCase) для `field='status'`
- **Передавать большой JSON через echo/stdin** — длина команды и кавычки ломают
- **Писать в `~/.gigacode/tmp/`** — Filesystem Guard блокирует
- **Накапливать tool-ответы в контексте перед обработкой** — переполнит при 24 задачах. Streaming!
- **Запросить changelog для всех 28 задач** — лишние ~20 запросов. Только active.
- **Использовать `fields="*"` + `expand=changelog`** — катастрофа по контексту
- **Создавать `main.py`, `process_timing.py`, `run_timing.sh`, `batch_timing.json`** или любые другие файлы. Только `helper.py`, `enriched.json`, `step-3.md`, `pipeline/tmp/<key>.json`.

### Обычные

- Параллельные tool calls
- Считать дни через `(d2 - d1).days` — отбрасывает дробную часть, нужно `total_seconds() / 86400`
- Игнорировать часовые пояса
- Не делать пауз между tool calls

## 14. Критерий успеха

После запуска:
1. `pipeline/enriched.json` валиден, у каждой задачи поле `timing` заполнено
2. У активных задач `timing.computed = true` с реальными числами в `phase_days`
3. У неактивных `timing.computed = false`, нули
4. `metadata.skills_completed` содержит `[..., "timing-analyzer"]`
5. Создан `pipeline/step-3-after-timing-analyzer.md`
6. **Никаких** лишних файлов в проекте

## 15. Что отложено

- v3.3: расчёт timing для **finished** задач
- v4: timing с разбивкой по sprint
- v6: расчёт загрузки команд во времени
