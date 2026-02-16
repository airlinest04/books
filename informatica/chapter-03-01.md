# §3.1 Определение источников (Sources)

В [§2.4](chapter-02-04.md) мы завершили главу о репозитории и метаданных. Теперь переходим к **источникам данных** (Sources): как определять структуру таблиц и файлов, из которых Integration Service будет извлекать данные. Source definition — это метаданные (имена колонок, типы данных, ограничения), а не сами данные. В этом разделе мы рассмотрим Source Analyzer, импорт из реляционных БД, импорт плоских файлов, типы источников и создание source вручную. Подробнее коннекторы и типы данных — в [§3.3](chapter-03-03.md), Flat File и БД — в [§3.4](chapter-03-04.md). См. [Глоссарий](glossary.md).

---

## 3.1.1. Source definition: что это и зачем

**Source definition** (определение источника) — объект репозитория, описывающий **структуру** данных-источника: имена колонок (портов), типы данных, точность и масштаб для числовых типов, ограничения (primary key, not null) при импорте из БД. Source не содержит сами данные — только метаданные, по которым Designer строит маппинг, а Integration Service формирует запросы при выполнении сессии. См. [Глоссарий](glossary.md).

Source definition нужен, чтобы:

- **Mapping Designer** знал, какие порты (поля) доступны от источника и как их связывать с трансформациями;
- **Source Qualifier** генерировал корректный SQL (по умолчанию `SELECT col1, col2, ... FROM source_table`);
- **Integration Service** при выполнении Session подключался к реальному источнику по Connection и читал данные по структуре, заданной в Source.

Без Source definition маппинг не может использовать источник. Source создаётся один раз и переиспользуется в нескольких маппингах; при изменении структуры таблицы в БД Source нужно обновить (re-import или ручное редактирование). Источник: [Creating Source Definitions](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-2/creating-source-definitions.html).

---

## 3.1.2. Source Analyzer

**Source Analyzer** — панель (tool) в Designer для создания и редактирования source definitions. Открывается через меню **Tools → Source Analyzer** или соответствующую вкладку. В Source Analyzer отображаются источники выбранной папки; здесь выполняют импорт из БД и из файлов. Источник: [Creating Source Definitions](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-2/creating-source-definitions.html).

В Navigator (дерево объектов) папка содержит узел **Sources**; под ним — импортированные источники. При импорте из БД создаётся узел **DBD** (Database Definition) с именем ODBC data source; под ним — список таблиц. Source definition отображается в виде диаграммы: прямоугольник с именем таблицы/файла и списком портов (колонок).

---

## 3.1.3. Импорт из реляционной БД

Импорт структуры таблиц из реляционной БД выполняется через **Sources → Import from Database**. Источник: [Importing Relational Source Definitions](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-sources/working-with-relational-sources/importing-relational-source-definitions.html).

Пошагово:

1. В Source Analyzer выбрать **Sources → Import from Database**.
2. Выбрать **ODBC data source** — источник данных ODBC, настроенный в системе (ODBC Administrator) для подключения к БД. При необходимости нажать Browse и создать или изменить data source.
3. Ввести **username** и **password** для подключения к БД. Требуются права на просмотр метаданных (например, SELECT на системные каталоги или на саму таблицу).
4. При необходимости указать **owner name** (владелец объектов) — для фильтрации списка таблиц, если пользователь имеет доступ к нескольким схемам.
5. Нажать **Connect**.
6. В списке объектов развернуть владельца и узел TABLES (или VIEWS, SYNONYMS в зависимости от типа объекта).
7. Выбрать одну или несколько таблиц (Shift — блок, Ctrl — отдельные). Можно выбрать папку и **Select All** для импорта всех таблиц папки.
8. Нажать **OK**.

Импортированные source definitions появляются в Source Analyzer и в Navigator под узлом Sources. Для Oracle владелец (owner) обычно совпадает с именем пользователя; имя может требоваться в верхнем регистре (например, JDOE). При импорте из БД Informatica получает метаданные через ODBC: имена колонок, типы, длину, precision/scale, primary key. PowerCenter подключается к реляционным БД через ODBC или нативные драйверы; PowerExchange не обязателен для стандартных СУБД. См. [§3.4](chapter-03-04.md).

---

## 3.1.4. Импорт плоских файлов

Импорт структуры плоского файла выполняется через **Sources → Import from File**. Источник: [Importing Delimited Flat Files](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-flat-files/importing-flat-files/importing-delimited-flat-files.html) (по веб-поиску).

Пошагово:

1. В Source Analyzer выбрать **Sources → Import from File**.
2. В диалоге **Open Flat File** выбрать файл на диске (или образец файла с нужной структурой).
3. Выбрать **code page** (кодировку), соответствующую данным в файле (например, UTF-8, Windows-1251).
4. Указать тип файла:
   - **Delimited** (разделённый) — колонки разделены символом (запятая, табуляция, точка с запятой); каждая строка — запись.
   - **Fixed-width** (фиксированная ширина) — колонки имеют фиксированные позиции и длину.
5. Для delimited: указать разделитель, qualifier (кавычки для строк), настройки.
6. В окне предпросмотра проверить разбор колонок; при необходимости скорректировать имена и типы.
7. Подтвердить импорт.

Source definition для плоского файла сохраняет имена колонок, типы, длину (precision) для строковых колонок. При выполнении Session путь к фактическому файлу задаётся в настройках Session (или через parameter `$InputFileName`); Source definition описывает только структуру. Подробнее форматы файлов — в [§3.4](chapter-03-04.md).

---

## 3.1.5. Типы источников

PowerCenter поддерживает несколько типов source definitions. Источник: [Objects Created in the Designer](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata/objects-created-in-the-designer.html).

| Тип | Описание |
|-----|----------|
| **Relational** | Таблицы, представления (views), синонимы (synonyms) реляционных БД. Импорт через Import from Database. |
| **Flat file** | Плоские файлы: delimited (CSV, TSV) или fixed-width. Импорт через Import from File. |
| **XML** | XML-файлы. Специфичный импорт с указанием схемы или примера файла. |
| **COBOL** | Файлы в формате COBOL (copybook). Импорт через соответствующий wizard. |

В большинстве ETL-сценариев используются **relational** (таблицы БД) и **flat file** (CSV, фиксированная ширина). XML и COBOL — для специализированных источников. Подробнее коннекторы и типы данных — в [§3.3](chapter-03-03.md).

---

## 3.1.6. Создание source вручную

При необходимости source можно создать **вручную** без импорта: в Source Analyzer **Sources → Create** (или аналог по версии). Задаются имя источника, порты (колонки) с именами и типами. Ручное создание полезно, когда:

- нет доступа к БД на этапе разработки;
- структура известна, но отличается от текущей таблицы (например, подмножество колонок);
- источник — файл с нестандартной структурой, которую проще описать вручную.

При ручном создании важно корректно задать типы данных и длину, иначе при выполнении возможны ошибки преобразования или усечение данных.

---

## 3.1.7. Source и Connection: разделение ответственности

**Source definition** (в Designer) и **Connection** (в Workflow Manager) — разные объекты:

| Объект | Где создаётся | Что хранит |
|--------|---------------|------------|
| **Source** | Designer (Source Analyzer) | Структура: имена колонок, типы данных. Метаданные для маппинга. |
| **Connection** | Workflow Manager | Параметры доступа: хост, порт, база, пользователь, пароль. Используется Integration Service при выполнении. |

Source не содержит сведений о том, *где* находится таблица (какой сервер, какая БД). Connection не знает о структуре таблицы. В Session указывается: для реляционного источника — Connection; структура берётся из Source в маппинге. Для плоского файла Connection не нужен (тип None); путь к файлу задаётся в Session. См. [§2.3](chapter-02-03.md), [§1.4](chapter-01-04.md).

---

## 3.1.8. Обновление source при изменении структуры

Если структура таблицы в БД изменилась (добавлены колонки, изменены типы), source definition нужно обновить. Варианты:

- **Re-import:** снова выполнить Import from Database для той же таблицы; при конфликте имён выбрать Replace. Существующий source будет перезаписан новой структурой.
- **Ручное редактирование:** добавить или изменить порты в Source Analyzer. Подходит для небольших изменений.

После обновления source маппинги, использующие этот source, могут перейти в состояние **Impacted** или **Invalid**, если связи портов нарушены (например, удалена колонка, на которую ссылается Expression). Нужно обновить маппинги. См. [§1.3.3](chapter-01-03.md).

---

## 3.1.9. ODBC data source: предварительная настройка

Для импорта из БД требуется **ODBC data source** (DSN), настроенный в ОС. На Windows — через ODBC Data Source Administrator (odbcad32); на Linux — через файлы `odbc.ini` и `odbcinst.ini`. Data source указывает драйвер (Oracle, SQL Server, PostgreSQL и т.д.), хост, порт, имя базы. Designer при Import from Database подключается к БД через этот DSN для чтения метаданных.

Важно: ODBC data source используется **только при импорте** в Designer. При выполнении Session Integration Service использует **Connection object** из Workflow Manager, который может указывать на другой сервер или базу (например, production вместо dev). DSN и Connection — разные сущности. См. [Configuring a Third-Party ODBC Data Source](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-sources/working-with-relational-sources/configuring-a-third-party-odbc-data-source.html).

---

## 3.1.10. Типичные ошибки при определении источников

- **Путать Source и Connection:** Source — структура; Connection — параметры доступа. Оба нужны для реляционного источника при выполнении.
- **Импорт из неверной схемы:** при нескольких схемах в БД указать правильного owner; иначе можно импортировать таблицу с тем же именем из другой схемы.
- **Неверная кодировка для flat file:** при импорте файла выбрать code page, соответствующую данным; иначе кириллица или спецсимволы отобразятся некорректно.
- **Использовать production-БД для импорта в dev:** Designer подключается к БД при импорте; для разработки лучше использовать dev-копию или снимок структуры, чтобы не нагружать production.
- **Забыть обновить source после изменения таблицы:** маппинг будет ссылаться на старую структуру; при выполнении возможны ошибки или потеря данных. Регулярно re-import при изменении схемы.

---

## Ключевое

- **Source definition** — метаданные структуры источника (колонки, типы); не содержит данных. Создаётся в Source Analyzer, хранится в папке репозитория.
- **Source Analyzer** — инструмент Designer для импорта и редактирования sources. Меню: Sources → Import from Database, Sources → Import from File.
- **Импорт из БД:** ODBC data source, username/password, owner, выбор таблиц. Поддерживаются таблицы, views, synonyms.
- **Импорт плоских файлов:** Import from File, выбор файла, code page, delimited или fixed-width, настройка разделителей.
- **Типы источников:** Relational, Flat file, XML, COBOL. Наиболее частые — relational и flat file.
- **Source vs Connection:** Source — структура (Designer); Connection — параметры доступа (Workflow Manager). При выполнении Session использует Connection; структура — из Source в маппинге.
- **Обновление:** при изменении таблицы в БД — re-import или ручное редактирование source; проверить маппинги на Impacted/Invalid.

В [§3.2](chapter-03-02.md) мы разберём определение приёмников (Targets): Target Designer, импорт и создание таблиц и файлов.
