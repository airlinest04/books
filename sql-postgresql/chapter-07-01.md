# §7.1 Агрегатные функции

В [§6.7](chapter-06-07.md) мы завершили главу о манипулировании данными. **Агрегатные функции** вычисляют одно значение из набора строк: количество, сумму, среднее, минимум, максимум и т.п. В этом разделе разберём основные функции COUNT, SUM, AVG, MIN, MAX, обработку NULL и пустого набора, ограничение на использование в WHERE и опцию FILTER. GROUP BY и HAVING рассматриваются в [§7.2](chapter-07-02.md) и [§7.3](chapter-07-03.md). См. [документацию по агрегатным функциям](https://www.postgresql.org/docs/current/functions-aggregate.html) и [туториал](https://www.postgresql.org/docs/current/tutorial-agg.html).

---

## 7.1.1. Что такое агрегатная функция

Агрегатная функция принимает множество значений (все строки таблицы или группы) и возвращает одно. Типичные примеры:
- **COUNT** — число строк или непустых значений;
- **SUM** — сумма;
- **AVG** — среднее арифметическое;
- **MIN**, **MAX** — минимум и максимум.

Без GROUP BY вся таблица (после WHERE) рассматривается как одна группа; результат — одна строка. С GROUP BY агрегат вычисляется отдельно для каждой группы (см. [§7.2](chapter-07-02.md)).

---

## 7.1.2. COUNT

**COUNT(\*)** возвращает число строк в группе. Строки с NULL во всех столбцах тоже учитываются.

**COUNT(выражение)** возвращает число строк, в которых выражение не NULL.

```sql
SELECT count(*) FROM orders;                    -- всего заказов
SELECT count(status) FROM orders;               -- заказов с непустым status
SELECT count(DISTINCT customer_id) FROM orders; -- уникальных клиентов
```

`count(*)` даёт 0 для пустой таблицы; `count(столбец)` — тоже 0, если все значения NULL.

---

## 7.1.3. SUM и AVG

**SUM(выражение)** — сумма значений. NULL пропускается. Для пустого набора возвращает NULL.

**AVG(выражение)** — среднее арифметическое непустых значений. NULL не участвует в расчёте.

```sql
SELECT sum(total) AS total_revenue FROM orders;
SELECT avg(price) AS avg_price FROM products;
SELECT sum(quantity * price) AS order_total FROM order_items WHERE order_id = 1;
```

Для типа `integer` и `bigint` SUM возвращает `bigint`; для `numeric` — `numeric`; для `real`/`double precision` — соответствующий тип. AVG для целых возвращает `numeric`.

---

## 7.1.4. MIN и MAX

**MIN(выражение)** и **MAX(выражение)** возвращают минимальное и максимальное значение. NULL пропускается. Поддерживаются числовые, строковые, даты, интервалы и другие сравниваемые типы.

```sql
SELECT min(price), max(price) FROM products;
SELECT min(created_at) AS first_order, max(created_at) AS last_order FROM orders;
```

Для пустого набора MIN и MAX возвращают NULL.

---

## 7.1.5. Обработка NULL

| Функция | Поведение при NULL |
|---------|---------------------|
| COUNT(*) | Учитывает все строки, включая с NULL |
| COUNT(столбец) | Игнорирует NULL |
| SUM, AVG, MIN, MAX | Игнорируют NULL |

Пример: в таблице 3 строки, в столбце `x` значения 10, NULL, 20. Тогда `count(*) = 3`, `count(x) = 2`, `sum(x) = 30`, `avg(x) = 15`.

---

## 7.1.6. Пустой набор строк

Если после применения WHERE не остаётся ни одной строки:
- **COUNT** возвращает 0;
- **SUM, AVG, MIN, MAX** возвращают NULL.

Чтобы получить 0 вместо NULL для суммы, используйте COALESCE:

```sql
SELECT coalesce(sum(total), 0) AS total FROM orders WHERE customer_id = 99999;
```

---

## 7.1.7. Агрегаты нельзя использовать в WHERE

Условие WHERE вычисляется **до** агрегации: оно отбирает строки, которые попадут в расчёт. Агрегат вычисляется **после** отбора. Поэтому агрегатную функцию в WHERE указывать нельзя.

Ошибочно:

```sql
SELECT city FROM weather WHERE temp_lo = max(temp_lo);  -- ошибка
```

Правильно — подзапрос:

```sql
SELECT city FROM weather WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
```

Для фильтрации по результату агрегации используется HAVING (см. [§7.3](chapter-07-03.md)).

---

## 7.1.8. FILTER — фильтр для одного агрегата

Опция **FILTER (WHERE условие)** ограничивает строки, участвующие только в этом агрегате:

```sql
SELECT
  count(*) AS total,
  count(*) FILTER (WHERE status = 'shipped') AS shipped,
  count(*) FILTER (WHERE status = 'pending') AS pending
FROM orders;
```

Каждый агрегат с FILTER учитывает только строки, удовлетворяющие его условию; остальные агрегаты в том же SELECT работают со своими наборами строк.

---

## 7.1.9. Несколько агрегатов и псевдонимы

В одном SELECT можно использовать несколько агрегатов:

```sql
SELECT
  count(*) AS order_count,
  sum(total) AS total_amount,
  avg(total) AS avg_order,
  min(created_at) AS first_order,
  max(created_at) AS last_order
FROM orders;
```

Псевдонимы (AS) облегчают чтение результата. Агрегаты можно комбинировать с обычными столбцами только при наличии GROUP BY (см. [§7.2](chapter-07-02.md)); без GROUP BY в SELECT допускаются только агрегаты и константы.

---

## 7.1.10. Другие агрегатные функции

PostgreSQL поддерживает и другие агрегаты (по [документации](https://www.postgresql.org/docs/current/functions-aggregate.html)):
- **string_agg** — объединение строк с разделителем;
- **array_agg** — сбор значений в массив;
- **bool_and**, **bool_or** — логическое И и ИЛИ для булевых значений;
- **stddev**, **variance** — статистические функции;
- **percentile_cont**, **percentile_disc** — перцентили (с особым синтаксисом WITHIN GROUP).

Их использование по мере необходимости; базовые COUNT, SUM, AVG, MIN, MAX покрывают большинство задач.

---

## Ключевое

- **Агрегатная функция** — вычисляет одно значение из набора строк (COUNT, SUM, AVG, MIN, MAX и др.).
- Без GROUP BY вся таблица — одна группа; результат — одна строка.
- **COUNT(\*)** — число строк; **COUNT(столбец)** — число непустых значений. SUM, AVG, MIN, MAX игнорируют NULL.
- Для пустого набора: COUNT даёт 0; SUM, AVG, MIN, MAX — NULL.
- Агрегаты нельзя использовать в WHERE; для фильтрации по агрегату — подзапрос или HAVING.
- **FILTER (WHERE условие)** — ограничение строк только для данного агрегата.

В [§7.2](chapter-07-02.md) рассмотрим GROUP BY — разбиение на группы и агрегацию по каждой группе.
