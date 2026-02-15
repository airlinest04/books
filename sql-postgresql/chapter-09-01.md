# §9.1 Подзапросы в FROM

В [§8.3](chapter-08-03.md) мы использовали подзапрос в FROM для вложенной агрегации. **Подзапрос в FROM** (производная таблица, derived table) — это SELECT в скобках, результат которого используется как источник данных для внешнего запроса. В этом разделе разберём синтаксис, обязательный псевдоним, связывание с внешним запросом и типичные сценарии. Подзапросы в SELECT и WHERE — в [§9.2](chapter-09-02.md). См. [документацию Table Expressions](https://www.postgresql.org/docs/current/queries-table-expressions.html) и [FROM](https://www.postgresql.org/docs/current/sql-select.html).

---

## 9.1.1. Синтаксис

Подзапрос в FROM заключается в скобки и получает **псевдоним** (алиас). По [документации](https://www.postgresql.org/docs/current/queries-table-expressions.html) PostgreSQL допускает опустить псевдоним в простых случаях, но стандарт SQL и хорошая практика требуют его указывать:

```sql
SELECT *
FROM (SELECT id, name, price FROM products WHERE price > 100) AS expensive_products;
```

Подзапрос возвращает набор строк со столбцами `id`, `name`, `price`. Внешний запрос обращается к нему через псевдоним; столбцы можно квалифицировать: `expensive_products.name`.

---

## 9.1.2. Псевдонимы столбцов

Если в подзапросе используются вычисляемые столбцы без имён или нужно переименовать столбцы, их задают в скобках после псевдонима таблицы:

```sql
SELECT sub.category_id, sub.order_count
FROM (
  SELECT category_id, count(*) AS order_count
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p ON p.id = oi.product_id
  GROUP BY category_id
) AS sub
WHERE sub.order_count > 5;
```

Здесь `sub` — псевдоним подзапроса, `order_count` задан в самом подзапросе. Альтернатива — указать список столбцов после псевдонима: `) AS sub(category_id, order_count)`.

---

## 9.1.3. Подзапрос как «виртуальная таблица»

Подзапрос в FROM может содержать любые допустимые в SELECT конструкции: WHERE, JOIN, GROUP BY, HAVING, ORDER BY (порядок в подзапросе не гарантируется во внешнем результате, если нет ORDER BY снаружи). Результат подзапроса используется как обычная таблица.

Пример: агрегация по категориям, затем фильтрация и сортировка снаружи:

```sql
SELECT * FROM (
  SELECT category_id, count(*) AS cnt, sum(price) AS total
  FROM products
  GROUP BY category_id
) stats
WHERE stats.cnt >= 3
ORDER BY stats.total DESC;
```

---

## 9.1.4. JOIN с подзапросом

Подзапрос может участвовать в JOIN наравне с таблицами:

```sql
SELECT c.name, stats.order_count, stats.total_spent
FROM customers c
JOIN (
  SELECT customer_id, count(*) AS order_count, sum(total) AS total_spent
  FROM orders
  GROUP BY customer_id
) stats ON stats.customer_id = c.id
WHERE stats.order_count > 2;
```

Подзапрос даёт статистику по клиентам; внешний запрос соединяет её с таблицей `customers` и добавляет фильтр.

---

## 9.1.5. Несколько подзапросов в FROM

В FROM может быть несколько подзапросов, соединённых через запятую или JOIN:

```sql
SELECT a.id, a.name, b.avg_price
FROM (
  SELECT id, name, category_id FROM products WHERE active = true
) a
JOIN (
  SELECT category_id, avg(price) AS avg_price
  FROM products
  GROUP BY category_id
) b ON b.category_id = a.category_id;
```

---

## 9.1.6. Вложенные агрегаты — повторение

Подзапрос в FROM — стандартный способ получить «агрегат от агрегата» (см. [§8.3](chapter-08-03.md)):

```sql
SELECT max(order_count) AS max_orders
FROM (
  SELECT customer_id, count(*) AS order_count
  FROM orders
  GROUP BY customer_id
) sub;
```

Внутренний запрос считает число заказов по клиентам; внешний находит максимум среди этих значений.

---

## 9.1.7. Зачем использовать подзапрос в FROM

- **Разбиение сложной логики** — многошаговые вычисления проще читать и отлаживать.
- **Вложенная агрегация** — когда нужен агрегат от результата другого агрегата.
- **Переиспользование результата** — один подзапрос можно использовать в нескольких JOIN (при необходимости лучше CTE, см. гл. 13).
- **Изоляция группировки** — сначала получаем агрегированные данные, затем фильтруем, соединяем или сортируем снаружи.

---

## Ключевое

- **Подзапрос в FROM** — SELECT в скобках, результат используется как таблица; обязательно указывать псевдоним.
- Подзапрос может содержать WHERE, JOIN, GROUP BY, HAVING; к нему применяются те же правила, что и к обычному SELECT.
- Подзапрос участвует в JOIN как обычная таблица.
- Типичные задачи: вложенная агрегация, многошаговые запросы, предварительная фильтрация или агрегация перед соединением.

В [§9.2](chapter-09-02.md) рассмотрим подзапросы в SELECT (скалярные) и в WHERE (IN, EXISTS, сравнение).
