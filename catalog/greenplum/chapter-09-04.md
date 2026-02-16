# §9.4 External tables

В [§9.3](chapter-09-03.md) мы разобрали форматы Parquet, ORC и Avro и их поддержку в PXF. Внешние данные становятся доступны в Greenplum через **внешние таблицы** (external tables): объект в системном каталоге с определением столбцов, **LOCATION** (источник данных) и **FORMAT**. Запросы к такой таблице читают или пишут данные во внешней системе «на лету». В этом разделе — синтаксис **CREATE EXTERNAL TABLE** и **CREATE WRITABLE EXTERNAL TABLE**, предложения **LOCATION** (в том числе протокол **pxf://**) и **FORMAT**, использование внешних таблиц в запросах (SELECT, JOIN, загрузка в обычную таблицу) и кратко про протоколы **gpfdist** и **file**. По [Tanzu Greenplum 7 — CREATE EXTERNAL TABLE](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/ref_guide-sql_commands-CREATE_EXTERNAL_TABLE.html), [§9.1](chapter-09-01.md)–[§9.3](chapter-09-03.md).

---

## 9.4.1. Синтаксис CREATE EXTERNAL TABLE

**Внешняя таблица** — объект БД, не хранящий данные в кластере: определение задаёт **источник** (LOCATION) и **формат** данных (FORMAT). Различают **читаемые** (readable) и **записываемые** (writable) внешние таблицы. См. [Глоссарий](glossary.md) (Внешняя таблица).

**Читаемая внешняя таблица** (по умолчанию) используется для **чтения** данных из внешнего источника: запросы SELECT, JOIN с другими таблицами, загрузка в обычную таблицу (INSERT INTO ... SELECT * FROM external_table). На читаемой внешней таблице **нельзя** выполнять INSERT, UPDATE, DELETE, TRUNCATE и создавать индексы. Создание: **CREATE EXTERNAL TABLE** (или **CREATE READABLE EXTERNAL TABLE**).

**Записываемая внешняя таблица** используется для **выгрузки** данных из Greenplum во внешний источник: допустима только операция **INSERT** (INSERT INTO writable_external_table SELECT ... FROM ...). SELECT, UPDATE, DELETE по записываемой таблице выполнять нельзя. Создание: **CREATE WRITABLE EXTERNAL TABLE**.

Общий вид команды (упрощённо по документации Greenplum):

```text
CREATE [READABLE] EXTERNAL [TEMPORARY | TEMP] TABLE имя_таблицы
  ( столбец тип [, ...] | LIKE другая_таблица )
  LOCATION ( 'uri' [, ...] )
  [ON COORDINATOR]
  FORMAT 'TEXT' | 'CSV' | 'CUSTOM' ( ... )
  [ ENCODING 'кодировка' ]
  [ LOG ERRORS [PERSISTENTLY] SEGMENT REJECT LIMIT n [ ROWS | PERCENT ] ];
```

Для **записываемой** таблицы после FORMAT может указываться **DISTRIBUTED BY** или **DISTRIBUTED RANDOMLY** — политика распределения строк при выгрузке; совпадение с распределением исходной таблицы уменьшает пересылку данных. См. справочник Greenplum.

---

## 9.4.2. LOCATION: протоколы и URI

В **LOCATION** задаётся один или несколько URI внешних данных. Поддерживаемые протоколы (по версии Greenplum):

- **pxf://** — доступ через PXF к HDFS, S3, Azure, GCS, Hive, JDBC и др. Формат: `pxf://путь_или_объект?PROFILE=профиль[&SERVER=сервер][&опция=значение ...]`. Путь зависит от источника ([§9.2](chapter-09-02.md)); для объектных хранилищ обязателен SERVER=. См. [§9.1](chapter-09-01.md).
- **gpfdist://** — данные, раздаваемые утилитой **gpfdist** (параллельная загрузка/выгрузка с файлового сервера). URI вида `gpfdist://хост:порт/путь`; путь относителен каталогу, из которого запущен gpfdist; допустимы шаблоны (например, `*.csv`). Несколько URI в списке распределяются по сегментам для параллельного чтения. См. [§7.3](chapter-07-03.md).
- **file://** — файлы в локальной файловой системе **сегментов**; URI вида `file://хост:порт/абсолютный_путь`. Каждый сегмент читает свой файл из списка LOCATION.
- **s3://** — в ряде версий Greenplum поддерживается нативный протокол S3 (LOCATION с s3://, параметры region, config); детали см. в документации. Альтернатива — PXF с профилем s3:* и SERVER=.
- **http://** — для **внешних web-таблиц** (EXTERNAL WEB TABLE); один URI на источник.

Для **PXF** в одном LOCATION указывается один URI с протоколом pxf:// и параметрами в query-части. Для **gpfdist** и **file** можно перечислить несколько URI — по одному на сегмент или группу сегментов. См. справочник CREATE EXTERNAL TABLE.

---

## 9.4.3. FORMAT и опции для PXF

**FORMAT** задаёт способ разбора (чтение) или формирования (запись) данных:

- **FORMAT 'TEXT'** — текст с разделителем столбцов (DELIMITER), опции NULL, ESCAPE, HEADER и др. Для протокола **pxf** опция **HEADER** не поддерживается.
- **FORMAT 'CSV'** — CSV с настройками QUOTE, DELIMITER, NULL, FORCE NOT NULL и др.
- **FORMAT 'CUSTOM' (FORMATTER='имя')** — для PXF при чтении бинарных форматов (Parquet, ORC, Avro) указывают **pxfwritable_import**, при записи — **pxfwritable_export**. Для текстовых профилей PXF (hdfs:text, s3:text и т.д.) обычно используется FORMAT 'TEXT' или 'CSV' с DELIMITER. См. документацию PXF по профилям.

Пример читаемой внешней таблицы для **текстового файла в HDFS** (по документации PXF):

```sql
CREATE EXTERNAL TABLE pxf_hdfs_text(location text, month text, num_orders int, total_sales float8)
  LOCATION ('pxf://data/pxf_examples/pxf_hdfs_simple.txt?PROFILE=hdfs:text')
  FORMAT 'TEXT' (delimiter=E',');
```

Пример для **Parquet в S3** (обязателен SERVER=):

```sql
CREATE EXTERNAL TABLE pxf_s3_parquet(id int, name text, ts timestamp)
  LOCATION ('pxf://my-bucket/data/file.parquet?PROFILE=s3:parquet&SERVER=s3srvcfg')
  FORMAT 'CUSTOM' (FORMATTER='pxfwritable_import');
```

Пример **записываемой** внешней таблицы для выгрузки в HDFS в формате Parquet:

```sql
CREATE WRITABLE EXTERNAL TABLE pxf_export_parquet(id int, name text)
  LOCATION ('pxf://data/export?PROFILE=hdfs:parquet')
  FORMAT 'CUSTOM' (FORMATTER='pxfwritable_export')
  DISTRIBUTED BY (id);
```

Дополнительные опции профиля (например, COMPRESSION_CODEC для Parquet) задаются в query-части LOCATION. См. [§9.2](chapter-09-02.md), [§9.3](chapter-09-03.md), документацию PXF.

---

## 9.4.4. Запросы к внешним данным

После создания внешней таблицы с ней работают средствами SQL так же, как с обычной таблицей (с учётом того, что это только чтение или только запись).

**Читаемая внешняя таблица:**
- **SELECT** — выборка, фильтрация (WHERE), агрегация, сортировка; при использовании PXF и поддерживаемых форматов применяются predicate pushdown и column projection ([§9.3](chapter-09-03.md)).
- **JOIN** с локальными таблицами Greenplum — в одном запросе можно объединять данные из кластера и из внешнего источника (федеративный запрос).
- **INSERT INTO внутренняя_таблица SELECT ... FROM внешняя_таблица** — загрузка выбранного подмножества данных в обычную таблицу; удобно для ETL и переноса только нужных строк/столбцов.
- **VIEW** — можно создавать представления над внешними таблицами.

**Записываемая внешняя таблица:**
- **INSERT INTO записываемая_внешняя_таблица SELECT ... FROM ...** — выгрузка результата запроса во внешний источник (файлы в HDFS, объекты в S3 и т.д.).

Данные во внешней таблице **не кэшируются** в Greenplum: каждый запрос читает актуальные данные из источника. Статистика по внешним таблицам в общем случае не собирается (ANALYZE для них не применим или не имеет смысла), поэтому планировщик может опираться только на оценки по умолчанию; для сложных запросов с JOIN стоит проверять план (EXPLAIN). См. [§8.1](chapter-08-01.md), [§8.2](chapter-08-02.md).

---

## 9.4.5. Дополнительные опции

- **LOG ERRORS SEGMENT REJECT LIMIT** — для читаемых внешних таблиц (текст, CSV, gpfdist, file): режим «изоляции ошибок по строкам»; строки с ошибками формата отбрасываются, пока число таких строк на сегменте не превысит лимит (в строках или в процентах). Ошибки можно просматривать функцией **gp_read_error_log('имя_таблицы')**. С PXF поддержка зависит от профиля; см. документацию.
- **ENCODING** — кодировка символов данных (например, 'UTF8').
- **TEMPORARY | TEMP** — внешняя таблица создаётся как временная; удаляется в конце сессии.
- **ON COORDINATOR** — для протоколов, поддерживающих эту опцию (например, s3), все операции с таблицей выполняются только на координаторе; может влиять на производительность. Протоколы pxf, gpfdist, file не поддерживают ON COORDINATOR.

---

## 9.4.6. Типичные ошибки

- **Не указать SERVER= для объектного хранилища (S3, Azure, GCS):** запрос к внешней таблице завершится ошибкой; в LOCATION для pxf:// с такими источниками всегда указывайте SERVER=.
- **Использовать HEADER с протоколом pxf:** PXF не поддерживает опцию HEADER в FORMAT; заголовки в файле при необходимости обрабатывайте отдельно или не используйте.
- **Выполнять INSERT/UPDATE по читаемой внешней таблице:** допустимы только SELECT и загрузка из неё в другую таблицу; для выгрузки создавайте WRITABLE EXTERNAL TABLE.
- **Ожидать актуальную статистику по внешней таблице:** ANALYZE по ним не собирает статистику; при плохих планах JOIN проверяйте план и при необходимости подправляйте запрос или объём читаемых данных.

---

## Ключевое

- **CREATE EXTERNAL TABLE** — читаемая внешняя таблица (источник данных); **CREATE WRITABLE EXTERNAL TABLE** — записываемая (выгрузка). LOCATION задаёт URI (pxf://, gpfdist://, file:// и др.); FORMAT — TEXT, CSV или CUSTOM с форматтером PXF.
- Для **pxf://** в LOCATION обязательны **PROFILE=** и для объектных хранилищ **SERVER=**; путь зависит от типа источника (HDFS, S3, Hive и т.д.). Parquet/ORC/Avro — FORMAT 'CUSTOM' (pxfwritable_import/export).
- К внешней таблице обращаются как к обычной: **SELECT**, **JOIN** с локальными таблицами, **INSERT INTO внутренняя SELECT FROM внешняя**; записываемая — только **INSERT INTO внешняя SELECT ...**. Статистика по внешним таблицам не собирается.

В [§10.1](chapter-10-01.md) мы перейдём к **in-database аналитике**: Apache MADlib — библиотека машинного обучения и статистики внутри Greenplum.
