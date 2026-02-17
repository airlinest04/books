# §6.3 DDL в Hive: создание таблиц

В [§6.2](chapter-06-02.md) мы разобрали метаданные Hive и типы таблиц (внутренние и внешние). **DDL** (Data Definition Language) в Hive — команды создания и изменения структуры объектов: таблицы, базы данных. В этом разделе — **CREATE TABLE**: схема (столбцы и типы), **формат хранения** (TEXTFILE, PARQUET, ORC), **партиции** (PARTITIONED BY) и **расположение** (LOCATION). Детали партиционирования и bucketing — в [§6.5](chapter-06-05.md); загрузка данных и запросы (DML) — в [§6.4](chapter-06-04.md). См. [Глоссарий](glossary.md).

---

## 6.3.1. CREATE TABLE: схема и базовая структура

Таблица в Hive задаётся именем, **списком столбцов с типами** и опциями хранения. Базовая форма:

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] имя_таблицы
  ( столбец1 тип1 [COMMENT 'комментарий'], столбец2 тип2, ... )
  [COMMENT 'комментарий к таблице']
  [PARTITIONED BY ( столбец_партиции тип [COMMENT '...'], ... )]
  [CLUSTERED BY ( столбец ) INTO N BUCKETS]
  [ROW FORMAT ...]
  [STORED AS формат]
  [LOCATION 'путь_в_HDFS']
  [TBLPROPERTIES (...)];
```

- **EXTERNAL** — внешняя таблица ([§6.2](chapter-06-02.md)); без EXTERNAL таблица внутренняя (managed).
- **Столбцы:** имя и тип (INT, BIGINT, STRING, DOUBLE, TIMESTAMP, DATE, DECIMAL, ARRAY, MAP, STRUCT и др.). Столбцы партиций не включают в основной список — они задаются в PARTITIONED BY и при чтении добавляются к таблице виртуально.
- **IF NOT EXISTS** — не выдавать ошибку, если таблица уже есть.

Пример простой таблицы без партиций и без явного формата (по умолчанию текст в Hive):

```sql
CREATE TABLE events (
  id BIGINT,
  user_id INT,
  event_type STRING,
  ts TIMESTAMP
);
```

Данные такой таблицы по умолчанию хранятся как текст в каталоге warehouse; формат строки и разделители можно задать через ROW FORMAT (см. ниже).

---

## 6.3.2. Формат хранения: STORED AS

**STORED AS** задаёт **формат файлов** в каталоге таблицы; от него зависят способ записи/чтения и совместимость с [гл. 5](chapter-05-01.md). См. [Глоссарий](glossary.md).

**TEXTFILE** — текстовые файлы (по умолчанию для простого CREATE TABLE без STORED AS). Строки хранятся как текст; разделители полей и строк задаются через ROW FORMAT (например, ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' LINES TERMINATED BY '\n'). Подходит для CSV-подобных данных; сжатие возможно на уровне файла (Gzip и т.д.), но без выборочного чтения столбцов.

**PARQUET** — колоночный формат Parquet ([§5.3](chapter-05-03.md)). Схема хранится в файле; поддерживаются выборка столбцов и предикат pushdown. Указывается как `STORED AS PARQUET`.

**ORC** — колоночный формат ORC ([§5.3](chapter-05-03.md)). Аналогично: колоночное хранение, сжатие, предикат pushdown. Указывается как `STORED AS ORC` (часто с опциями, например ORC с компрессией).

**SEQUENCEFILE** — бинарный формат пар ключ–значение ([§5.2](chapter-05-02.md)); реже используется для новых таблиц.

Пример таблицы в Parquet:

```sql
CREATE TABLE events_parquet (
  id BIGINT,
  user_id INT,
  event_type STRING,
  ts TIMESTAMP
)
STORED AS PARQUET;
```

Пример таблицы в ORC:

```sql
CREATE TABLE events_orc (
  id BIGINT,
  user_id INT,
  event_type STRING,
  ts TIMESTAMP
)
STORED AS ORC;
```

Выбор формата влияет на производительность запросов и место на диске; для аналитических витрин обычно выбирают Parquet или ORC ([§5.4](chapter-05-04.md)).

---

## 6.3.3. ROW FORMAT для текстовых таблиц

Для таблиц в формате **TEXTFILE** (или по умолчанию) структура строки задаётся **ROW FORMAT**. Типичный вариант — разделители полей и строк:

```sql
ROW FORMAT DELIMITED
  FIELDS TERMINATED BY '\t'
  LINES TERMINATED BY '\n'
  NULL DEFINED AS '\\N'
```

- **FIELDS TERMINATED BY** — разделитель столбцов (табуляция, запятая и т.д.).
- **LINES TERMINATED BY** — разделитель строк (обычно перевод строки).
- **NULL DEFINED AS** — последовательность символов, обозначающая NULL в файле.

Для сложных форматов (JSON, регулярные выражения) используется SerDe (например, OpenX JsonSerDe). Таблицы в Parquet и ORC не используют ROW FORMAT в том же смысле — формат файла уже определяет структуру.

---

## 6.3.4. PARTITIONED BY и LOCATION

**PARTITIONED BY** задаёт **столбцы партиции**; они не хранятся в теле файлов, а кодируются в структуре каталогов (подкаталог на каждую комбинацию значений партиций). Синтаксис: `PARTITIONED BY (col_name type [, ...])`. Столбцы партиций не перечисляются в основном списке столбцов таблицы — они добавляются к схеме при чтении. Детали партиционирования, загрузка в партиции и ускорение запросов — в [§6.5](chapter-06-05.md). См. [Глоссарий](glossary.md).

Пример таблицы, партиционированной по дате:

```sql
CREATE TABLE events_part (
  id BIGINT,
  user_id INT,
  event_type STRING
)
PARTITIONED BY (dt STRING)
STORED AS PARQUET;
```

Данные будут лежать в подкаталогах вида `.../events_part/dt=2024-01-15/`. При запросе с условием `WHERE dt = '2024-01-15'` Hive может читать только соответствующий подкаталог (partition pruning).

**LOCATION** задаёт **каталог в HDFS**, в котором лежат (или будут лежать) файлы таблицы. Для внешней таблицы LOCATION обычно обязателен и указывает на уже существующий каталог; для внутренней — опционален (по умолчанию используется каталог в warehouse).

Пример внешней таблицы с явным путём и форматом:

```sql
CREATE EXTERNAL TABLE raw_events (
  id BIGINT,
  payload STRING
)
PARTITIONED BY (dt STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/data/raw/events';
```

---

## 6.3.5. Другие DDL-операции

- **DROP TABLE [IF EXISTS] имя;** — удаляет таблицу. Для внутренней таблицы удаляются метаданные и каталог с данными в HDFS; для внешней — только метаданные ([§6.2](chapter-06-02.md)).
- **CREATE DATABASE [IF NOT EXISTS] имя;** — создаёт базу (пространство имён для таблиц). Таблицы можно создавать в конкретной базе: `CREATE TABLE db_name.table_name (...)`.
- **USE db_name;** — переключить текущую базу для последующих команд.
- **ALTER TABLE** — изменение свойств таблицы: переименование столбцов, добавление столбцов, изменение LOCATION, добавление/удаление партиций (например, ADD PARTITION, DROP PARTITION). Для партиционированных таблиц после копирования файлов в новый подкаталог часто вызывают `MSCK REPAIR TABLE table_name` или `ALTER TABLE ... ADD PARTITION` для регистрации партиций в Metastore.

Полный синтаксис и опции зависят от версии Hive; здесь приведены основные конструкции для создания таблиц с нужным форматом и партициями.

---

## Ключевое

- **CREATE TABLE** задаёт схему (столбцы, типы), при необходимости PARTITIONED BY, ROW FORMAT (для текста), STORED AS и LOCATION.
- **STORED AS:** TEXTFILE (текст), PARQUET, ORC — основные форматы; Parquet и ORC предпочтительны для аналитических витрин.
- **PARTITIONED BY** — столбцы партиции; данные организуются в подкаталогах; ускоряет запросы с фильтром по партиции ([§6.5](chapter-06-05.md)).
- **LOCATION** — явный путь в HDFS к каталогу таблицы; для внешних таблиц обычно обязателен.
- **CREATE EXTERNAL TABLE** + LOCATION — таблица над существующим каталогом; DROP не удаляет данные.

В [§6.4](chapter-06-04.md) разберём DML: SELECT, JOIN, агрегация, загрузка данных (LOAD, INSERT OVERWRITE), просмотр плана и выполнение запросов.
