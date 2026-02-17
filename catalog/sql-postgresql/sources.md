# Список литературы

Источники, использованные при подготовке книги.

---

## Установка и администрирование PostgreSQL

- PostgreSQL. [Downloads](https://www.postgresql.org/download/). — Официальная страница загрузки: пакеты и установщики для Linux, Windows, macOS, BSD, Solaris.
- PostgreSQL. [Linux downloads (Ubuntu)](https://www.postgresql.org/download/linux/ubuntu/). — Инструкции по установке на Ubuntu через apt и официальный репозиторий.
- PostgreSQL. [Linux downloads (Red Hat)](https://www.postgresql.org/download/linux/redhat/). — Инструкции по установке на Red Hat, Rocky, AlmaLinux, Fedora.
- PostgreSQL. [Windows installers](https://www.postgresql.org/download/windows/). — Установщик EDB для Windows.
- PostgreSQL. [macOS packages](https://www.postgresql.org/download/macosx/). — Варианты установки на macOS (EDB, Postgres.app, Homebrew).
- PostgreSQL Documentation. [Client Authentication](https://www.postgresql.org/docs/current/client-authentication.html). — Аутентификация клиентов, pg_hba.conf.
- PostgreSQL Documentation. [psql](https://www.postgresql.org/docs/current/app-psql.html). — Мета-команды и работа с интерактивным клиентом.
- PostgreSQL Documentation. [CREATE DATABASE](https://www.postgresql.org/docs/current/sql-createdatabase.html). — Синтаксис и параметры создания базы данных.
- PostgreSQL Documentation. [CREATE SCHEMA](https://www.postgresql.org/docs/current/sql-createschema.html). — Создание схемы.
- PostgreSQL Documentation. [Schemas](https://www.postgresql.org/docs/current/ddl-schemas.html). — Схемы, search_path, public, pg_catalog.

## SQL-запросы и синтаксис

- PostgreSQL Documentation. [SELECT](https://www.postgresql.org/docs/current/sql-select.html). — Синтаксис SELECT, FROM, предложения.
- PostgreSQL Documentation. [Lexical Structure](https://www.postgresql.org/docs/current/sql-syntax-lexical.html). — Идентификаторы, ключевые слова, регистр, точка с запятой.
- PostgreSQL Documentation. [Functions and Operators](https://www.postgresql.org/docs/current/functions.html). — Обзор встроенных функций и операторов.
- PostgreSQL Documentation. [String Functions and Operators](https://www.postgresql.org/docs/current/functions-string.html). — Конкатенация, UPPER, LOWER, LENGTH, substring, trim и др.
- PostgreSQL Documentation. [Mathematical Functions](https://www.postgresql.org/docs/current/functions-math.html). — round, trunc, арифметика.
- PostgreSQL Documentation. [Date/Time Functions and Operators](https://www.postgresql.org/docs/current/functions-datetime.html). — now(), current_date, extract, date_trunc, interval.
- PostgreSQL Documentation. [Comparison Functions and Operators](https://www.postgresql.org/docs/current/functions-comparison.html). — =, <>, BETWEEN, IS NULL, IS DISTINCT FROM.
- PostgreSQL Documentation. [Logical Operators](https://www.postgresql.org/docs/current/functions-logical.html). — AND, OR, NOT, трёхзначная логика.
- PostgreSQL Documentation. [Pattern Matching](https://www.postgresql.org/docs/current/functions-matching.html). — LIKE, ILIKE, SIMILAR TO, регулярные выражения.
- PostgreSQL Documentation. [Subquery Expressions](https://www.postgresql.org/docs/current/functions-subquery.html). — IN, NOT IN, EXISTS, ANY, ALL.
- PostgreSQL Documentation. [Sorting Rows (ORDER BY)](https://www.postgresql.org/docs/current/queries-order.html). — ORDER BY, ASC, DESC, NULLS FIRST, NULLS LAST.
- PostgreSQL Documentation. [Combining Queries (UNION, INTERSECT, EXCEPT)](https://www.postgresql.org/docs/current/queries-union.html). — Синтаксис, приоритет операторов, скобки.

## Транзакции

- PostgreSQL Documentation. [BEGIN](https://www.postgresql.org/docs/current/sql-begin.html). — Начало транзакции.
- PostgreSQL Documentation. [COMMIT](https://www.postgresql.org/docs/current/sql-commit.html). — Подтверждение транзакции.
- PostgreSQL Documentation. [ROLLBACK](https://www.postgresql.org/docs/current/sql-rollback.html). — Отмена транзакции.
- PostgreSQL Documentation. [SAVEPOINT](https://www.postgresql.org/docs/current/sql-savepoint.html). — Точка сохранения, ROLLBACK TO.
- PostgreSQL Documentation. [Transactions (Tutorial)](https://www.postgresql.org/docs/current/tutorial-transactions.html). — Транзакции, атомарность, savepoints.
- PostgreSQL Documentation. [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html). — READ COMMITTED, REPEATABLE READ, SERIALIZABLE; аномалии.
- PostgreSQL Documentation. [Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html). — Глава о параллельном доступе.
- PostgreSQL Documentation. [13.1. Introduction (MVCC)](https://www.postgresql.org/docs/current/mvcc-intro.html). — MVCC, снимки данных, отсутствие блокировок чтения записью и наоборот.
- PostgreSQL Documentation. [SET TRANSACTION](https://www.postgresql.org/docs/current/sql-set-transaction.html). — Уровень изоляции, READ ONLY, DEFERRABLE.
- PostgreSQL Documentation. [Window Functions (Tutorial)](https://www.postgresql.org/docs/current/tutorial-window.html). — Введение в оконные функции, OVER, PARTITION BY.
- PostgreSQL Documentation. [Window Functions](https://www.postgresql.org/docs/current/functions-window.html). — ROW_NUMBER, RANK, LAG, LEAD и др.
- PostgreSQL Documentation. [Window Function Calls (Value Expressions)](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS). — Синтаксис OVER, PARTITION BY, ORDER BY, frame_clause.
- PostgreSQL Documentation. [WITH Queries (Common Table Expressions)](https://www.postgresql.org/docs/current/queries-with.html). — WITH ... AS, несколько CTE, рекурсия, изменение данных.

## Модель данных

- PostgreSQL Documentation. [CREATE TABLE](https://www.postgresql.org/docs/current/sql-createtable.html). — Создание таблиц, столбцы, ограничения.
- PostgreSQL Documentation. [DROP TABLE](https://www.postgresql.org/docs/current/sql-droptable.html). — Удаление таблиц.
- PostgreSQL Documentation. [Data Types](https://www.postgresql.org/docs/current/datatype.html). — Обзор встроенных типов.
- PostgreSQL Documentation. [Numeric Types](https://www.postgresql.org/docs/current/datatype-numeric.html). — Целые, numeric, real, serial.
- PostgreSQL Documentation. [Character Types](https://www.postgresql.org/docs/current/datatype-character.html). — VARCHAR, CHAR, TEXT.
- PostgreSQL Documentation. [Date/Time Types](https://www.postgresql.org/docs/current/datatype-datetime.html). — DATE, TIMESTAMP, INTERVAL.
- PostgreSQL Documentation. [Boolean Type](https://www.postgresql.org/docs/current/datatype-boolean.html). — Логический тип.
- PostgreSQL Documentation. [JSON Types](https://www.postgresql.org/docs/current/datatype-json.html). — JSON и JSONB.
- PostgreSQL Documentation. [Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html). — NOT NULL, UNIQUE, PRIMARY KEY, CHECK, FOREIGN KEY.
- PostgreSQL Documentation. [ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html). — Изменение структуры таблицы.

## Манипулирование данными

- PostgreSQL Documentation. [INSERT](https://www.postgresql.org/docs/current/sql-insert.html). — Вставка строк, VALUES, SELECT, RETURNING, ON CONFLICT.
- PostgreSQL Documentation. [UPDATE](https://www.postgresql.org/docs/current/sql-update.html). — Изменение строк, SET, WHERE, FROM, RETURNING.
- PostgreSQL Documentation. [DELETE](https://www.postgresql.org/docs/current/sql-delete.html). — Удаление строк, WHERE, USING, RETURNING.
- PostgreSQL Documentation. [MERGE](https://www.postgresql.org/docs/current/sql-merge.html). — Условная вставка, обновление и удаление; WHEN MATCHED, WHEN NOT MATCHED, merge_action().
- PostgreSQL Documentation. [CREATE VIEW](https://www.postgresql.org/docs/current/sql-createview.html). — Представления, обновляемые представления, WITH CHECK OPTION.
- PostgreSQL Documentation. [CREATE MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-creatematerializedview.html). — Материализованные представления, WITH DATA/NO DATA.
- PostgreSQL Documentation. [REFRESH MATERIALIZED VIEW](https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html). — Обновление материализованного представления, CONCURRENTLY.
- PostgreSQL Documentation. [DROP VIEW](https://www.postgresql.org/docs/current/sql-dropview.html). — Удаление представлений.
- PostgreSQL Documentation. [COPY](https://www.postgresql.org/docs/current/sql-copy.html). — Массовая загрузка и выгрузка, форматы text/CSV/binary, опции.
- PostgreSQL Documentation. [Aggregate Functions](https://www.postgresql.org/docs/current/functions-aggregate.html). — COUNT, SUM, AVG, MIN, MAX и др.
- PostgreSQL. [Tutorial: Aggregate Functions](https://www.postgresql.org/docs/current/tutorial-agg.html). — Введение в агрегаты, WHERE/HAVING, FILTER.
- PostgreSQL Documentation. [Table Expressions (GROUP BY, HAVING)](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPBY). — GROUP BY, правило SELECT, HAVING.
