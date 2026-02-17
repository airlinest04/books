# §13.2 Читаемость и поддержка

В [§13.1](chapter-13-01.md) мы рассмотрели синтаксис CTE. Вложенные подзапросы часто трудно читать и отлаживать. **CTE** позволяют вынести каждый уровень в отдельное именованное выражение и собирать запрос «сверху вниз» — от простых шагов к сложному результату. В этом разделе сравним вложенные подзапросы и CTE, покажем переписывание и преимущества для поддержки. Рекурсивные CTE — в [§13.3](chapter-13-03.md). См. [документацию WITH Queries](https://www.postgresql.org/docs/current/queries-with.html).

---

## 13.2.1. Проблема вложенных подзапросов

Многоуровневые подзапросы читаются «изнутри наружу»: сначала нужно разобрать самый внутренний SELECT, затем следующий и т.д. Скобки и отступы усложняют обзор.

Пример: топ-2 сотрудника по зарплате в каждом отделе. Вариант с подзапросом:

```sql
SELECT depname, empno, salary, rn
FROM (
  SELECT depname, empno, salary,
    row_number() OVER (PARTITION BY depname ORDER BY salary DESC) AS rn
  FROM empsalary
) sub
WHERE rn <= 2;
```

При двух уровнях ещё терпимо. При трёх и более вложенностях разбор усложняется.

---

## 13.2.2. Переписывание на CTE

Тот же запрос в виде CTE:

```sql
WITH ranked AS (
  SELECT depname, empno, salary,
    row_number() OVER (PARTITION BY depname ORDER BY salary DESC) AS rn
  FROM empsalary
)
SELECT depname, empno, salary, rn
FROM ranked
WHERE rn <= 2;
```

Логика та же. Разница: шаги разделены по именам. Сначала видно, что есть `ranked` — ранжированные строки; затем — выборка по `rn <= 2`. Порядок чтения — «сверху вниз».

---

## 13.2.3. Многошаговый пример

Запрос с тремя уровнями: агрегация по парам (клиент, категория), затем среднее по категориям, затем максимум этих средних. Вложенный вариант:

```sql
SELECT max(avg_orders) AS max_of_avg
FROM (
  SELECT category_id, avg(order_count) AS avg_orders
  FROM (
    SELECT customer_id, category_id, count(*) AS order_count
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    JOIN products p ON p.id = oi.product_id
    GROUP BY customer_id, category_id
  ) per_cust_cat
  GROUP BY category_id
) per_cat;
```

С CTE:

```sql
WITH per_cust_cat AS (
  SELECT customer_id, category_id, count(*) AS order_count
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p ON p.id = oi.product_id
  GROUP BY customer_id, category_id
),
per_cat AS (
  SELECT category_id, avg(order_count) AS avg_orders
  FROM per_cust_cat
  GROUP BY category_id
)
SELECT max(avg_orders) AS max_of_avg
FROM per_cat;
```

Каждый шаг имеет понятное имя и легко читается по отдельности.

---

## 13.2.4. Отладка по шагам

CTE можно выполнять по частям: сначала проверить результат первого CTE, потом второго и т.д. Достаточно временно заменить основной запрос на `SELECT * FROM имя_cte` и выполнить весь блок WITH. Это упрощает отладку и проверку промежуточных данных.

---

## 13.2.5. Переиспользование CTE

Если один и тот же подзапрос нужен в нескольких местах основного запроса, CTE описан один раз и используется многократно. Без CTE пришлось бы дублировать подзапрос или строить обходные конструкции. CTE при многократном обращении вычисляется один раз (см. документацию по материализации).

---

## Ключевое

- Вложенные подзапросы читаются «изнутри наружу»; CTE — «сверху вниз».
- Каждый CTE — отдельный шаг с осмысленным именем; логика запроса становится прозрачнее.
- CTE удобны для отладки: можно проверять результат каждого шага по отдельности.
- CTE переиспользуются без дублирования; при нескольких обращениях вычисление выполняется один раз.

В [§13.3](chapter-13-03.md) рассмотрим рекурсивные CTE — базовый и рекурсивный члены, иерархии и последовательности.
