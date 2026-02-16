# §3.3 Коннекторы и типы данных

В [§3.1](chapter-03-01.md) и [§3.2](chapter-03-02.md) мы рассмотрели определение источников и приёмников. При импорте из БД или файла Designer получает **native datatypes** (типы, специфичные для источника или приёмника). Внутри маппинга данные проходят через **transformation datatypes** — универсальные типы PowerCenter, позволяющие читать из Oracle и писать в SQL Server, из flat file — в PostgreSQL. В этом разделе мы разберём коннекторы (ODBC, нативные драйверы), различие native и transformation datatypes, маппинг типов при чтении и записи, а также способы преобразования данных. Подробнее форматы flat file и конкретные БД — в [§3.4](chapter-03-04.md). См. [Глоссарий](glossary.md).

---

## 3.3.1. Коннекторы: ODBC и нативные драйверы

**Коннектор** (connector) — механизм подключения PowerCenter к источнику или приёмнику. Для реляционных БД используются ODBC и нативные драйверы. Источник: [Relational Database Connections](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/relational-database-connections.html), [PowerExchange ODBC Drivers](https://docs.informatica.com/data-integration/powercenter/10-5-8/powerexchange-interfaces-for-powercenter/part-1--introduction/powerexchange-interfaces-for-powercenter/powerexchange-odbc-drivers.html).

**ODBC (Open Database Connectivity)** — стандартный интерфейс доступа к БД. PowerCenter поддерживает ODBC data source, настроенные в системе (ODBC Administrator). При импорте Source/Target в Designer выбирается ODBC data source; при выполнении Session Connection указывает на тот же DSN или на нативный connect string. ODBC обеспечивает совместимость с широким кругом СУБД (Oracle, SQL Server, PostgreSQL, DB2, Sybase, Teradata и др.) без отдельного драйвера для каждой.

**Нативные драйверы** — специализированные драйверы Informatica (PowerExchange или встроенные) для конкретных СУБД. Обеспечивают лучшую производительность, поддержку bulk load, Kerberos-аутентификации (Oracle, DB2, SQL Server, Sybase). Для Microsoft SQL Server доступен выбор Provider Type: ODBC или Oledb (deprecated).

**Connection types** при настройке Session: Relational (источник, приёмник, lookup, stored procedure), FTP (доступ к файлам по FTP/SFTP), Loader (внешние загрузчики: DB2 Autoloader, Teradata FastLoad), Queue (WebSphere MQ, MSMQ), Application (PowerExchange, SAP NetWeaver, Netezza и др.), None (flat file, XML). См. [Connection Types](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview/connection-types.html).

---

## 3.3.2. Native datatypes и Transformation datatypes

Designer отображает два вида типов данных. Источник: [Datatype Reference Overview](https://docs.informatica.com/data-integration/powercenter/10-4-1/designer-guide/datatype-reference/datatype-reference-overview.html).

**Native datatypes** — типы, специфичные для источника или приёмника (Oracle NUMBER, VARCHAR2; SQL Server int, nvarchar; PostgreSQL bigint, text; flat file — string с длиной). Отображаются в Source Analyzer и Target Designer, в source/target definitions в Mapping Designer. При импорте из БД Informatica получает native types через ODBC; для flat file типы задаются при импорте (string, integer и т.д.).

**Transformation datatypes** — внутренние типы PowerCenter, основанные на ANSI SQL-92; используются во всех трансформациях маппинга. Позволяют переносить данные между разными платформами: читать из Oracle, писать в Sybase; читать из flat file, писать в SQL Server. Integration Service при чтении преобразует native datatypes в transformation datatypes; при записи — transformation datatypes в native datatypes целевой системы.

| Этап | Преобразование |
|------|----------------|
| Чтение из источника | Native (source) → Transformation |
| Поток в маппинге | Transformation datatypes |
| Запись в приёмник | Transformation → Native (target) |

---

## 3.3.3. Transformation datatypes: основные типы

Список transformation datatypes (по документации PowerCenter 10.4.1/10.5). Источник: [Transformation Data Types](https://docs.informatica.com/data-integration/powercenter/10-4-1/designer-guide/datatype-reference/transformation-data-types.html).

| Тип | Размер | Описание |
|-----|--------|----------|
| **Integer** | 4 байта | Целое: от -2 147 483 648 до 2 147 483 647 (precision 10, scale 0) |
| **Small Integer** | 4 байта | Целое: от -32 768 до 32 767 (precision 5, scale 0) |
| **Bigint** | 8 байт | Целое: от -9 223 372 036 854 775 808 до 9 223 372 036 854 775 807 (precision 19, scale 0) |
| **Decimal** | 8–24 байта | Десятичное с precision и scale; precision 1–38, scale 0–38 |
| **Double** | 8 байт | Число с плавающей точкой двойной точности |
| **Real** | 8 байт | Число с плавающей точкой (precision 7, scale 0) |
| **String** | 1–104 857 600 символов | Строка фиксированной или переменной длины; размер зависит от code page (ASCII/Unicode) |
| **Text** | 1–104 857 600 символов | Строка; аналогично String |
| **Nstring** | 1–104 857 600 символов | Строка Unicode (NCHAR/NVARCHAR) |
| **Ntext** | 1–104 857 600 символов | Строка Unicode; аналогично Nstring |
| **Date/Time** | 16 байт | Дата и время: 0001-01-01 до 9999-12-31, точность до наносекунды |
| **timestampWithTZ** | 40 байт | Timestamp с часовым поясом; диапазон 1947–2040, зоны -12:00 до +14:00 |
| **Binary** | 1–104 857 600 байт | Бинарные данные; не поддерживается для COBOL и flat file sources |
| **Array, Map, Struct** | Переменный | Сложные типы для complex sources/targets |

Для multibyte character set типы String/Text/Nstring/Ntext выделяют дополнительное место (до 3 байт на символ). Decimal и Double допускают настройку precision и scale; при несовпадении с целевым native type возможна потеря точности или округление.

---

## 3.3.4. Маппинг native → transformation при чтении

При импорте Source из БД Informatica получает метаданные через ODBC: для каждой колонки — native type (например, Oracle NUMBER(10,2), VARCHAR2(100)). Designer сохраняет native type в source definition; при добавлении Source в маппинг Source Qualifier автоматически сопоставляет native type с transformation type по таблице маппинга Informatica.

Примеры (типичные, точный маппинг зависит от версии и СУБД):

- Oracle NUMBER → Decimal или Integer (в зависимости от precision/scale)
- Oracle VARCHAR2, CHAR → String
- Oracle DATE, TIMESTAMP → Date/Time
- SQL Server int → Integer; bigint → Bigint; nvarchar → Nstring
- PostgreSQL bigint → Bigint; text → String
- Flat file (string) → String; (integer) → Integer

При выполнении Session Integration Service читает данные из источника и преобразует каждое значение в transformation type. Если значение не помещается (например, слишком длинная строка) или не конвертируется (нечисловая строка в Integer) — ошибка или NULL в зависимости от настроек.

---

## 3.3.5. Маппинг transformation → native при записи

При записи в target Integration Service преобразует transformation datatypes в native types целевой БД или файла. Target definition содержит native types (импортированные из БД или заданные для flat file). Несовпадение типов (например, Decimal с precision 38 в Oracle NUMBER(10,2)) может привести к округлению, усечению или ошибке.

Рекомендации:

- Импортировать target из реальной БД, чтобы native types совпадали.
- При ручном создании target — сверять precision/scale с целевой таблицей.
- Для flat file — задавать длину строки не меньше максимальной длины данных; иначе усечение.

---

## 3.3.6. Преобразование данных в маппинге

Преобразование из одного типа в другой внутри маппинга выполняется тремя способами. Источник: [Converting Data](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/appendix-a--datatype-reference/converting-data.html).

**1. Port-to-port conversion (передача между портами разных типов)**

При связывании выходного порта одной трансформации с входным портом другой, если типы различаются, Integration Service выполняет неявное преобразование. Поддерживаемые пары (например, String → Date/Time при корректном формате, Integer → Decimal) описаны в документации. Неподдерживаемые преобразования приводят к ошибке валидации маппинга.

**2. Функции трансформаций**

Использование встроенных функций: `TO_DATE()`, `TO_CHAR()`, `TO_DECIMAL()`, `TO_INTEGER()` и др. — для явного преобразования в Expression. Пример: `TO_DATE(IN_DATE_STR, 'YYYY-MM-DD')` для строки в дату.

**3. Арифметические операторы**

Арифметика в Expression может приводить к неявному преобразованию (например, Integer + Decimal → Decimal).

---

## 3.3.7. Преобразование строк в даты и числа

Специализированные сценарии: преобразование строк в даты и числа. Источник: [Converting Data](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/appendix-a--datatype-reference/converting-data.html).

**Строка → дата:** функция `TO_DATE(string, format)`. Формат задаётся маской (YYYY, MM, DD, HH24, MI, SS и т.д.). Если строка не соответствует формату — NULL или ошибка. Рекомендуется явно указывать формат и проверять данные на корректность.

**Строка → число:** `TO_DECIMAL()`, `TO_INTEGER()` и аналоги. Нечисловые символы в строке приводят к NULL или ошибке. Для очистки можно использовать `REPLACECHR()`, `LTRIM()` и т.п. перед преобразованием.

---

## 3.3.8. Code page и Unicode

При работе со строками важна **code page** (кодовая страница) — набор символов для интерпретации байтов. Для flat file и некоторых БД Connection или Source/Target definition содержат настройку Code Page. В Unicode mode (UTF-8, UTF-16) типы String/Text/Nstring/Ntext занимают больше места: `(precision + 1) * 2` байт. Несовпадение code page между источником и приёмником может привести к искажению символов (например, кириллица). Для мультибайтовых символов (до 3 байт на символ) выделяется дополнительное пространство.

---

## 3.3.9. Типичные ошибки при работе с типами

- **Несоответствие precision/scale:** Decimal с precision 38 в target NUMBER(10,2) — округление или ошибка. Сверять с целевой таблицей.
- **Усечение строк:** String(100) в target VARCHAR(50) — обрезка. Увеличить длину target или обрезать в Expression.
- **Неверный формат даты:** TO_DATE при некорректной строке — NULL. Проверять формат и данные; использовать IIF/ISNULL для обработки ошибок.
- **Преобразование NULL:** при передаче NULL между портами разных типов NULL обычно сохраняется; проверять логику в Expression.
- **Несовпадение code page:** искажение при копировании в другую кодировку. Задавать единую code page для источника и приёмника или конвертировать явно.

---

## Ключевое

- **Коннекторы:** ODBC и нативные драйверы для реляционных БД; Connection types: Relational, FTP, Loader, Queue, Application, None.
- **Native datatypes** — типы источника/приёмника (Oracle, SQL Server, flat file); отображаются в Source Analyzer и Target Designer.
- **Transformation datatypes** — универсальные типы PowerCenter (Integer, Decimal, String, Date/Time и др.); используются во всех трансформациях; позволяют переносить данные между разными платформами.
- **Поток:** чтение — native → transformation; маппинг — transformation; запись — transformation → native.
- **Преобразование:** port-to-port (неявное), функции (TO_DATE, TO_DECIMAL и др.), арифметика.
- **Code page** — важна для корректной работы со строками и Unicode; при multibyte — дополнительное место.

В [§3.4](chapter-03-04.md) мы разберём Flat File и базы данных: CSV, фиксированная ширина, Oracle, SQL Server, PostgreSQL и другие типы источников и приёмников.
