# §3.4 Flat File и базы данных

В [§3.1](chapter-03-01.md)–[§3.3](chapter-03-03.md) мы рассмотрели определение источников и приёмников, коннекторы и типы данных. В этом разделе — практические детали работы с **flat file** (CSV, delimited, fixed-width) и с основными реляционными БД (Oracle, SQL Server, PostgreSQL и др.): настройки импорта, connect strings, особенности типов и подключения. См. [Глоссарий](glossary.md).

---

## 3.4.1. Flat file: delimited и fixed-width

**Flat file** (плоский файл) — текстовый файл с данными в виде строк и колонок. PowerCenter поддерживает два формата. Источник: [Importing Delimited Flat Files](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-flat-files/importing-flat-files/importing-delimited-flat-files.html), [Importing Fixed-Width Flat Files](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-flat-files/importing-flat-files/importing-fixed-width-flat-files.html).

| Формат | Ориентация | Описание |
|--------|------------|----------|
| **Delimited** (разделённый) | Character-oriented, line sequential | Колонки разделены символом (запятая, табуляция, точка с запятой). Длина колонок переменная. Типичные форматы: CSV, TSV. |
| **Fixed-width** (фиксированная ширина) | Byte-oriented | Каждая колонка занимает фиксированное число байт. Позиции колонок задаются границами (column breaks). |

Оба формата — line sequential: каждая строка заканчивается символом новой строки. Не поддерживаются бинарные данные и multibyte-символы более 2 байт на символ (ограничение для импорта). При размере файла > 256 KB или строке > 16 KB — проверять корректность precision после импорта.

---

## 3.4.2. Импорт delimited flat file (CSV, TSV)

Импорт delimited-файла: **Sources → Import from File** (Source Analyzer) или **Targets → Import from File** (Target Designer). Открывается **Flat File Wizard**. Источник: [Importing Delimited Flat Files](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-flat-files/importing-flat-files/importing-delimited-flat-files.html).

**Шаг 1:**
- **Flat File Type:** Delimited.
- **Enter a Name for This Source:** имя source/target в репозитории.
- **Start Import At Row:** с какой строки начинать чтение (например, 2 — пропустить заголовок).
- **Import Field Names From First Line:** использовать первую строку как имена колонок. Некорректные имена получают префикс `FIELD_`.

**Шаг 2:**
- **Delimiters:** символ-разделитель (запятая, табуляция, точка с запятой; Other — произвольный). Должен быть печатаемым, отличным от escape и quote.
- **Treat Consecutive Delimiters as One:** несколько подряд идущих разделителей считать одним (иначе — NULL между ними).
- **Escape Character:** символ экранирования перед разделителем или кавычкой внутри строки.
- **Remove Escape Character From Data:** по умолчанию включено — escape не попадает в данные.
- **Text Qualifier:** No Quote, Single Quote или Double Quotes — границы строки; разделители внутри кавычек игнорируются.

**Шаг 3:**
- Для каждой колонки: **Name**, **Datatype** (Text, Numeric, Datetime), **Length/Precision**, **Scale**, **Width**.
- Для Text: Precision — максимальное число символов; для delimited Wizard игнорирует precision при чтении.
- Для Numeric: Precision — число значащих цифр; только символы 0–9 считаются числовыми.

---

## 3.4.3. Импорт fixed-width flat file

Импорт fixed-width: **Sources/Targets → Import from File**, в Wizard выбрать **Fixed Width**. Источник: [Importing Fixed-Width Flat Files](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-flat-files/importing-flat-files/importing-fixed-width-flat-files.html).

**Особенности:**
- Длины полей задаются **в байтах**.
- **Column breaks** — границы колонок в окне предпросмотра; можно создавать, перемещать (drag), удалять (double-click).
- Для shift-sensitive файлов: single-byte shift — `'.'`, double-byte — `'..'` в предпросмотре.
- **Precision** для text — в байтах; для numeric — число значащих цифр, **Width** — байт для чтения/записи.
- Неверное положение границ при multibyte-символах ведёт к смещению и ошибкам при выполнении.

---

## 3.4.4. Code page и путь к файлу

**Code page** при импорте flat file должна соответствовать кодировке данных (UTF-8, Windows-1251, ISO-8859-1 и т.д.). Несовпадение — искажение символов. См. [§3.3](chapter-03-03.md).

**Путь к файлу:** Source/Target definition не содержит пути. При выполнении Session в настройках Source/Target указывается фактический путь (или параметр `$InputFileName` / `$OutputFileName` в parameter file). Для файлов на FTP — Connection type FTP; для локальных — Connection type None.

---

## 3.4.5. Реляционные БД: Oracle

**Oracle** — одна из наиболее распространённых СУБД в корпоративном ETL. Источник: [Native Connect Strings](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview/native-connect-strings.html), [Relational Database Connections](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/relational-database-connections.html).

**Connect String:** `dbname.world` (аналогично записи в TNSNAMES.ORA). Пример: `oracle.world`. Connect string можно параметризовать.

**Особенности:**
- Kerberos-аутентификация поддерживается.
- Для BLOB/CLOB/NCLOB пользователь должен иметь права на создание и доступ к temporary tablespace.
- Oracle OS Authentication: User Name = `PmNullUser`, Password = `PmNullPassword`.
- Impersonate User — для подключения от имени другого пользователя.

**Типы:** NUMBER, VARCHAR2, CHAR, DATE, TIMESTAMP, CLOB, BLOB, NCLOB — маппятся на transformation datatypes при импорте.

---

## 3.4.6. Реляционные БД: Microsoft SQL Server

**Connect String:** для SQL Server и Sybase ASE не требуется в том же виде — используются **Server Name** и **Database Name** (или Use DSN). Синтаксис при использовании connect string: `servername@dbname`. Пример: `sqlserver@mydatabase`. SSL: `sqlserver@mydatabase;Encrypt=Yes`.

**Provider Type:** ODBC (рекомендуется) или Oledb (deprecated).

**Особенности:**
- Kerberos поддерживается.
- **Use Trusted Connection** — Windows-аутентификация; пользователь, под которым запущен Integration Service, должен иметь доступ к БД.
- **Domain Name** — для SQL Server на Windows.

**Типы:** int, bigint, nvarchar, varchar, datetime, decimal и др.

---

## 3.4.7. Реляционные БД: PostgreSQL, DB2, Teradata

**PostgreSQL:** подключается через ODBC. Connect string в документации PowerCenter для native connect strings не указан явно; обычно используется ODBC DSN или настройка через Database Name и Server. Поддержка — через ODBC-драйвер PostgreSQL.

**IBM DB2:** Connect string — `dbname`. Пример: `mydatabase`.

**Teradata:** Connect string — `ODBC_data_source_name` или `ODBC_data_source_name@db_name` или `ODBC_data_source_name@db_user_name`. Пример: `TeradataODBC`, `TeradataODBC@mydatabase`. Требуется ODBC data source. Database Name и User Name в Connection переопределяют значения из ODBC entry.

**Sybase ASE:** как SQL Server — `servername@dbname`. Kerberos поддерживается.

---

## 3.4.8. Сводка: connect strings и Connection

| СУБД | Connect String (пример) | Примечание |
|------|-------------------------|------------|
| Oracle | `oracle.world` | TNSNAMES-формат |
| SQL Server | `servername@dbname` | Или Server Name + Database Name |
| Sybase ASE | `servername@dbname` | Аналогично SQL Server |
| DB2 | `dbname` | Имя базы |
| Teradata | `DSN` или `DSN@db` | ODBC data source |
| PostgreSQL | ODBC DSN | Через ODBC |

Для всех реляционных БД, кроме SQL Server и Sybase ASE, connect string обязателен. Connection создаётся в Workflow Manager; при импорте Source/Target в Designer используется ODBC data source только для чтения метаданных.

---

## 3.4.9. Flat file как target: запись

При записи в flat file target в Session указывается путь к выходному файлу (или параметр). Формат (delimited/fixed-width) и структура колонок заданы в target definition. Integration Service создаёт файл при выполнении; при необходимости перезапись или дополнение настраивается в свойствах Session. Connection type — None.

---

## 3.4.10. Типичные ошибки

- **Неверная code page для flat file:** искажение кириллицы и спецсимволов. Задавать кодировку, соответствующую файлу.
- **Заголовок в данных:** не задан Start Import At Row — первая строка попадает как данные. Установить Start Import At Row = 2 и при необходимости Import Field Names From First Line.
- **Разделитель внутри поля без qualifier:** при запятой в значении и разделителе-запятой — некорректный разбор. Использовать Text Qualifier (кавычки) или другой разделитель.
- **Fixed-width и multibyte:** границы колонок в байтах; при UTF-8 один символ — несколько байт. Смещение границ — ошибки. Проверять code page и ширину колонок.
- **Connect string для неверной СУБД:** Oracle-формат для PostgreSQL не сработает. Использовать синтаксис, соответствующий типу Connection.

---

## Ключевое

- **Delimited flat file:** character-oriented; разделитель, qualifier, escape; CSV, TSV. Wizard: 3 шага.
- **Fixed-width flat file:** byte-oriented; column breaks, precision в байтах. Осторожно с multibyte.
- **Code page** — должна совпадать с кодировкой файла. Путь к файлу — в Session, не в definition.
- **Oracle:** connect string `dbname.world`; Kerberos, OS Auth, BLOB/CLOB — temporary tablespace.
- **SQL Server, Sybase ASE:** `servername@dbname`; Server Name, Database Name; Trusted Connection, Kerberos.
- **DB2:** `dbname`. **Teradata:** ODBC DSN. **PostgreSQL:** ODBC.
- **Flat file target:** Connection type None; путь в Session.

В [§4.1](chapter-04-01.md) мы перейдём к маппингам: что такое Mapping, как он связывает источники, трансформации и приёмники в единый поток данных.
