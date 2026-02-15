# §7.3 HAVING

В [§7.2](chapter-07-02.md) мы рассмотрели GROUP BY. **HAVING** отбирает группы после группировки и вычисления агрегатов. Условие HAVING может использовать результаты агрегатных функций и столбцы из GROUP BY. В этом разделе разберём синтаксис HAVING, отличие от WHERE, типичные примеры и комбинирование условий. Порядок выполнения предложений — в [§7.4](chapter-07-04.md). Ошибка фильтрации по агрегату в WHERE — в [§8.2](chapter-08-02.md). См. [документацию Table Expressions](https://www.postgresql.org/docs/current/queries-table-expressions.html#QUERIES-GROUPBY).

---

## 7.3.1. Назначение HAVING

**WHERE** отбирает строки **до** группировки: работает с отдельными строками таблицы. **HAVING** отбирает группы **после** группировки: работает с уже посчитанными агрегатами и значениями столбцов групп.

Если нужно оставить только те группы, у которых, например, `count(*) > 5` или `sum(total) > 1000`, условие задают в HAVING. В WHERE агрегаты использовать нельзя.

---

## 7.3.2. Синтаксис

```sql
SELECT столбцы_и_агрегаты
FROM таблица
[WHERE условие]
GROUP BY столбец1 [, столбец2, ...]
HAVING условие_для_групп
[ORDER BY ...];
```

HAVING идёт после GROUP BY. Условие — булево выражение: агрегаты, столбцы из GROUP BY, константы, логические операторы AND, OR, NOT.

---

## 7.3.3. Условия по результатам агрегации

Типичные примеры — отбор групп по count, sum, avg, min, max:

```sql
-- Клиенты с более чем 3 заказами
SELECT customer_id, count(*) AS order_count
FROM orders
GROUP BY customer_id
HAVING count(*) > 3;

-- Категории с суммой продаж больше 10000
SELECT category_id, sum(price * quantity) AS revenue
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY category_id
HAVING sum(price * quantity) > 10000;

-- Категории со средним чеком от 500
SELECT category_id, avg(total) AS avg_order
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY category_id
HAVING avg(total) >= 500;
```

---

## 7.3.4. Условия по столбцам групп

В HAVING можно ссылаться на столбцы из GROUP BY — у каждой группы одно значение:

```sql
SELECT x, sum(y) FROM test1 GROUP BY x HAVING x < 'c';
```

Такой же результат даёт `WHERE x < 'c'`, если `x` участвует в группировке: условие по столбцу группы можно ставить и в WHERE (до группировки), и в HAVING (после). Для производительности лучше WHERE — меньше строк участвует в группировке. HAVING нужен, когда условие зависит от **агрегата**.

---

## 7.3.5. Комбинирование условий

В HAVING можно задавать несколько условий через AND и OR:

```sql
SELECT customer_id, count(*) AS cnt, sum(total) AS total_spent
FROM orders
GROUP BY customer_id
HAVING count(*) >= 2 AND sum(total) > 500;
```

С учётом NULL: если агрегат возвращает NULL (например, группа пуста), сравнение даёт NULL, и строка отбрасывается. При необходимости явно обрабатывайте NULL (например, `coalesce(sum(total), 0) > 500`).

---

## 7.3.6. Псевдонимы в HAVING

В стандартном SQL в HAVING нельзя использовать псевдонимы из SELECT — HAVING выполняется до вычисления выходных выражений. Нужно повторять выражение:

```sql
SELECT category_id, count(*) AS cnt
FROM products
GROUP BY category_id
HAVING count(*) > 5;   -- не HAVING cnt > 5
```

В PostgreSQL в HAVING допустимы только те псевдонимы, которые разворачиваются в выражения, доступные на этапе группировки. Надёжнее везде использовать полное выражение.

---

## 7.3.7. HAVING без GROUP BY

Технически HAVING может использоваться без GROUP BY: вся таблица считается одной группой. Тогда HAVING ведёт себя как фильтр по одному агрегированному результату:

```sql
SELECT sum(total) FROM orders HAVING sum(total) > 1000;
```

Такой стиль встречается редко; для одного агрегата обычно оборачивают в подзапрос и фильтруют во внешнем WHERE.

---

## Ключевое

- **HAVING** отбирает группы после GROUP BY; может использовать агрегаты и столбцы групп.
- **WHERE** — до группировки, **HAVING** — после; для фильтрации по агрегату нужен HAVING.
- В HAVING допустимы условия по результатам COUNT, SUM, AVG, MIN, MAX и по столбцам из GROUP BY.
- Несколько условий задаются через AND и OR.
- В HAVING надёжнее не использовать псевдонимы из SELECT, а писать полные выражения.

В [§7.4](chapter-07-04.md) рассмотрим порядок выполнения предложений SELECT: FROM, WHERE, GROUP BY, HAVING, SELECT, ORDER BY, LIMIT.
