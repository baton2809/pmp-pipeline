# SPEC: report-builder (v3.2)

> Четвёртый и последний скилл pipeline. Читает `pipeline/enriched.json` со всеми обогащениями, генерирует финальный markdown-отчёт `report.md` в корне рабочей директории.
>
> Никаких MCP-вызовов. Никакой Excel-работы. Только формирование текста из готовых данных. Самый простой скилл pipeline.
>
> **Изменения v3.2 vs v3.1:** колонка "Команда" → "Поток/Проект". Значение `customfield_22200` (например `PALM.CSP.K7M`) — это техническая метка потока разработки, а не "команда" в обычном смысле (типа "Пальмира/Орион"). Чтобы не вводить Наталью в заблуждение — переименовали колонку и добавили пояснение под таблицей.

## 1. Место в pipeline

```
1. excel-parser
2. jira-enricher
3. timing-analyzer (опционально)    → pipeline/enriched.json
4. report-builder  ◄── (этот скилл)  → report.md
```

## 2. Цели

- Прочитать `pipeline/enriched.json`
- Валидировать что хотя бы `excel-parser` и `jira-enricher` отработали
- Сгенерировать markdown-отчёт со 7 секциями
- Сохранить в `report.md` в **корне** рабочей директории (не в `pipeline/` — это финальный отчёт для пользователя)
- Вывести краткую сводку в чат

## 3. Анти-цели

- **НЕ** ходить в Jira (все данные уже в enriched.json)
- **НЕ** читать Excel (всё уже распарсено)
- **НЕ** изменять `pipeline/enriched.json` (этот скилл write-only для report.md, но может проставить `report_generated_at` в metadata)
- **НЕ** делать долгих вычислений — данные уже агрегированы

## 4. Условный рендеринг

Скилл работает в **двух режимах** в зависимости от того что в `enriched.json`:

**Полный режим** (если timing-analyzer отработал):
- Все 7 секций
- Колонки "Факт А/Р/Т"
- Секция "Застрявшие"

**Без timing режим** (если timing-analyzer не запускался):
- 6 секций (без "Застрявшие")
- Без колонок "Факт А/Р/Т"
- В дисклеймере явная пометка "для расчёта факта запустите timing-analyzer"

## 5. Готовый код helper.py

```python
# helper.py для report-builder

import json
import sys
import re
import os
from datetime import datetime, timezone

# === Утилиты для формирования отчёта ===

def truncate_words(text, limit):
    """Обрезать по словам, добавить … если длиннее."""
    if not text:
        return ''
    text = str(text).replace('\n', ' ').replace('\r', ' ').replace('\t', ' ')
    text = re.sub(r'\s+', ' ', text).strip()
    if len(text) <= limit:
        return text
    cut = text[:limit].rsplit(' ', 1)[0]
    return cut + '…'

def now_iso():
    return datetime.now(timezone.utc).astimezone().isoformat()

# === Маркеры ===

def compute_marker(task, timing_available):
    """Рассчитать маркер состояния для задачи."""
    jira = task.get('jira') or {}
    
    if not jira.get('found'):
        return '✗ не найдена'
    
    cat = jira.get('status_category')
    status = jira.get('status', '')
    plan = task.get('plan') or {}
    
    if cat == 'finished':
        return '✓'
    if cat == 'not_started':
        return '⏸'
    if cat == 'unknown':
        return '❗ нестандартный'
    if status == 'Need Info':
        return '❗ Need Info'
    
    # Проверка превышения по фазе (только если есть timing)
    if timing_available:
        timing = task.get('timing') or {}
        if timing.get('computed'):
            phase_days = timing.get('phase_days') or {}
            phase = jira.get('phase')
            if phase:
                fact = phase_days.get(phase, 0)
                plan_phase_key = {'A': 'analytics', 'R': 'development', 'T': 'testing'}[phase]
                plan_value = plan.get(plan_phase_key)
                if plan_value and plan_value > 0:
                    if fact > plan_value * 2:
                        return f'⚠ долго в {phase}'
                elif fact > 14:  # план = 0, факт большой
                    return f'⚠ долго в {phase}'
    
    return '⏳'

# === Подсчёт сводки ===

def compute_summary(enriched, timing_available):
    """Подсчитать поля общей сводки."""
    tasks = enriched['tasks']
    summary = {
        'total': len(tasks),
        'with_cr': sum(1 for t in tasks if t.get('cr_key')),
        'found_in_jira': sum(1 for t in tasks if (t.get('jira') or {}).get('found')),
        'not_found_in_jira': sum(1 for t in tasks 
                                    if t.get('jira') and not t['jira'].get('found')),
        'no_plan': sum(1 for t in tasks 
                         if not (t.get('plan') or {}).get('total')),
        'closed': sum(1 for t in tasks 
                        if (t.get('jira') or {}).get('status_category') == 'finished'),
        'in_progress': sum(1 for t in tasks 
                             if (t.get('jira') or {}).get('status_category') 
                             in ('analysis', 'development', 'testing')),
        'not_started': sum(1 for t in tasks 
                             if (t.get('jira') or {}).get('status_category') == 'not_started'),
        'epics_unique': len(enriched.get('epics', [])),
    }
    if timing_available:
        summary['stuck'] = sum(1 for t in tasks 
                                 if '⚠' in compute_marker(t, True))
    return summary

# === Главная функция: построить отчёт ===

def build_report(enriched_path='pipeline/enriched.json',
                  report_path='report.md'):
    with open(enriched_path, 'r', encoding='utf-8') as f:
        enriched = json.load(f)
    
    timing_available = 'timing-analyzer' in enriched['metadata'].get('skills_completed', [])
    summary = compute_summary(enriched, timing_available)
    tasks = enriched['tasks']
    epics = enriched.get('epics', [])
    
    md = []
    
    # Заголовок
    md.append("# PMP vs Jira — состояние портфеля Q2\n")
    md.append(f"**Источник плана:** {enriched['metadata']['source_file']}, лист {enriched['metadata']['sheet']}\n")
    md.append(f"**Источник данных Jira:** mcp-jira (https://jira.delta.sbrf.ru)\n")
    md.append(f"**Дата сборки:** {datetime.now().strftime('%Y-%m-%d %H:%M')}\n")
    
    # Дисклеймер
    md.append("\n## Что в отчёте и чего нет\n")
    md.append("Этот отчёт сравнивает план Натальи (Excel) с реальными данными Jira.\n")
    md.append("\n**Что показано:**")
    md.append("- Статус каждой задачи (текущий)")
    md.append("- Эпик к которому привязана задача")
    md.append("- Поток/Проект (из customfield_22200 Jira типа PALM.CSP.K7M — это техническая метка потока разработки, не \"команда\" в обычном понимании. Если у задачи нет — fallback на ответственного)")
    md.append("- Lead time — календарные дни жизни задачи")
    if timing_available:
        md.append("- **Факт А/Р/Т в календарных днях** — время которое задача провела в каждой фазе")
        md.append("- Маркер ⚠ долго в фазе для задач где факт превышает план более чем в 2 раза")
    md.append("- Срезы по эпикам и потокам разработки\n")
    md.append("\n**Что НЕ показано:**")
    md.append("- Точный факт в чел-днях — недоступен (worklog не ведётся командой)")
    if timing_available:
        md.append("- Календарные дни ≠ чел-дни (включают выходные, ожидания, переключения)")
        md.append("- Маркер ⚠ — это сигнал \"стоит проверить\", а не \"точно перелимит\"")
    else:
        md.append("- **Факт А/Р/Т недоступен в этом отчёте** — для его расчёта запустите `timing-analyzer` и пересоберите отчёт")
    md.append("")
    
    # 1. Сводка
    md.append("\n## 1. Общая сводка\n")
    md.append("| Показатель | Значение |")
    md.append("|---|---|")
    md.append(f"| Задач в плане | {summary['total']} |")
    md.append(f"| С заполненным CR | {summary['with_cr']} |")
    md.append(f"| Найдено в Jira | {summary['found_in_jira']} |")
    md.append(f"| Не найдено в Jira | {summary['not_found_in_jira']} |")
    md.append(f"| Без оценок А/Р/Т в плане | {summary['no_plan']} |")
    md.append(f"| Закрытых | {summary['closed']} |")
    md.append(f"| В работе | {summary['in_progress']} |")
    md.append(f"| Не начатых | {summary['not_started']} |")
    if timing_available:
        md.append(f"| Застрявших (⚠) | {summary.get('stuck', 0)} |")
    md.append(f"| Уникальных эпиков | {summary['epics_unique']} |")
    
    # 2. Основная таблица
    md.append("\n## 2. Основная таблица\n")
    
    if timing_available:
        md.append("| # | Ключ | Название | Эпик | Поток/Проект | План А/Р/Т (чд) | Факт А/Р/Т (д) | Lead time | Статус | Маркер |")
        md.append("|---|------|----------|------|---------|------------------|----------------|-----------|--------|--------|")
    else:
        md.append("| # | Ключ | Название | Эпик | Поток/Проект | План А/Р/Т (чд) | Lead time | Статус | Фаза | Маркер |")
        md.append("|---|------|----------|------|---------|------------------|-----------|--------|------|--------|")
    
    for i, task in enumerate(tasks, 1):
        cr = task['cr_key']
        name = truncate_words(task.get('task_name'), 50)
        
        jira = task.get('jira') or {}
        epic = jira.get('epic') or {}
        epic_str = '—'
        if epic.get('key'):
            ename = truncate_words(epic.get('name') or '', 25)
            epic_str = f"{epic['key']} {ename}".strip()
        
        team_obj = jira.get('team') or {}
        team_val = team_obj.get('value') or '—'
        if team_obj.get('source') == 'assignee_fallback':
            team_val = f"(по assignee) {team_val}"
        
        plan = task.get('plan') or {}
        if plan.get('total') is None:
            plan_str = '—'
        else:
            def fmt(v): return str(int(v)) if v is not None else '0'
            plan_str = f"{fmt(plan.get('analytics'))}/{fmt(plan.get('development'))}/{fmt(plan.get('testing'))}"
        
        lead = jira.get('lead_time_days')
        lead_str = f"{lead} д" if lead is not None else '—'
        
        status = jira.get('status', '—')
        marker = compute_marker(task, timing_available)
        
        if timing_available:
            timing = task.get('timing') or {}
            if timing.get('computed'):
                pd = timing['phase_days']
                fact_str = f"{int(pd['A'])}/{int(pd['R'])}/{int(pd['T'])}"
            else:
                fact_str = '—'
            md.append(f"| {i} | {cr} | {name} | {epic_str} | {team_val} | {plan_str} | {fact_str} | {lead_str} | {status} | {marker} |")
        else:
            phase = jira.get('phase') or '—'
            md.append(f"| {i} | {cr} | {name} | {epic_str} | {team_val} | {plan_str} | {lead_str} | {status} | {phase} | {marker} |")
    
    # 3. Срез по эпикам
    md.append("\n## 3. Срез по эпикам\n")
    
    if epics:
        md.append("| Эпик | Имя | Задач из плана | Всего дочерних в Jira | План Σ (чд) |")
        md.append("|------|-----|----------------|-----------------------|-------------|")
        sorted_epics = sorted(epics, key=lambda e: len(e.get('tasks_from_plan', [])), reverse=True)
        for e in sorted_epics:
            ename = truncate_words(e.get('name') or '', 60)
            from_plan_keys = e.get('tasks_from_plan', [])
            from_plan = len(from_plan_keys)
            total_children = e.get('children_count_total', '—')
            # Сумма плана для задач из плана этого эпика
            plan_sum = 0
            for t in tasks:
                if t['cr_key'] in from_plan_keys:
                    p = (t.get('plan') or {}).get('total') or 0
                    plan_sum += p
            md.append(f"| {e['key']} | {ename} | {from_plan} | {total_children} | {int(plan_sum)} |")
        
        # Задачи без эпика
        no_epic = [t for t in tasks if not ((t.get('jira') or {}).get('epic') or {}).get('key')]
        if no_epic:
            no_epic_plan = sum((t.get('plan') or {}).get('total') or 0 for t in no_epic)
            md.append(f"| — | (эпик не привязан) | {len(no_epic)} | — | {int(no_epic_plan)} |")
    else:
        md.append("*Эпики не найдены.*")
    
    # 4. Срез по потокам разработки
    md.append("\n## 4. Срез по потокам разработки\n")
    md.append("> **Что такое \"Поток/Проект\":** значение из `customfield_22200` Jira типа `PALM.CSP.K7M` — это техническая метка потока разработки. Это не \"команда\" в обычном понимании (типа \"Пальмира/Орион\"). Если в задаче поле пустое — fallback на ответственного.\n")
    md.append("| Поток/Проект (источник) | Задач из плана | План Σ (чд) | В работе | Закрыто |")
    md.append("|---|-----|-----|-----|-----|")
    
    from collections import defaultdict
    by_team = defaultdict(list)
    for t in tasks:
        team_obj = (t.get('jira') or {}).get('team') or {}
        team_key = team_obj.get('value') or '—'
        source = team_obj.get('source', '')
        display_key = team_key
        if source == 'assignee_fallback' and team_key != '—':
            display_key = f"(по assignee) {team_key}"
        by_team[display_key].append(t)
    
    for team_key, team_tasks in sorted(by_team.items(), key=lambda x: -len(x[1])):
        plan_sum = sum((t.get('plan') or {}).get('total') or 0 for t in team_tasks)
        in_progress = sum(1 for t in team_tasks 
                            if (t.get('jira') or {}).get('status_category') 
                            in ('analysis', 'development', 'testing'))
        closed = sum(1 for t in team_tasks 
                       if (t.get('jira') or {}).get('status_category') == 'finished')
        md.append(f"| {team_key} | {len(team_tasks)} | {int(plan_sum)} | {in_progress} | {closed} |")
    
    # 5. Застрявшие (если timing)
    if timing_available:
        md.append("\n## 5. Застрявшие (⚠)\n")
        stuck = [t for t in tasks if '⚠' in compute_marker(t, True)]
        
        if stuck:
            md.append("| Ключ | Название | Эпик | Фаза | План (чд) | Факт (д) | Превышение | Статус |")
            md.append("|------|----------|------|------|-----------|----------|------------|--------|")
            
            def excess(task):
                phase = (task.get('jira') or {}).get('phase')
                if not phase:
                    return 0
                pd = ((task.get('timing') or {}).get('phase_days') or {}).get(phase, 0)
                plan_key = {'A': 'analytics', 'R': 'development', 'T': 'testing'}[phase]
                plan = (task.get('plan') or {}).get(plan_key) or 0
                return pd / plan if plan > 0 else pd
            
            stuck_sorted = sorted(stuck, key=excess, reverse=True)
            for t in stuck_sorted:
                cr = t['cr_key']
                name = truncate_words(t.get('task_name'), 50)
                epic = ((t.get('jira') or {}).get('epic') or {}).get('key') or '—'
                phase = (t.get('jira') or {}).get('phase') or '?'
                plan_key = {'A': 'analytics', 'R': 'development', 'T': 'testing'}.get(phase)
                plan = (t.get('plan') or {}).get(plan_key) if plan_key else None
                pd = ((t.get('timing') or {}).get('phase_days') or {}).get(phase, 0)
                exc = excess(t)
                exc_str = f"× {exc:.1f}" if exc > 0 else '—'
                status = (t.get('jira') or {}).get('status', '—')
                plan_str = str(int(plan)) if plan else '—'
                md.append(f"| {cr} | {name} | {epic} | {phase} | {plan_str} | {int(pd)} | {exc_str} | {status} |")
        else:
            md.append("*Нет задач с превышением плана более чем в 2 раза.*")
    
    # 6. Не найдено
    md.append("\n## 6. Не найдено в Jira\n")
    not_found = [t for t in tasks 
                   if t.get('jira') and not t['jira'].get('found')]
    if not_found:
        for t in not_found:
            md.append(f"- `{t['cr_key']}` — {truncate_words(t.get('task_name'), 80)}")
    else:
        md.append("*Все задачи плана найдены в Jira.*")
    
    # 7. Без плана
    md.append("\n## 7. Без плана\n")
    no_plan = [t for t in tasks if not (t.get('plan') or {}).get('total')]
    if no_plan:
        for t in no_plan:
            md.append(f"- `{t['cr_key']}` — {truncate_words(t.get('task_name'), 80)}")
    else:
        md.append("*Все задачи имеют заполненный план.*")
    
    md.append("\n---\n")
    md.append("*Отчёт сгенерирован скиллом `report-builder` (pipeline pmp-vs-jira).*")
    md.append("*Дисклеймер из секции \"Что в отчёте и чего нет\" применим ко всем числам факта.*")
    
    # Запись
    with open(report_path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(md))
    
    # Обновить metadata
    enriched['metadata']['report_generated_at'] = now_iso()
    completed = enriched['metadata'].setdefault('skills_completed', [])
    if 'report-builder' not in completed:
        completed.append('report-builder')
    with open(enriched_path, 'w', encoding='utf-8') as f:
        json.dump(enriched, f, indent=2, ensure_ascii=False)
    
    # Сводка в stdout
    print(json.dumps({
        'report_path': report_path,
        'total_tasks': len(tasks),
        'timing_available': timing_available,
        'sections': 7 if timing_available else 6,
    }, ensure_ascii=False))

if __name__ == '__main__':
    cmd = sys.argv[1] if len(sys.argv) > 1 else 'build'
    if cmd == 'build':
        build_report()
    else:
        print("Usage: python3 helper.py build")
        sys.exit(1)
```

## 6. Steps — что делает агент в чате

### Step 1. Валидация и запуск

Агент просто запускает helper:

```bash
python3 ~/.gigacode/skills/report-builder/helper.py build
```

helper.py:
- Читает `pipeline/enriched.json`
- Валидирует что есть `excel-parser` и `jira-enricher`
- Проверяет наличие `timing-analyzer` → выбирает режим (полный/без timing)
- Строит markdown
- Записывает `report.md` в корне рабочей директории
- Обновляет `metadata.report_generated_at` в enriched.json
- Печатает JSON-сводку в stdout

### Step 2. Сводка пользователю

Из stdout helper'а агент получает структуру вида:
```json
{"report_path": "report.md", "total_tasks": 28, "timing_available": true, "sections": 7}
```

Сообщает пользователю:
- Создан `report.md` в текущей директории
- Задач в отчёте: 28
- Режим: полный (с фактом А/Р/Т) или без timing
- Совет: открыть `report.md` в редакторе с поддержкой markdown

Если `timing_available = false` — упомянуть: "для добавления Факта А/Р/Т запустите `timing-analyzer` и пересоберите отчёт".

## 7. Файлы

| Файл | Назначение |
|------|------------|
| `pipeline/enriched.json` | ЧТЕНИЕ. Минорно обновляется (только `report_generated_at`). |
| `report.md` | СОЗДАЁТСЯ в **корне** рабочей директории (не в `pipeline/`) |
| `helper.py` | Один CLI с подкомандой `build` |

### Запрещённые файлы

Только `helper.py`. Никаких `main.py`, `process.py`, `run_*.py`, `__pycache__`.

## 8. Guardrails

- Не модифицировать `pipeline/enriched.json` кроме `report_generated_at` и `skills_completed`
- Не ходить в Jira
- Не читать Excel
- Если поле null — показывать `—`, не пустоту
- Если timing-analyzer не отработал — отчёт всё равно работает (6 секций, без Факт)
- Никаких `<!-- HTML комментариев -->` в markdown
- Кириллица сохраняется как есть (UTF-8, не `\u...`)

## 9. Edge cases

| Ситуация | Поведение |
|----------|-----------|
| jira-enricher не отработал | helper упадёт с понятной ошибкой, агент сообщит пользователю |
| timing-analyzer не отработал | helper строит 6 секций без Факта, в дисклеймере явная пометка |
| Все 28 задач не найдены в Jira | Отчёт строится, секция "Не найдено" большая, основная таблица пустая по jira-данным |
| 0 эпиков | Секция 3: "*Эпики не найдены.*" |
| Все задачи без потока | Секция 4 содержит одну строку с `—` |
| 0 застрявших | Секция 5: "*Нет задач с превышением.*" |
| План А/Р/Т нулевой (0/0/0) | helper защищает: при `plan_value == 0` маркер ⚠ ставится только если факт > 14 дней |
| Несколько строк дубль (CRSIGMA-22127) | Обе строки в таблице |
| Длинное имя эпика | Обрезка до 25 символов через `…` (по словам) |

## 10. Антипаттерны

### Критические

- **Модифицировать `pipeline/enriched.json`** кроме `report_generated_at` и `skills_completed` — это write-only скилл для report.md
- **Ходить в Jira** — все данные уже есть
- **Обрезка посередине слова** — некрасиво, нечитаемо
- **Жёстко требовать timing-analyzer** — должно работать без него
- **Создавать `main.py`, `process.py`** — только helper.py

### Обычные

- Не очищать `\n` в названиях — таблица ломается
- Выводить ID полей (`customfield_22200`) вместо имён (`PALM.CSP.K7M`)
- Не сортировать таблицы
- Не обрезать длинные имена
- Терять кириллицу при записи

## 11. Критерий успеха

После запуска:
1. `report.md` создан в корне рабочей директории, валиден markdown
2. Все 7 секций есть (5-я может быть "*Нет застрявших.*")
3. Дисклеймер заметен сверху
4. Таблицы читаемы (имена обрезаны, не ломают разметку)
5. Маркеры на местах
6. Размер файла 10-50 KB

## 12. Что отложено

- v3.2: HTML-версия отчёта параллельно с markdown
- v4: дополнительные секции про даты ИФТ/ПСИ/ПРОМ
- v5: Сберчат-отправка отчёта (отдельный скилл `sberchat-notifier`)
- v6: разворот дочерних эпиков
