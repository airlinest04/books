# §13.1 Синтаксис WITH ... AS

В [§12.5](chapter-12-05.md) мы завершили главу об оконных функциях. **CTE** (Common Table Expression, обобщённое табличное выражение) — именованный подзапрос в начале запроса, результат которого можно использовать в основном запросе как таблицу. В этом разделе разберём базовый синтаксис WITH ... AS и несколько CTE через запятую. Читаемость и замена вложенных подзапросов — в [§13.2](chapter-13-02.md). См. [документацию WITH Queries](https://www.postgresql.org/docs/current/queries-with.html).

---

## 13.1.1. Базовый синтаксис

```sql
WITH имя AS (
  SELECT ...
)
SELECT ... FROM имя ...
```

`имя` — имя CTE, по которому к нему обращаются в основном запросе. В скобках — обычный SELECT (с JOIN, WHERE, GROUP BY и т.д.). Основной запрос может использовать CTE в FROM, JOIN, подзапросах.

Пример из [§9.3](chapter-09-03.md):

```sql
WITH customer_orders AS (
  SELECT customer_id, count(*) AS order_count
  FROM orders
  GROUP BY customer_id
)
SELECT max(order_count) AS max_orders, avg(order_count) AS avg_orders
FROM customer_orders;
```

---

## 13.1.2. Несколько CTE через запятую

Можно объявить несколько CTE подряд; каждое следующее может ссылаться на предыдущие:

```sql
WITH
  step1 AS (SELECT ...),
  step2 AS (SELECT ... FROM step1 ...),
  step3 AS (SELECT ... FROM step2 ...)
SELECT ... FROM step3 ...;
```

По [документации](https://www.postgresql.org/docs/current/queries-with.html) CTE вычисляются один раз; повторное использование имени в основном запросе не приводит к повторному выполнению.

---

## 13.1.3. Пример: многошаговая логика

Задача: вывести продажи по продуктам только в тех регионах, где общие продажи выше 10% от общей суммы. Без CTE пришлось бы писать вложенные подзапросы.

С CTE:

```sql
WITH regional_sales AS (
  SELECT region, sum(amount) AS total_sales
  FROM orders
  GROUP BY region
),
top_regions AS (
  SELECT region
  FROM regional_sales
  WHERE total_sales > (SELECT sum(total_sales) / 10 FROM regional_sales)
)
SELECT region, product, sum(quantity) AS units, sum(amount) AS sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

`regional_sales` — продажи по регионам; `top_regions` — регионы с долей выше 10%; основной запрос фильтрует по ним и группирует по региону и продукту.

---

## 13.1.4. Список столбцов в определении CTE

При необходимости можно явно задать имена столбцов:

```sql
WITH named_cols(col_a, col_b) AS (
  SELECT x, y FROM some_table
)
SELECT col_a FROM named_cols;
```

Список столбцов полезен, когда в подзапросе нет имён (например, результаты вычислений) или нужно переименовать.

---

## 13.1.5. CTE и основной запрос

CTE видно только в том запросе, к которому привязано предложение WITH. Его нельзя использовать в другом запросе в той же сессии. WITH может предварять SELECT, INSERT, UPDATE, DELETE, MERGE; использование с изменяющими командами — в [§13.4](chapter-13-04.md).

---

## Ключевое

- **WITH имя AS (SELECT ...)** — именованный подзапрос в начале запроса; используется как таблица в основном запросе.
- Несколько CTE перечисляются через запятую; каждое может ссылаться на предыдущие.
- CTE вычисляется один раз при многократном обращении.
- Список столбцов в скобках после имени задаёт имена столбцов результата.

В [§13.2](chapter-13-02.md) рассмотрим, как CTE улучшают читаемость и заменяют вложенные подзапросы.
