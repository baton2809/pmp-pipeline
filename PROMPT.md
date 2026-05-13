# PROMPT для GigaCode CLI — excel-parser (v3.1)

```
Прочитай SPEC.md и CONTRACT.md в корне репозитория. excel-parser — первый из 4 скиллов pipeline pmp-vs-jira.

Сгенерируй РОВНО два файла:

1. skill/excel-parser/SKILL.md  — главная инструкция для агента
2. skill/excel-parser/helper.py — вспомогательные Python-функции

Что делает скилл:
- Читает Бэклог и цели.xlsx, лист Q2_26_оценки_new_name (имена жёстко зафиксированы)
- Извлекает задачи через поиск колонок ПО ИМЕНИ ЗАГОЛОВКА (case-insensitive подстрока)
- Создаёт pipeline/enriched.json по структуре из CONTRACT.md секция "После excel-parser"
- Создаёт pipeline/step-1-after-excel-parser.md — читаемый снимок
- НЕ ходит в Jira

Критически важно:

1. ИСПОЛЬЗОВАТЬ ТОЛЬКО openpyxl. НЕ парсить xlsx как XML вручную. В предыдущей попытке был ручной XML-парсинг — это привело к сдвигу колонок и из 28 задач нашлось 3. См. раздел 11 SPEC.md "Антипаттерны".

2. ИСКАТЬ КОЛОНКИ ПО ИМЕНИ ЗАГОЛОВКА (case-insensitive подстрока), не по фиксированной букве (B, C, F, G...). См. раздел 5 SPEC.md.

3. РАЗДЕЛ 7 SPEC.md УЖЕ СОДЕРЖИТ ГОТОВЫЙ КОД helper.py. Использовать его как основу. Все функции (open_workbook, find_header_row, detect_columns, parse_cr_key, extract_plan, clean_text, save_enriched, write_step1_markdown, now_iso) — уже прописаны с правильной реализацией. Не изобретать заново.

4. Папка для промежуточных файлов называется pipeline/ (БЕЗ точки в начале). Это видимая папка в Linux/Mac, в отличие от .cache/. См. CONTRACT.md.

5. Создаётся ДВА файла на выходе:
   - pipeline/enriched.json (для следующих скиллов)
   - pipeline/step-1-after-excel-parser.md (читаемый снимок для пользователя)

6. Раздел 8 SPEC.md (формат SKILL.md): SKILL.md — это инструкция для агента в чате, не Python-программа. Шаги на естественном языке. helper.py — это набор функций которые SKILL.md просит вызвать.

7. Раздел 11 SPEC.md (антипаттерны): ТОЛЬКО helper.py. Никаких main.py, process.py, run_*.py, generate_*.py, __pycache__, requirements.txt, виртуальных окружений. В прошлой v3 GigaCode создал 4 файла и сломал pipeline — не повторять.

8. parse_cr_key использует re.SEARCH (не match, не fullmatch). См. раздел 5 SPEC.md.

9. Колонки 'аналитика', 'разработка', 'тестирование' встречаются в файле ДВАЖДЫ — это две версии плана. extract_plan берёт ПОСЛЕДНЕЕ непустое значение для каждой фазы.

10. CONTRACT.md секция "После excel-parser" — структура json строго оттуда.

11. Никаких MCP-вызовов. Этот скилл с Jira не общается.

Когда сгенерируешь — выведи в чат:
- Структуру созданных файлов (ровно 2)
- Список функций в helper.py
- Один пример Step из SKILL.md
- Подтверди что используется только openpyxl
- Подтверди что поиск колонок по имени, не по букве
- Подтверди что используется re.search
- Подтверди что создаются два файла: enriched.json И step-1-after-excel-parser.md
```

## Ожидаемый результат прогона

После запуска `excel-parser` на реальном файле Натальи:
- Извлечено задач: **28**
- Создан `pipeline/enriched.json`
- Создан `pipeline/step-1-after-excel-parser.md`
- 27 уникальных CR + 1 дубликат
- Распределение: 17 ASFC + 10 CRSIGMA + 1 TIBDS
