# §12.4 Смещение

В [§12.3](chapter-12-03.md) мы рассмотрели функции ранжирования. Функции **смещения** возвращают значение из другой строки окна — предыдущей, следующей, первой или последней. В этом разделе разберём LAG, LEAD, FIRST_VALUE и LAST_VALUE. Агрегаты как оконные — в [§12.5](chapter-12-05.md). См. [документацию Window Functions](https://www.postgresql.org/docs/current/functions-window.html).

---

## 12.4.1. LAG и LEAD

**LAG(value [, offset [, default]])** — значение столбца из строки, расположенной на `offset` строк **выше** текущей в порядке ORDER BY окна. По умолчанию `offset = 1`, `default = NULL`.

**LEAD(value [, offset [, default]])** — значение из строки на `offset` строк **ниже** текущей.

ORDER BY в OVER обязателен — иначе порядок строк не определён.

Пример: заказы с суммой предыдущего заказа клиента:

```sql
SELECT customer_id, order_date, total,
  lag(total, 1) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_total
FROM orders;
```

Для первой строки партиции (первый заказ клиента) `prev_order_total` будет NULL. Можно задать свой default: `lag(total, 1, 0)` — тогда 0 вместо NULL.

Сравнение с предыдущим и следующим:

```sql
SELECT order_date, total,
  lag(total) OVER (ORDER BY order_date) AS prev,
  lead(total) OVER (ORDER BY order_date) AS next
FROM orders;
```

---

## 12.4.2. FIRST_VALUE и LAST_VALUE

**FIRST_VALUE(value)** — значение из **первой** строки окна (в порядке ORDER BY).

**LAST_VALUE(value)** — значение из **последней** строки окна. По умолчанию рамка окна заканчивается на текущей строке, поэтому LAST_VALUE для текущей строки часто возвращает значение текущей же строки, а не последней в партиции.

Чтобы получить значение из последней строки **всей** партиции, нужно явно задать рамку:

```sql
last_value(total) OVER (
  PARTITION BY customer_id
  ORDER BY order_date
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

---

## 12.4.3. Пример: разница с предыдущим

Типичная задача — разница между текущим и предыдущим значением:

```sql
SELECT order_date, total,
  total - lag(total) OVER (ORDER BY order_date) AS diff_from_prev
FROM orders;
```

Для первой строки `lag` даёт NULL, разница тоже NULL. При необходимости — `COALESCE(lag(total), 0)` или `FILTER` по `diff_from_prev IS NOT NULL` в подзапросе.

---

## 12.4.4. LAG/LEAD по нескольким строкам

`offset` может быть больше 1 — сдвиг на несколько строк:

```sql
SELECT order_date, total,
  lag(total, 2) OVER (ORDER BY order_date) AS two_orders_ago,
  lead(total, 3) OVER (ORDER BY order_date) AS three_orders_ahead
FROM orders;
```

Для строк у начала или конца партиции соответствующая строка отсутствует — возвращается `default` (NULL, если не задан).

---

## Ключевое

- **LAG(value [, offset [, default]])** — значение из строки на `offset` позиций выше; по умолчанию offset=1, default=NULL.
- **LEAD(value [, offset [, default]])** — значение из строки на `offset` позиций ниже.
- **FIRST_VALUE(value)** — значение из первой строки окна.
- **LAST_VALUE(value)** — из последней строки рамки; по умолчанию рамка до текущей строки; для последней в партиции нужна `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.
- Для LAG/LEAD в OVER обязателен ORDER BY.

В [§12.5](chapter-12-05.md) рассмотрим агрегаты как оконные функции — бегущая сумма, скользящее среднее.
