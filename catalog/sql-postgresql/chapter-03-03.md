# §3.3 WHERE: условия отбора

В [§3.2](chapter-03-02.md) мы рассмотрели выражения и функции в SELECT. Чтобы ограничить результат запроса определёнными строками, используют предложение **WHERE** — оно задаёт условие фильтрации. В этом разделе разберём операторы сравнения, логические операторы AND/OR/NOT, проверку на NULL, шаблоны LIKE и ILIKE, а также IN и BETWEEN. Уникальные значения через DISTINCT рассматриваются в [§3.4](chapter-03-04.md). См. [документацию SELECT](https://www.postgresql.org/docs/current/sql-select.html) и [операторы сравнения](https://www.postgresql.org/docs/current/functions-comparison.html).

---

## 3.3.1. Синтаксис WHERE

Предложение WHERE задаёт условие отбора строк. Строки, для которых условие даёт FALSE или NULL, исключаются из результата; остаются только строки, где условие даёт TRUE. По [документации](https://www.postgresql.org/docs/current/sql-select.html#SQL-WHERE):

```sql
SELECT столбцы FROM таблица WHERE условие;
```

Условие — любое выражение типа boolean (логическое). В нём можно использовать столбцы таблицы, константы, функции и операторы.

```sql
SELECT id, name, price FROM products WHERE price > 100;
SELECT * FROM orders WHERE status = 'pending';
SELECT name FROM users WHERE created_at >= '2024-01-01';
```

WHERE применяется до вычисления агрегатов (если есть GROUP BY) и до ORDER BY, LIMIT. Точный порядок выполнения описан в документации SELECT.

---

## 3.3.2. Операторы сравнения

Стандартные операторы сравнения по [документации](https://www.postgresql.org/docs/current/functions-comparison.html):

| Оператор | Описание |
|----------|----------|
| `=` | Равно |
| `<>` или `!=` | Не равно |
| `<` | Меньше |
| `>` | Больше |
| `<=` | Меньше или равно |
| `>=` | Больше или равно |

`<>` — стандартная SQL-нотация «не равно», `!=` — синоним, преобразуемый в `<>` при разборе.

Примеры:

```sql
SELECT * FROM products WHERE price = 99.99;
SELECT * FROM orders WHERE total <> 0;
SELECT * FROM users WHERE age >= 18;
SELECT * FROM events WHERE event_date < current_date;
```

Работают для числовых, строковых и типов даты/времени с естественным порядком.

---

## 3.3.3. Логические операторы AND, OR, NOT

Для объединения условий используются логические операторы. По [документации](https://www.postgresql.org/docs/current/functions-logical.html):

- **AND** — оба условия истинны;
- **OR** — хотя бы одно условие истинно;
- **NOT** — отрицание условия.

SQL использует трёхзначную логику: TRUE, FALSE, NULL (неизвестно). При NULL в одном из операндов AND/OR результат может быть NULL.

Примеры:

```sql
SELECT * FROM products WHERE price > 50 AND price < 200;
SELECT * FROM orders WHERE status = 'pending' OR status = 'processing';
SELECT * FROM users WHERE NOT active;
SELECT * FROM events WHERE (status = 'open' OR status = 'tentative') AND capacity > 10;
```

Скобки задают приоритет: без них AND имеет более высокий приоритет, чем OR.

---

## 3.3.4. IS NULL и IS NOT NULL

Сравнение с NULL через `=` или `<>` не работает: результат всегда NULL (неизвестно), и строка не проходит фильтр WHERE. Для проверки на NULL нужно использовать специальные предикаты. По [документации](https://www.postgresql.org/docs/current/functions-comparison.html):

```sql
выражение IS NULL
выражение IS NOT NULL
```

`expression = NULL` в SQL **не** даёт TRUE даже при NULL в выражении; это типичная ошибка. Всегда используйте `IS NULL` и `IS NOT NULL`.

Примеры:

```sql
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM orders WHERE shipped_at IS NOT NULL;
SELECT * FROM products WHERE discount IS NULL OR discount = 0;
```

Синтаксис `ISNULL` и `NOTNULL` — нестандартный, но поддерживается PostgreSQL; рекомендуются `IS NULL` и `IS NOT NULL`.

---

## 3.3.5. LIKE: поиск по шаблону

LIKE проверяет, совпадает ли строка с шаблоном. Спецсимволы:

- `%` — любая последовательность символов (в том числе пустая);
- `_` — ровно один символ.

Шаблон сопоставляется **со всей строкой**, а не с подстрокой. По [документации](https://www.postgresql.org/docs/current/functions-matching.html):

```sql
строка LIKE шаблон [ESCAPE escape-символ]
строка NOT LIKE шаблон [ESCAPE escape-символ]
```

Примеры:

```sql
SELECT * FROM products WHERE name LIKE 'Apple%';       -- начинается с Apple
SELECT * FROM users WHERE email LIKE '%@gmail.com';    -- заканчивается на @gmail.com
SELECT * FROM products WHERE name LIKE '%phone%';      -- содержит phone
SELECT * FROM users WHERE name LIKE 'J_n';             -- J, любой символ, n
```

Без `%` и `_` LIKE ведёт себя как обычное сравнение на равенство.

---

## 3.3.6. Экранирование в LIKE

Чтобы искать символы `%` и `_` как обычные, их экранируют. По умолчанию escape-символ — обратный слэш:

```sql
SELECT * FROM logs WHERE message LIKE '%\%%';          -- содержит %
SELECT * FROM files WHERE name LIKE 'file\_%';        -- file_, затем что угодно
```

Можно задать свой escape-символ через `ESCAPE`:

```sql
SELECT * FROM logs WHERE path LIKE '%#_%' ESCAPE '#';
```

---

## 3.3.7. ILIKE: поиск без учёта регистра

ILIKE — расширение PostgreSQL; ведёт себя как LIKE, но без учёта регистра (по активной локали):

```sql
SELECT * FROM users WHERE name ILIKE 'john%';
SELECT * FROM products WHERE name ILIKE '%phone%';
```

`ILIKE` не входит в стандарт SQL. В других СУБД часто используют `LOWER(column) LIKE LOWER(pattern)`.

---

## 3.3.8. IN: принадлежность списку

IN проверяет, входит ли значение в список. Список — скобки с перечислением через запятую или подзапрос:

```sql
expression IN (значение1, значение2, ...)
expression IN (подзапрос)
```

Примеры:

```sql
SELECT * FROM orders WHERE status IN ('pending', 'processing', 'shipped');
SELECT * FROM products WHERE category_id IN (1, 5, 7);
SELECT * FROM users WHERE id IN (SELECT user_id FROM admins);
```

`NOT IN` — значение не входит в список:

```sql
SELECT * FROM products WHERE status NOT IN ('archived', 'deleted');
```

При NULL в списке `IN` и `NOT IN` ведут себя по трёхзначной логике; см. подраздел про NULL ниже.

---

## 3.3.9. BETWEEN: диапазон значений

BETWEEN проверяет, попадает ли значение в диапазон (границы включительно). По [документации](https://www.postgresql.org/docs/current/functions-comparison.html):

```sql
expression BETWEEN низ AND верх
expression NOT BETWEEN низ AND верх
```

Эквивалентно `expression >= низ AND expression <= верх`. Границы должны быть заданы в правильном порядке (низ <= верх); иначе можно использовать `BETWEEN SYMMETRIC`.

Примеры:

```sql
SELECT * FROM products WHERE price BETWEEN 50 AND 100;
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
SELECT * FROM users WHERE age NOT BETWEEN 0 AND 17;
```

---

## 3.3.10. Приоритет операторов и скобки

Приоритет (от высшего к низшему):

1. `NOT`
2. `AND`
3. `OR`

Скобки меняют порядок вычисления:

```sql
SELECT * FROM t WHERE a = 1 OR b = 2 AND c = 3;   -- то же, что a = 1 OR (b = 2 AND c = 3)
SELECT * FROM t WHERE (a = 1 OR b = 2) AND c = 3; -- другое условие
```

При сомнениях лучше явно расставлять скобки.

---

## 3.3.11. NULL в условиях

В трёхзначной логике SQL условие отсекает строку, если результат — FALSE или NULL. Только TRUE оставляет строку.

Сравнения с NULL:

- `7 = NULL` → NULL (не TRUE);
- `7 <> NULL` → NULL (не TRUE);
- `NULL = NULL` → NULL.

Поэтому `WHERE column = NULL` не найдёт строк с NULL; нужно `WHERE column IS NULL`.

**IN и NULL:** если левая часть — NULL или в списке есть NULL и совпадений нет, результат IN может быть NULL. `NOT IN` с NULL в списке тоже может дать NULL или неожиданный результат; при работе с возможными NULL часто удобнее `NOT EXISTS` или явная обработка (см. гл. 9).

---

## 3.3.12. IS DISTINCT FROM и IS NOT DISTINCT FROM

Для сравнения с учётом NULL (NULL как обычное значение) используются предикаты:

```sql
a IS DISTINCT FROM b   -- не равны, включая случай когда оба NULL
a IS NOT DISTINCT FROM b  -- равны, включая случай когда оба NULL
```

По [документации](https://www.postgresql.org/docs/current/functions-comparison.html):

- `1 IS DISTINCT FROM NULL` → true;
- `NULL IS DISTINCT FROM NULL` → false;
- `NULL IS NOT DISTINCT FROM NULL` → true.

Удобно, когда нужно сравнивать значения, в том числе NULL.

---

## 3.3.13. Комбинированный пример

```sql
SELECT id, name, price, status
FROM products
WHERE (category_id IN (1, 2, 3) OR price BETWEEN 10 AND 50)
  AND status = 'active'
  AND name ILIKE '%phone%'
  AND (discount IS NULL OR discount > 0);
```

(Среда: PostgreSQL 14+, psql)

---

## 3.3.14. Типичные ошибки

- **`column = NULL`** — не сработает; используйте `column IS NULL`.
- **Приоритет AND/OR** — AND выполняется раньше OR; при сложных условиях используйте скобки.
- **LIKE и подстрока** — для поиска подстроки нужны `%` с обеих сторон: `'%substring%'`.
- **Регистр в LIKE** — LIKE учитывает регистр; для игнорирования используйте ILIKE или `LOWER(column) LIKE LOWER(pattern)`.
- **NOT IN с NULL** — при NULL в правой части результат может быть NULL; предпочтительнее альтернативы (NOT EXISTS и т.п.).
- **BETWEEN и границы** — обе границы включены; порядок «низ, верх» обязателен для обычного BETWEEN.

---

## Ключевое

- WHERE задаёт условие фильтрации; остаются только строки, для которых условие даёт TRUE.
- **Сравнение:** `=`, `<>`, `!=`, `<`, `>`, `<=`, `>=`.
- **Логика:** AND, OR, NOT; приоритет: NOT > AND > OR.
- **NULL:** только `IS NULL` и `IS NOT NULL`; `= NULL` не работает.
- **LIKE:** `%` — любая последовательность, `_` — один символ; ILIKE — без учёта регистра.
- **IN** — принадлежность списку; **BETWEEN** — диапазон (границы включены).
- **IS DISTINCT FROM** — сравнение с учётом NULL.

В [§3.4](chapter-03-04.md) рассмотрим DISTINCT — как получать уникальные значения в результате запроса.
