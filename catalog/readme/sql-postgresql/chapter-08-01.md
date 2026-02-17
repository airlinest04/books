# §8.1 Столбцы не из GROUP BY в SELECT

В [§7.4](chapter-07-04.md) мы рассмотрели порядок выполнения запроса. Типичная ошибка при группировке — указать в SELECT столбец, который не входит в GROUP BY и не обёрнут в агрегатную функцию. PostgreSQL выдаёт ошибку, поскольку для группы такое значение не определено однозначно. В этом разделе разберём текст ошибки, причины, способы исправления и случаи, когда PostgreSQL допускает «лишние» столбцы (функциональная зависимость). Фильтрация по агрегату в WHERE — в [§8.2](chapter-08-02.md). См. [документацию SELECT](https://www.postgresql.org/docs/current/sql-select.html) и [Table Expressions](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPBY).

---

## 8.1.1. Текст ошибки

При нарушении правила возникает ошибка:

```text
ERROR:  column "products.name" must appear in the GROUP BY clause or be used in an aggregate function
LINE 1: SELECT category_id, name, count(*) FROM products GROUP BY ca...
```

Иногда сообщение указывает на позицию (порядковый номер столбца) или на конкретное выражение. Суть одна: в SELECT использован столбец (или выражение), который не входит в GROUP BY и не является аргументом агрегата.

---

## 8.1.2. Почему возникает ошибка

При GROUP BY строки объединяются в группы по одинаковым значениям столбцов группировки. В одной группе может быть несколько строк с разными значениями незагруппированного столбца. PostgreSQL не выбирает значение «произвольно» — требуется явное указание: либо столбец участвует в группировке (тогда у группы одно значение), либо результат получают через агрегат (MIN, MAX, ANY_VALUE и т.п.).

Пример: `products` с полями `category_id`, `name`, `price`. Группировка по `category_id`:

```sql
SELECT category_id, name, count(*) FROM products GROUP BY category_id;  -- ошибка
```

В одной категории несколько товаров с разными `name`. Какой `name` выводить — не определено. Запрос семантически некорректен.

---

## 8.1.3. Способы исправления

**1. Добавить столбец в GROUP BY:**

```sql
SELECT category_id, name, count(*)
FROM products
GROUP BY category_id, name;
```

Тогда группа — пара (category_id, name); в результате будет строка на каждую такую пару. Это меняет смысл запроса: вместо «по категории» получается «по категории и имени».

**2. Обернуть в агрегатную функцию:**

Если нужно одно значение на группу, используйте MIN, MAX, ANY_VALUE или другую агрегатную функцию:

```sql
SELECT category_id, min(name) AS sample_name, count(*)
FROM products
GROUP BY category_id;
```

`min(name)` и `max(name)` дают лексикографический минимум/максимум; для «любого» значения в стандартном SQL — `any_value(name)` (PostgreSQL 17+). В более старых версиях — `min(name)` или `(array_agg(name))[1]`.

**3. Убрать столбец из SELECT:**

Если столбец не нужен в выводе — просто удалите его:

```sql
SELECT category_id, count(*)
FROM products
GROUP BY category_id;
```

---

## 8.1.4. Функциональная зависимость (PostgreSQL)

Если столбец **функционально зависит** от столбцов GROUP BY (например, однозначно определяется первичным ключом), PostgreSQL позволяет не включать его в GROUP BY. См. [§7.2](chapter-07-02.md).

Пример: `product_id` — первичный ключ, от него зависят `name` и `price`:

```sql
SELECT product_id, name, sum(quantity)
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY product_id;
```

`name` не в GROUP BY, но функционально зависит от `product_id`, поэтому запрос допустим. В других СУБД пришлось бы писать `GROUP BY product_id, name`.

Функциональная зависимость работает для первичного ключа и уникальных ограничений; для произвольных столбцов — нет.

---

## 8.1.5. Ошибка после JOIN

При группировке по результату JOIN в GROUP BY должны быть все неагрегируемые столбцы из SELECT (с учётом функциональной зависимости):

```sql
-- Ошибка: c.name не в GROUP BY и не в агрегате
SELECT c.id, c.name, count(o.id)
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id;
```

Если `c.id` — первичный ключ, то `c.name` от него функционально зависит; в PostgreSQL такой запрос допустим. Если бы группировали только по `c.name`, пришлось бы добавить `c.id` в GROUP BY (или использовать `GROUP BY c.id, c.name`), иначе возможна ошибка.

---

## 8.1.6. ORDER BY и псевдонимы

В ORDER BY можно ссылаться на столбцы, не входящие в GROUP BY, если они заданы в SELECT (в том числе через агрегат или выражение):

```sql
SELECT category_id, count(*) AS cnt
FROM products
GROUP BY category_id
ORDER BY cnt DESC;
```

`cnt` — псевдоним для `count(*)`; в ORDER BY он допустим, поскольку ORDER BY выполняется после SELECT. Сортировка по незагруппированному «сырому» столбцу, не представленному в SELECT, в стандарте может быть ограничена; в PostgreSQL обычно работает, если столбец однозначно определяется группой.

---

## Ключевое

- Ошибка «must appear in the GROUP BY clause or be used in an aggregate function» возникает при использовании в SELECT столбца, не входящего в GROUP BY и не обёрнутого в агрегат.
- Исправления: добавить столбец в GROUP BY, обернуть в агрегат (MIN, MAX, ANY_VALUE) или убрать из SELECT.
- В PostgreSQL допустима функциональная зависимость: столбцы, однозначно определяемые первичным ключом в GROUP BY, можно не включать в GROUP BY.
- После JOIN проверяйте, что все неагрегируемые столбцы из SELECT либо в GROUP BY, либо функционально зависят от него.

В [§8.2](chapter-08-02.md) разберём ошибку: использование агрегата в WHERE вместо HAVING.
