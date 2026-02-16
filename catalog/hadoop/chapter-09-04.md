# §9.4 Связь Sqoop с Hive и Oozie

В [§9.3](chapter-09-03.md) мы разобрали экспорт из HDFS в БД. Импортированные Sqoop данные часто нужно сразу использовать в **Hive** (таблицы, запросы), а сам импорт и экспорт — запускать по расписанию или в цепочке с другими шагами через **Oozie**. В этом разделе — **импорт напрямую в таблицу Hive** (опции Sqoop) и **использование Sqoop в Oozie workflow** (Sqoop action). На этом заканчивается глава 9 и основной объём книги; приложения — [Глоссарий](glossary.md) и список литературы. См. [Глоссарий](glossary.md).

---

## 9.4.1. Импорт в таблицу Hive

По умолчанию **sqoop import** пишет файлы в указанный каталог HDFS (--target-dir). Чтобы данные сразу стали доступны в Hive как таблица, можно: создать таблицу Hive вручную поверх каталога ([§6.2](chapter-06-02.md), [§6.3](chapter-06-03.md)) или использовать встроенную поддержку Sqoop для Hive. См. [Глоссарий](glossary.md).

**Опция --hive-import** — после импорта в HDFS Sqoop создаёт таблицу Hive (если её нет) и загружает метаданные: имя таблицы, схема по столбцам источника, расположение (каталог с выгруженными файлами). В результате данные оказываются в каталоге (как при обычном import) и одновременно регистрируются в Hive Metastore. Дополнительные опции:

- **--hive-table** — имя таблицы в Hive (по умолчанию совпадает с именем таблицы БД).
- **--hive-database** — база Hive (по умолчанию default).
- **--hive-overwrite** — перезаписать существующие данные в таблице (при повторном импорте).
- **--hive-drop-import-delims** — удалить символы, которые Hive трактует как разделители строк/полей внутри полей (например, переносы строк в текстовых полях), чтобы не ломать парсинг.

Пример:

```bash
sqoop import --connect jdbc:mysql://dbhost:3306/mydb --username user --password pass \
  --table orders --target-dir /data/import/orders \
  --hive-import --hive-table orders --hive-database raw
```

После выполнения в Hive будет таблица `raw.orders`, указывающая на каталог /data/import/orders; запросы HiveQL обращаются к ней как к обычной таблице. Формат файлов при --hive-import — текстовый с разделителями, совместимый с Hive (запятая по умолчанию); при необходимости задают --fields-terminated-by, чтобы совпадало с ожидаемым в Hive.

---

## 9.4.2. Импорт в HDFS и создание таблицы Hive вручную

Альтернатива --hive-import — **импорт только в HDFS** (обычный sqoop import с --target-dir), затем **создание внешней таблицы Hive** с LOCATION на этот каталог. Подход гибче: можно задать формат (STORED AS PARquet, ORC), партиции, свою схему. После импорта в каталог выполняют, например:

```sql
CREATE EXTERNAL TABLE raw.orders (id BIGINT, user_id INT, amount DOUBLE, dt STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/import/orders';
```

Либо при импорте в Parquet (--as-parquetfile) создают таблицу с STORED AS PARQUET и LOCATION на каталог вывода Sqoop. Так Sqoop отвечает только за выгрузку из БД в HDFS, а схема и формат в Hive задаются отдельно.

---

## 9.4.3. Sqoop в Oozie workflow

Sqoop часто вызывается как **шаг пайплайна** из Oozie ([§8.2](chapter-08-02.md)): сначала импорт из БД (Sqoop action), затем обработка (Pig, Hive action) и при необходимости экспорт (ещё один Sqoop action). В workflow описание **Sqoop action** содержит конфигурацию, эквивалентную параметрам командной строки: connect, username, password (или password-file), command (import или export) и аргументы (--table, --target-dir, --hive-import и т.д.). См. [Глоссарий](glossary.md).

Пример фрагмента workflow (идея):

```xml
<action name="sqoop_import">
  <sqoop xmlns="uri:oozie:sqoop-action:1.0">
    <job-tracker>${jobTracker}</job-tracker>
    <name-node>${nameNode}</name-node>
    <command>import --connect ${jdbcUrl} --username ${dbUser} --password ${dbPassword} --table orders --target-dir ${targetDir} --hive-import --hive-table orders</command>
  </sqoop>
  <ok to="next_action"/>
  <error to="fail"/>
</action>
```

Переменные (jdbcUrl, targetDir, dbUser, dbPassword) задаются в job.properties при запуске workflow или **передаются из coordinator** ([§8.3](chapter-08-03.md)): например, targetDir=/data/import/orders/${date}, где ${date} вычисляется в coordinator (дата за вчера). Так один workflow обслуживает ежедневный импорт с актуальной датой в путях.

---

## 9.4.4. Типичный пайплайн: Sqoop → Hive/Pig → экспорт

Связка Sqoop, Hive и Oozie даёт полный цикл:

1. **Coordinator** по расписанию (например, раз в день) запускает workflow с параметром date.
2. **Sqoop action** импортирует из БД в HDFS (--target-dir /data/raw/orders/${date}) и при необходимости в таблицу Hive (--hive-import).
3. **Hive action** или **Pig action** выполняет обработку: агрегация, джойны, запись результата в каталог витрины или в другую таблицу Hive.
4. При необходимости **второй Sqoop action** экспортирует результат из HDFS в целевую БД.

Все шаги выполняются в заданном порядке; при падении одного переходят в узел ошибки (kill или уведомление). Параметризация по дате позволяет обрабатывать данные за каждый день без изменения определений workflow.

---

## Ключевое

- **Импорт в Hive:** опция **--hive-import** создаёт таблицу Hive поверх каталога импорта; --hive-table, --hive-database, --hive-overwrite; альтернатива — импорт в HDFS и ручное CREATE EXTERNAL TABLE с LOCATION.
- **Sqoop в Oozie:** Sqoop action в workflow с тегом &lt;command&gt; и параметрами; переменные из job.properties или из coordinator (дата, пути).
- **Пайплайн:** coordinator → workflow: Sqoop import (в HDFS/Hive) → Hive/Pig обработка → при необходимости Sqoop export; параметризация по дате.

На этом заканчивается глава 9 и основной материал книги по Hadoop и экосистеме. Термины собраны в [Глоссарии](glossary.md); для углубления полезны официальная документация Apache (Hadoop, Hive, Pig, Oozie, Sqoop) и литература по большим данным.
