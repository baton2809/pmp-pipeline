# PROMPT для GigaCode CLI — excel-parser

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория целиком. excel-parser — первый из 4 скиллов pipeline pmp-vs-jira.

Сгенерируй два файла:

1. skill/excel-parser/SKILL.md  — главная инструкция для агента
2. skill/excel-parser/helper.py — вспомогательные Python-функции

Что делает скилл:
- Читает Бэклог и цели.xlsx, лист Q2_26_оценки_new_name (жёстко зафиксированы)
- Извлекает задачи по позициям колонок B, C, F, G, J/K/L (v1), R/S/T (v2)
- Создаёт .cache/enriched.json по структуре из CONTRACT.md секция "После excel-parser"
- НЕ ходит в Jira

Критически важно:

1. Раздел 8 SPEC.md (формат SKILL.md): SKILL.md — это инструкция для агента в чате, не Python-программа. Шаги на естественном языке. helper.py — это набор функций которые SKILL.md просит вызвать.

2. Раздел 7 SPEC.md и раздел 11 (антипаттерны): ТОЛЬКО один Python-файл с именем helper.py. Никаких main.py, process.py, run_*.py, generate_*.py, __pycache__, requirements.txt. В прошлой v3 GigaCode создал 4 файла и сломал pipeline.

3. Раздел 5 SPEC.md: жёстко зафиксированные колонки B/C/F/G/J/K/L/R/S/T. Не искать по имени заголовка.

4. CONTRACT.md секция "После excel-parser" — структура json строго оттуда. Не выдумывать поля.

5. Никаких MCP-вызовов. Этот скилл с Jira не общается.

Когда сгенерируешь — выведи в чат:
- Структуру созданных файлов
- Список функций в helper.py
- Один пример Step из SKILL.md (чтобы было видно что это инструкция, не код)
- Проверь что нет лишних .py файлов
```
