# §12.5 Агрегаты как оконные

В [§12.4](chapter-12-04.md) мы рассмотрели LAG, LEAD, FIRST_VALUE и LAST_VALUE. Обычные **агрегатные функции** (SUM, AVG, COUNT, MIN, MAX) тоже можно использовать как оконные — с предложением OVER. В этом разделе разберём бегущую сумму, нарастающий итог и скользящее среднее. Глава 13 — CTE ([§13.1](chapter-13-01.md)). См. [документацию Window Functions](https://www.postgresql.org/docs/current/functions-window.html) и [туториал](https://www.postgresql.org/docs/current/tutorial-window.html).

---

## 12.5.1. Агрегат с OVER — идея

Агрегат без OVER вычисляется по группе (GROUP BY) или по всей таблице и даёт одну строку. С **OVER** тот же агрегат считается по окну, и **каждая строка** получает своё значение.

```sql
SELECT depname, salary,
  sum(salary) OVER () AS total,
  avg(salary) OVER (PARTITION BY depname) AS avg_in_dept
FROM empsalary;
```

`total` — общая сумма во всех строках; `avg_in_dept` — среднее по отделу в каждой строке. Строки не сворачиваются.

---

## 12.5.2. Бегущая сумма (нарастающий итог)

При **ORDER BY** в OVER рамка по умолчанию — от начала партиции до текущей строки. Агрегат даёт **нарастающий итог**:

```sql
SELECT order_date, total,
  sum(total) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

В каждой строке `running_total` — сумма всех заказов от самого раннего до текущей даты включительно.

С PARTITION BY — свой нарастающий итог по каждому клиенту:

```sql
SELECT customer_id, order_date, total,
  sum(total) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total_per_customer
FROM orders;
```

---

## 12.5.3. Скользящее среднее (скользящее окно)

Чтобы считать агрегат не по всей партиции, а по **фиксированному числу строк** (например, последние 3 или 7), задают рамку вручную:

```sql
SELECT order_date, total,
  avg(total) OVER (ORDER BY order_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg_3
FROM orders;
```

`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` — текущая строка и две предыдущие. Итого 3 строки — скользящее среднее по 3 значениям.

Варианты рамки:

- `ROWS BETWEEN n PRECEDING AND m FOLLOWING` — от n строк выше до m строк ниже;
- `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — от начала партиции до текущей (значение по умолчанию при ORDER BY);
- `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` — вся партиция.

---

## 12.5.4. Агрегат по всей партиции при ORDER BY

Если задан ORDER BY, но нужен агрегат по **всей** партиции (одинаковое значение во всех строках), а не нарастающий итог, используют полную рамку:

```sql
SELECT depname, empno, salary,
  avg(salary) OVER (PARTITION BY depname ORDER BY salary ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS avg_in_dept
FROM empsalary;
```

`avg_in_dept` одинаков во всех строках отдела. Проще — убрать ORDER BY: `avg(salary) OVER (PARTITION BY depname)`.

---

## 12.5.5. FILTER в оконном агрегате

Оконные агрегаты поддерживают **FILTER** — учёт только строк, удовлетворяющих условию:

```sql
SELECT order_date, total,
  sum(total) OVER (ORDER BY order_date) AS running_total,
  sum(total) FILTER (WHERE total > 100) OVER (ORDER BY order_date) AS running_total_over_100
FROM orders;
```

Вторая сумма — нарастающий итог только по заказам с `total > 100`.

---

## Ключевое

- Агрегаты (SUM, AVG, COUNT и т.д.) с **OVER** работают как оконные — каждая строка сохраняется, добавляется результат по окну.
- С **ORDER BY** и рамкой по умолчанию получается **нарастающий итог** (бегущая сумма, нарастающее среднее).
- **ROWS BETWEEN n PRECEDING AND m FOLLOWING** — скользящее окно фиксированного размера.
- **ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING** — вся партиция; альтернатива — убрать ORDER BY.
- **FILTER (WHERE условие)** ограничивает строки, участвующие в расчёте агрегата.

В [§13.1](chapter-13-01.md) начнём главу о CTE — именованных подзапросах в начале запроса.
