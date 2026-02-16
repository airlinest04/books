# §3.4 Профилирование в SQL

В [§3.3](chapter-03-03.md) мы разобрали выявление null, дубликатов и выбросов. В этом разделе соберём **практические запросы профилирования в SQL**: сводный профиль по полям таблицы, шаблоны для типовых задач и отличия синтаксиса в PostgreSQL и Oracle. Цель — иметь под рукой набор запросов, которые можно запустить по новой таблице или срезу и получить структурированный профиль для принятия решений о правилах валидации (глава 4) и проверках в ETL. Среда примеров — PostgreSQL 14+; для Oracle указаны отличия там, где они существенны.

---

## 3.4.1. Сводный профиль по полям (одна строка на поле)

Удобно получить по таблице **одну строку на каждое поле** с базовыми показателями: общее число строк, число не-null, доля null, для числовых — MIN/MAX/AVG, для строк — MIN/MAX длины и кардинальность. Такой запрос можно собрать вручную для фиксированного набора колонок или сформировать динамически (через информацию о схеме). Ниже — шаблон для таблицы с известным списком полей.

Пример: таблица `staging.orders` с полями `order_id`, `order_date`, `client_id`, `amount`, `status`. Сводка по каждому полю в одном наборе результатов можно получить через UNION по подзапросам (по одному полю в подзапросе) или через перечень агрегатов в одном SELECT. Вариант с отдельными показателями по полям в одном запросе (PostgreSQL):

```sql
-- Общие метрики по таблице
SELECT COUNT(*) AS total_rows
FROM staging.orders
WHERE load_date = CURRENT_DATE - 1;

-- По каждому полю: заполненность, null, для числа — min/max/avg, для строки — длина и кардинальность
SELECT 'order_id'   AS col, COUNT(*) AS filled, COUNT(*) - COUNT(order_id) AS nulls,
       NULL::bigint AS min_val, NULL::bigint AS max_val, NULL::double precision AS avg_val,
       MIN(LENGTH(order_id::text)) AS len_min, MAX(LENGTH(order_id::text)) AS len_max,
       COUNT(DISTINCT order_id) AS distinct_cnt
FROM staging.orders WHERE load_date = CURRENT_DATE - 1
UNION ALL
SELECT 'amount', COUNT(amount), COUNT(*) - COUNT(amount),
       MIN(amount)::bigint, MAX(amount)::bigint, AVG(amount),
       NULL, NULL, COUNT(DISTINCT amount)
FROM staging.orders WHERE load_date = CURRENT_DATE - 1;
```

На практике часто делают **отдельный запрос на каждое поле** по одному шаблону и объединяют результаты в отчёт (в скрипте или в инструменте). Универсальный шаблон для произвольного поля `column_name` (числовый тип):

```sql
SELECT COUNT(*) AS total,
       COUNT(column_name) AS filled,
       COUNT(*) - COUNT(column_name) AS nulls,
       ROUND(100.0 * (COUNT(*) - COUNT(column_name)) / NULLIF(COUNT(*), 0), 2) AS pct_null,
       MIN(column_name) AS min_val, MAX(column_name) AS max_val, AVG(column_name) AS avg_val
FROM schema.table_name
WHERE partition_filter;
```

Для строкового поля — заменить MIN/MAX/AVG на MIN(LENGTH(column)), MAX(LENGTH(column)), COUNT(DISTINCT column). Для даты — MIN(column), MAX(column). См. [§3.2](chapter-03-02.md), [§3.3](chapter-03-03.md).

---

## 3.4.2. Скрипт-последовательность для полного профиля

Полный профиль по одной таблице можно собрать **последовательностью запросов** и оформить как скрипт (файл .sql или выполнение из приложения). Рекомендуемый порядок:

1. **Объём:** `SELECT COUNT(*) FROM table WHERE filter;`
2. **Null по полям:** для каждого поля запрос вида «COUNT(*), COUNT(col), доля null» (см. §3.3.1).
3. **Дубликаты по ключу:** `GROUP BY key HAVING COUNT(*) > 1` и сводка «total_rows, unique_keys, duplicate_rows» (см. §3.3.2).
4. **Числовые поля:** MIN, MAX, AVG (и при необходимости STDDEV, перцентили) по каждому числовому полю (см. §3.2.1).
5. **Категориальные поля:** COUNT(DISTINCT col), топ-N по частоте (GROUP BY col ORDER BY COUNT(*) DESC LIMIT N) (см. §3.2.3).
6. **Гистограмма** по одному-двум ключевым полям (разбиение на интервалы) (см. §3.2.2).
7. **Выбросы:** при необходимости — запрос по IQR или доменному диапазону (см. §3.3.3, §3.3.4).

Параметризация: имя таблицы, условие фильтра (партиция, дата) и ключ уникальности вынести в переменные или в начало скрипта, чтобы один и тот же скрипт запускать для разных таблиц и срезов. Результаты можно выгружать в отчёт (CSV, Excel) или в таблицу метрик для истории профилирования.

---

## 3.4.3. Отличия в Oracle

Логика профилирования та же; отличия в синтаксисе Oracle:

- **Перцентили:** в Oracle — `PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY column)` поддерживается (Oracle 9i+). Аналог IQR: вычислить Q1 и Q3 через PERCENTILE_CONT, затем границы Q1 − 1.5*(Q3−Q1) и Q3 + 1.5*(Q3−Q1).
- **Типы и приведение:** явное приведение к строке для LENGTH — `LENGTH(TO_CHAR(column))` или LENGTH для строковых колонок; для чисел LENGTH(TO_CHAR(amount)).
- **LIMIT:** в Oracle нет LIMIT; используется `FETCH FIRST N ROWS ONLY` (12c+) или подзапрос с ROWNUM: `SELECT * FROM (SELECT ... ORDER BY cnt DESC) WHERE ROWNUM <= 10`.
- **FILTER (WHERE):** в Oracle нет агрегатного FILTER; вместо `COUNT(*) FILTER (WHERE condition)` использовать `SUM(CASE WHEN condition THEN 1 ELSE 0 END)` или условный COUNT в подзапросе.
- **Boolean / логические типы:** в Oracle нет типа boolean в SQL; условия в CASE WHEN.

Пример топ-10 по частоте в Oracle (12c+):

```sql
SELECT status, COUNT(*) AS cnt
FROM staging.orders
WHERE load_date = TRUNC(SYSDATE) - 1
GROUP BY status
ORDER BY COUNT(*) DESC
FETCH FIRST 10 ROWS ONLY;
```

Для версий до 12c — обернуть в подзапрос и использовать ROWNUM. Остальные шаблоны (COUNT, MIN, MAX, AVG, GROUP BY key HAVING COUNT(*) > 1) в Oracle совпадают с стандартным SQL. См. [Глоссарий](glossary.md).

---

## 3.4.4. Профиль по метаданным: список колонок и типов

Для **структурной** части профиля полезно вывести список колонок таблицы и их типы без сканирования данных. В PostgreSQL — из `information_schema.columns`:

```sql
SELECT column_name, data_type, is_nullable, character_maximum_length, numeric_precision
FROM information_schema.columns
WHERE table_schema = 'staging' AND table_name = 'orders'
ORDER BY ordinal_position;
```

В Oracle — из `USER_TAB_COLUMNS` (или ALL_TAB_COLUMNS, DBA_TAB_COLUMNS):

```sql
SELECT column_name, data_type, nullable, data_length, data_precision
FROM user_tab_columns
WHERE table_name = 'ORDERS'
ORDER BY column_id;
```

По этому списку составляют план: по каким полям считать числовые агрегаты, по каким — длину и кардинальность, по каким — даты; и задают ключ для проверки дубликатов. Связь структуры с содержимым — в [§3.1](chapter-03-01.md).

---

## 3.4.5. Типичные ошибки

- **Профилировать без фильтра по партиции/дате:** на больших таблицах полный scan может быть долгим и тяжёлым; для инкрементальных данных всегда ограничивать срез (load_date, partition key).
- **Смешивать типы в одном универсальном запросе:** попытка вывести MIN/MAX и для числа, и для строки в одном SELECT без разведения по типам ведёт к ошибкам приведения; надёжнее отдельные шаблоны для числовых, строковых и датовых полей.
- **Забыть про null при расчёте долей:** знаменатель — COUNT(*) или COUNT(column); при доле «валидных» от не-null явно указывать, от какого объёма считаем проценты.
- **Не документировать ключ и фильтр:** сохранённый результат профиля без указания таблицы, условия фильтра и ключа уникальности теряет смысл при повторном использовании; в скрипте или в отчёте фиксировать параметры запуска.

---

## Ключевое

- **Сводный профиль по полям:** один запрос или набор запросов, дающий по каждому полю: total, filled, nulls, pct_null; для чисел — MIN/MAX/AVG; для строк — длина и кардинальность; шаблоны параметризуются именем таблицы и фильтром.
- **Полный профиль** собирается последовательностью: объём → null по полям → дубликаты по ключу → статистики по полям → частота значений / гистограмма → выбросы при необходимости.
- **Oracle:** те же идеи; отличия — FETCH FIRST N ROWS ONLY или ROWNUM вместо LIMIT; SUM(CASE WHEN ...) вместо FILTER; перцентили и агрегаты совместимы.
- **Метаданные** (information_schema в PostgreSQL, user_tab_columns в Oracle) дают список колонок и типов для планирования запросов профиля.
- Результат профилирования в SQL — основа для формулирования правил валидации (глава 4) и для проектирования проверок в ETL (главы 5–7).

Мы завершили главу 3 о профилировании данных. В [§4.1](chapter-04-01.md) мы переходим к **правилам валидации**: типы правил (бизнес, технические, правила соответствия) и их место в системе проверок качества.
