# §6.1 Source Qualifier

В [§5.1](chapter-05-01.md)–[§5.4](chapter-05-04.md) мы рассмотрели основы трансформаций. **Source Qualifier** — обязательная трансформация для реляционного или flat file источника: она связывает Source с потоком данных и определяет, как Integration Service читает данные. По умолчанию генерируется `SELECT * FROM table`; при необходимости можно задать **SQL-override**, **Source Filter**, **сортировку** и **join нескольких таблиц** — всё это выполняется на стороне источника (pushdown), что снижает объём передаваемых данных. В этом разделе разберём эти возможности и ограничения. См. [Глоссарий](glossary.md).

---

## 6.1.1. Назначение и default query

**Source Qualifier** — активная, подключённая, нативная трансформация. При добавлении реляционного или flat file source в маппинг Designer автоматически создаёт Source Qualifier и связывает его с source. Источник: [Source Qualifier Transformation](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation.html), [Working with Sources in a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/working-with-sources-in-a-mapping.html).

**Default query** — запрос по умолчанию. Для одной таблицы: `SELECT col1, col2, ... FROM schema.table`. Список колонок и их порядок определяются подключёнными портами Source Qualifier. Источник: [Overriding the Default Query](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/default-query/overriding-the-default-query.html).

При изменении свойств Source Qualifier (Source Filter, Number of Sorted Ports, User-Defined Join, Select Distinct) Designer модифицирует default query. Если же задан **SQL Query** (SQL-override), Integration Service использует только его — все остальные настройки игнорируются.

---

## 6.1.2. SQL-override

**SQL-override** (переопределение SQL) — возможность заменить default query на произвольный SQL, поддерживаемый источником. Источник: [Adding an SQL Query](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/adding-an-sql-query.html).

**Где задаётся:** Properties Source Qualifier → SQL Query → Open. Можно сгенерировать default query (Generate SQL) и отредактировать его, либо ввести запрос с нуля.

**Обязательные правила:**

| Правило | Описание |
|---------|----------|
| Порядок колонок | SELECT должен перечислять колонки в том же порядке, в каком порты отображаются в Source Qualifier. Иначе сессия может упасть или вернуть неверные данные. |
| Квалификация имён | Каждая колонка должна быть квалифицирована именем таблицы/представления: `ORDERS.ORDER_ID`, `CUSTOMERS.CUST_NAME`. |
| Зарезервированные слова | При использовании зарезервированных слов БД — заключать в кавычки. |
| Валидация | PowerCenter не проверяет синтаксис SQL-override; перед использованием тестировать запрос на источнике. |

**Mapping parameters и переменные:** можно использовать `$$ParameterName` и `$VariableName` в запросе. Строковые — в кавычках, соответствующих источнику; datetime — формат даты должен совпадать с форматом источника.

**Пример SQL-override с фильтром и сортировкой:**

```sql
SELECT ORDERS.ORDER_ID, ORDERS.CUST_ID, ORDERS.ORDER_DATE, ORDERS.AMOUNT
FROM ORDERS
WHERE ORDERS.ORDER_DATE >= $$StartDate
  AND ORDERS.STATUS = 'ACTIVE'
ORDER BY ORDERS.ORDER_DATE, ORDERS.ORDER_ID
```

При pushdown optimization (см. [§11.3](chapter-11-03.md)) Integration Service создаёт view на основе SQL-override и выполняет запрос к этой view. Источник: [Source Qualifier Transformation with an SQL Override](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization-and-transformations/source-qualifier-transformation/source-qualifier-transformation-with-an-sql-override.html).

---

## 6.1.3. Фильтрация на уровне источника (Source Filter)

**Source Filter** — условие фильтрации, добавляемое в WHERE default query. Уменьшает число строк, передаваемых из источника в Integration Service. Источник: [Entering a Source Filter](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/entering-a-source-filter.html).

**Где задаётся:** Properties Source Qualifier → Source Filter → Open.

**Важно:**

- В Source Filter **не включать** ключевое слово `WHERE` — Designer добавляет его автоматически.
- Включать имя таблицы и порта: `ORDERS.STATUS = 'ACTIVE'`, а не `STATUS = 'ACTIVE'`.
- Не включать строку `'WHERE'` и large objects — сессия завершится ошибкой.
- Можно использовать mapping parameters и переменные; строковые — в кавычках; datetime — формат должен соответствовать источнику.

**Пример:** `ORDERS.ORDER_DATE >= $$StartDate AND ORDERS.STATUS = 'ACTIVE'`

Если задан SQL-override, Source Filter игнорируется. SQL-override в Session properties переопределяет и Source Filter, и SQL Query на уровне маппинга.

---

## 6.1.4. Сортировка (Sorted Ports)

**Number of Sorted Ports** — число портов, по которым выполняется сортировка. Integration Service добавляет их в ORDER BY default query, начиная с верхних портов Source Qualifier. Источник: [Using Sorted Ports](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/using-sorted-ports.html).

**Где задаётся:** Properties Source Qualifier → Number of Sorted Ports.

**Применение:**

- **Aggregator** с sorted input — group by порты должны совпадать по порядку с sorted ports.
- **Joiner** с sorted input — порядок sorted ports в каждом Source Qualifier должен быть одинаковым для корректного merge join.

**Ограничения:**

- Только для реляционных источников (не flat file).
- Порядок сортировки БД должен соответствовать session sort order.
- Sybase: максимум 16 колонок в ORDER BY.
- При SQL-override Number of Sorted Ports игнорируется; ORDER BY нужно указывать в самом запросе.

При pushdown с ORDER BY в SQL-override на IBM DB2, Microsoft SQL Server, Sybase ASE или Teradata сессия может завершиться ошибкой. Источник: [Source Qualifier Transformation with an SQL Override](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization-and-transformations/source-qualifier-transformation/source-qualifier-transformation-with-an-sql-override.html).

---

## 6.1.5. Join нескольких таблиц

Один Source Qualifier может объединять данные из **нескольких реляционных таблиц** одного экземпляра или сервера БД. Join выполняется на стороне источника до передачи данных в Integration Service; при наличии индексов это повышает производительность. Источник: [Joining Source Data](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/joining-source-data.html).

**Как настроить:**

1. Отключить автосоздание Source Qualifier при добавлении source (настройки Designer).
2. Создать Source Qualifier вручную и подключить к нему несколько relational sources.
3. Задать join: User-Defined Join (условия соединения) или Key Relationships (связи по ключам). Поддерживаются INNER и OUTER join.

**Ограничения:**

- Только таблицы из одной БД (одного connection).
- Для **гетерогенных источников** (разные БД, flat file) — использовать трансформацию **Joiner** (см. [§7.3](chapter-07-03.md)).

---

## 6.1.6. Типичные ошибки

- **Неправильный порядок колонок в SQL-override:** SELECT должен перечислять колонки в том же порядке, что и порты Source Qualifier; иначе неверные данные или падение сессии.
- **Включить WHERE в Source Filter:** Designer добавляет WHERE сам; дублирование приведёт к ошибке.
- **SQL-override с ORDER BY при pushdown на DB2/SQL Server/Sybase/Teradata:** сессия может упасть; проверять документацию по pushdown.
- **SQL-override со ссылкой на database sequence:** сессия завершится ошибкой.
- **Select Distinct + SQL-override:** настройка Select Distinct игнорируется при наличии SQL-override.
- **Не тестировать SQL на источнике:** PowerCenter не валидирует синтаксис; ошибки проявятся только при выполнении сессии.

---

## Ключевое

- **Source Qualifier** — обязательная трансформация для relational/flat file source; по умолчанию генерирует `SELECT * FROM table`.
- **SQL-override** — полная замена default query; порядок колонок в SELECT должен совпадать с порядком портов; квалификация имён обязательна.
- **Source Filter** — условие WHERE без ключевого слова WHERE; уменьшает объём данных на этапе извлечения.
- **Number of Sorted Ports** — сортировка на стороне источника; полезна для Aggregator и Joiner с sorted input; при SQL-override игнорируется.
- **Join в Source Qualifier** — объединение нескольких таблиц одной БД; для гетерогенных источников — Joiner.

В [§6.2](chapter-06-02.md) мы разберём трансформацию Expression: вычисления, функции и условную логику на уровне строки.
