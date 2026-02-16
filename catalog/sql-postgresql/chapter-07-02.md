# §7.2 GROUP BY

В [§7.1](chapter-07-01.md) мы рассмотрели агрегатные функции. **GROUP BY** объединяет строки с одинаковыми значениями в указанных столбцах в группы и вычисляет агрегаты для каждой группы. В результате — по одной строке на группу. В этом разделе разберём синтаксис GROUP BY, правило SELECT при группировке, группировку по выражению и по нескольким столбцам, а также связь с JOIN. HAVING рассматривается в [§7.3](chapter-07-03.md). Ошибки с незагруппированными столбцами — в [§8.1](chapter-08-01.md). См. [документацию SELECT](https://www.postgresql.org/docs/current/sql-select.html) и [Table Expressions](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPBY).

---

## 7.2.1. Зачем нужен GROUP BY

Без GROUP BY агрегат вычисляется по всем строкам таблицы — результат в одной строке. С GROUP BY строки разбиваются на группы по одинаковым значениям; для каждой группы вычисляется свой агрегат. Например: количество заказов и сумма продаж **по каждому клиенту** или **по каждой категории товара**.

---

## 7.2.2. Синтаксис и базовый пример

```sql
SELECT столбцы_и_агрегаты
FROM таблица
[WHERE условие]
GROUP BY столбец1 [, столбец2, ...]
[HAVING условие]
[ORDER BY ...];
```

Столбцы в GROUP BY задают группы. Строки с одинаковыми значениями по этим столбцам объединяются в одну группу; для каждой группы возвращается одна строка.

Пример по [документации](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPBY):

```sql
SELECT x, sum(y) FROM test1 GROUP BY x;
```

Если в `test1` есть строки (a,3), (c,2), (b,5), (a,1), то результат:

| x | sum |
|---|-----|
| a | 4   |
| b | 5   |
| c | 2   |

---

## 7.2.3. Правило SELECT при GROUP BY

В SELECT при наличии GROUP BY можно указывать только:

1. **Столбцы из GROUP BY** — у каждой группы одно значение;
2. **Выражения на основе столбцов из GROUP BY** (например, столбец как есть);
3. **Агрегатные функции** — результат по группе.

Столбцы, не входящие в GROUP BY и не обёрнутые в агрегат, использовать **нельзя**: у группы несколько строк, и неясно, какое значение брать. Ошибка такого рода разбирается в [§8.1](chapter-08-01.md).

Правильно:

```sql
SELECT category_id, count(*) AS cnt, sum(price) AS total
FROM products
GROUP BY category_id;
```

Неправильно (name не в GROUP BY и не в агрегате):

```sql
SELECT category_id, name, count(*) FROM products GROUP BY category_id;  -- ошибка
```

---

## 7.2.4. GROUP BY по нескольким столбцам

Группировка идёт по уникальным комбинациям значений:

```sql
SELECT customer_id, status, count(*) AS order_count
FROM orders
GROUP BY customer_id, status;
```

Результат: одна строка на каждую пару (customer_id, status). Порядок столбцов в GROUP BY на состав групп не влияет.

---

## 7.2.5. GROUP BY по выражению

В GROUP BY можно указывать не только столбцы, но и выражения. В SELECT должно быть то же выражение (или алиас, если допустим контекстом).

Пример — группировка по году и месяцу:

```sql
SELECT date_trunc('month', created_at) AS month, count(*) AS orders
FROM orders
GROUP BY date_trunc('month', created_at);
```

Или по категории и признаку «дорогой/дешёвый»:

```sql
SELECT category_id, (price > 100) AS expensive, count(*)
FROM products
GROUP BY category_id, (price > 100);
```

---

## 7.2.6. Функциональная зависимость (PostgreSQL)

В стандартном SQL в GROUP BY должны быть все неагрегируемые столбцы из SELECT. PostgreSQL допускает **функциональную зависимость**: если столбец однозначно определяется первичным ключом (или уникальным ограничением), его можно не указывать в GROUP BY.

Пример: `product_id` — первичный ключ, `name` и `price` от него зависят:

```sql
SELECT product_id, p.name, sum(s.units) * p.price AS sales
FROM products p
LEFT JOIN sales s USING (product_id)
GROUP BY product_id;
```

Здесь `p.name` и `p.price` функционально зависят от `product_id`, поэтому их можно использовать в SELECT без добавления в GROUP BY. В других СУБД пришлось бы писать `GROUP BY product_id, p.name, p.price`.

---

## 7.2.7. GROUP BY и JOIN

При группировке после JOIN в GROUP BY должны быть все неагрегируемые столбцы из SELECT (с учётом функциональной зависимости в PostgreSQL):

```sql
SELECT c.name, count(o.id) AS order_count, sum(o.total) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
```

Или при функциональной зависимости от первичного ключа:

```sql
SELECT c.id, c.name, count(o.id) AS order_count
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id;
```

LEFT JOIN обеспечивает, что клиенты без заказов тоже попадут в результат (с `count = 0`, `sum = NULL`).

---

## 7.2.8. WHERE и GROUP BY

WHERE выполняется **до** группировки: отбираются строки, которые потом делятся на группы. Условия по неагрегируемым полям задают в WHERE:

```sql
SELECT category_id, count(*) 
FROM products 
WHERE price > 10 
GROUP BY category_id;
```

Здесь сначала отфильтровываются товары с `price > 10`, затем выполняется группировка. Фильтрация **по результату агрегации** делается в HAVING (см. [§7.3](chapter-07-03.md)).

---

## 7.2.9. ORDER BY при группировке

ORDER BY применяется к результату запроса (уже после GROUP BY). Можно сортировать по столбцам групп и по агрегатам:

```sql
SELECT category_id, count(*) AS cnt, sum(price) AS total
FROM products
GROUP BY category_id
ORDER BY total DESC;
```

---

## Ключевое

- **GROUP BY** объединяет строки с одинаковыми значениями в группах; для каждой группы возвращается одна строка.
- В SELECT допустимы только столбцы из GROUP BY (или выражения от них) и агрегатные функции.
- Группировка по нескольким столбцам — по уникальным комбинациям значений.
- GROUP BY по выражению — то же выражение должно фигурировать в GROUP BY и в SELECT (или через алиас, где возможно).
- PostgreSQL допускает функциональную зависимость: столбцы, однозначно определяемые первичным ключом, можно не включать в GROUP BY.
- WHERE фильтрует строки до группировки; для фильтрации по агрегату используется HAVING.

В [§7.3](chapter-07-03.md) рассмотрим HAVING — фильтрацию групп по результатам агрегации.
