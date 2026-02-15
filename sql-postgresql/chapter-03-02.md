# §3.2 Выражения и функции в SELECT

В [§3.1](chapter-03-01.md) мы рассмотрели список столбцов и псевдонимы. В SELECT можно указывать не только имена столбцов, но и **выражения** — формулы, вызовы функций и их комбинации. В этом разделе мы разберём конкатенацию строк, арифметику, строковые функции (UPPER, LOWER, LENGTH и др.), а также основные функции для работы с датами. Эти конструкции пригодятся и в WHERE, ORDER BY и других частях запроса. Фильтрация через WHERE рассматривается в [§3.3](chapter-03-03.md). См. [документацию по функциям](https://www.postgresql.org/docs/current/functions.html).

---

## 3.2.1. Выражения в SELECT

Выражение — это комбинация констант, столбцов, операторов и функций, дающая одно значение. Любое выражение может стоять в списке SELECT:

```sql
SELECT id, price * quantity AS total FROM order_items;
SELECT name, 2 + 3 AS five FROM users;
```

Результат вычисляется для каждой строки отдельно. При использовании выражений псевдоним особенно полезен — иначе столбец получит сгенерированное или пустое имя.

---

## 3.2.2. Конкатенация строк (||)

Оператор `||` объединяет строки. По [документации](https://www.postgresql.org/docs/current/functions-string.html):

```sql
SELECT 'Hello, ' || name || '!' AS greeting FROM users;
SELECT id || ': ' || name AS id_name FROM products;
```

Если один из операндов не строка (например, число), PostgreSQL приводит его к `text`:

```sql
SELECT 'Order #' || id || ', total: ' || total AS order_info FROM orders;
```

При `NULL` в любом операнде результат конкатенации — `NULL`. Чтобы заменить `NULL` на пустую строку, можно использовать `COALESCE` (см. ниже).

---

## 3.2.3. Функция concat

Функция `concat` принимает любое число аргументов, приводит их к тексту и объединяет. Аргументы `NULL` пропускаются:

```sql
SELECT concat('Hello, ', name, '!') AS greeting FROM users;
SELECT concat(id, ': ', name) FROM products;
SELECT concat('a', NULL, 'b') AS result;  -- результат: 'ab'
```

`concat_ws` объединяет с разделителем (первый аргумент — разделитель):

```sql
SELECT concat_ws(', ', last_name, first_name) AS full_name FROM customers;
SELECT concat_ws(' - ', id, name, status) AS line FROM orders;
```

`NULL` в аргументах (кроме разделителя) пропускается.

---

## 3.2.4. Арифметические операции

Для числовых типов доступны операторы `+`, `-`, `*`, `/`, `%` (остаток от деления). По [документации](https://www.postgresql.org/docs/current/functions-math.html):

```sql
SELECT price, quantity, price * quantity AS subtotal FROM order_items;
SELECT total, discount, total - total * discount / 100 AS final_price FROM orders;
SELECT amount, amount % 100 AS remainder FROM payments;
```

Целочисленное деление (`/`) для `integer` отбрасывает дробную часть:

```sql
SELECT 7 / 2;   -- 3
SELECT 7.0 / 2; -- 3.5
```

Для округления — `round`, для отсечения дробной части — `trunc`:

```sql
SELECT round(price * 1.2, 2) AS price_with_vat FROM products;
SELECT trunc(avg_rating, 0) AS avg_int FROM reviews;
```

Любая арифметика с `NULL` даёт `NULL`: `10 + NULL` → `NULL`.

---

## 3.2.5. Строковые функции: UPPER, LOWER, LENGTH

Часто используемые функции для строк (по [документации](https://www.postgresql.org/docs/current/functions-string.html)):

| Функция | Описание | Пример |
|---------|----------|--------|
| `upper(text)` | Верхний регистр | `upper('hello')` → `HELLO` |
| `lower(text)` | Нижний регистр | `lower('HELLO')` → `hello` |
| `initcap(text)` | Первая буква каждого слова заглавная | `initcap('hello world')` → `Hello World` |
| `length(text)` | Длина строки в символах | `length('hello')` → `5` |
| `char_length(text)` | То же, что `length` | `char_length('josé')` → `4` |
| `octet_length(text)` | Длина в байтах | `octet_length('josé')` → `5` (UTF-8) |

Примеры в запросе:

```sql
SELECT name, upper(name) AS name_upper, lower(email) AS email_lower FROM users;
SELECT title, length(title) AS title_length FROM articles;
SELECT initcap(status) AS status_cap FROM orders;
```

---

## 3.2.6. substring и left/right

`substring` извлекает подстроку. Синтаксис:

```sql
substring(строка FROM start [FOR count])
```

`start` — начало (1-based), `count` — число символов. Без `FOR count` — до конца строки:

```sql
SELECT substring('PostgreSQL' FROM 5 FOR 3);   -- 'gre'
SELECT substring('PostgreSQL' FROM 5);         -- 'greSQL'
```

Альтернатива — `left` и `right`:

```sql
SELECT left('abcdef', 3);   -- 'abc'
SELECT right('abcdef', 2);  -- 'ef'
```

---

## 3.2.7. trim, ltrim, rtrim

Удаление пробелов (или указанных символов) с краёв строки:

```sql
SELECT trim('  hello  ');           -- 'hello'
SELECT trim(both 'x' from 'xxhelloxx');  -- 'hello'
SELECT ltrim('  hello');            -- 'hello'
SELECT rtrim('hello  ');            -- 'hello'
```

---

## 3.2.8. replace и position

`replace` заменяет все вхождения подстроки:

```sql
SELECT replace('hello world', 'o', '0');  -- 'hell0 w0rld'
```

`position` возвращает позицию подстроки (1-based) или 0, если не найдено:

```sql
SELECT position('ell' in 'hello');  -- 2
```

---

## 3.2.9. Функции даты и времени

Основные функции (по [документации](https://www.postgresql.org/docs/current/functions-datetime.html)):

| Функция | Описание | Пример результата |
|---------|----------|-------------------|
| `now()` | Текущие дата и время (timestamptz) | `2024-03-15 14:30:00+03` |
| `current_date` | Текущая дата | `2024-03-15` |
| `current_timestamp` | Аналог `now()` | |
| `current_time` | Текущее время | `14:30:00+03` |

Примеры в запросе:

```sql
SELECT name, created_at, now() - created_at AS age FROM orders;
SELECT current_date, current_time;
```

---

## 3.2.10. extract и date_part

`extract` извлекает часть даты или времени:

```sql
extract(поле FROM источник)
```

Поля: `year`, `month`, `day`, `hour`, `minute`, `second`, `dow` (день недели 0–6), `quarter`, `week` и др.

```sql
SELECT extract(year FROM created_at) AS year FROM orders;
SELECT extract(month FROM created_at) AS month, extract(day FROM created_at) AS day FROM events;
SELECT date_part('hour', created_at) AS hour FROM logs;
```

`date_part('field', source)` — альтернативный синтаксис; поле задаётся строкой.

---

## 3.2.11. date_trunc

`date_trunc` обрезает дату/время до указанной точности:

```sql
SELECT date_trunc('day', created_at) AS day_start FROM orders;
SELECT date_trunc('month', created_at) AS month_start FROM events;
SELECT date_trunc('hour', now()) AS current_hour;
```

Первый аргумент: `microsecond`, `millisecond`, `second`, `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year`.

---

## 3.2.12. Арифметика с датами и интервалами

С датами и интервалами можно выполнять арифметику:

```sql
SELECT created_at + interval '7 days' AS week_later FROM orders;
SELECT current_date - birth_date AS age_days FROM users;
SELECT created_at - interval '1 month' AS month_ago FROM events;
```

`date - date` даёт число дней (integer). `timestamp - timestamp` даёт interval.

---

## 3.2.13. COALESCE: подстановка значения вместо NULL

`COALESCE` возвращает первый аргумент, который не `NULL`:

```sql
SELECT COALESCE(phone, email, 'no contact') AS contact FROM users;
SELECT COALESCE(discount, 0) * price AS discount_amount FROM products;
```

Удобно для конкатенации с возможными NULL:

```sql
SELECT concat(name, COALESCE(', ' || phone, '')) AS contact FROM users;
```

---

## 3.2.14. Комбинированный пример

```sql
SELECT
  id,
  upper(name) AS name_upper,
  length(name) AS name_length,
  round(price * 1.2, 2) AS price_with_vat,
  extract(year FROM created_at) AS year_created,
  concat_ws(' ', first_name, last_name) AS full_name
FROM products;
```

(Среда: PostgreSQL 14+, psql)

---

## 3.2.15. NULL в выражениях

В большинстве выражений при наличии `NULL` результат — `NULL`:

- `NULL + 10` → `NULL`
- `'a' || NULL` → `NULL`
- `upper(NULL)` → `NULL`

Исключение — функции вроде `concat`, которые специально обрабатывают `NULL`. Для подстановки значения вместо `NULL` используйте `COALESCE` или `NULLIF` (см. документацию).

---

## 3.2.16. Типичные ошибки

- **Деление на ноль:** `1/0` вызывает ошибку; проверяйте знаменатель или используйте `NULLIF(denominator, 0)`.
- **Конкатенация с NULL:** `'a' || NULL` даёт `NULL`; при необходимости используйте `COALESCE(column, '')`.
- **Разница length и octet_length:** в UTF-8 один символ может занимать несколько байт; `length` считает символы, `octet_length` — байты.
- **Позиции в substring:** отсчёт с 1, а не с 0.

---

## Ключевое

- В SELECT можно использовать выражения: константы, столбцы, операторы и функции.
- **Конкатенация:** `||` или `concat`; `concat` пропускает `NULL`.
- **Арифметика:** `+`, `-`, `*`, `/`, `%`; при `NULL` результат `NULL`.
- **Строки:** `upper`, `lower`, `initcap`, `length`, `substring`, `trim`, `replace`, `position`.
- **Даты:** `now()`, `current_date`, `extract`, `date_trunc`; арифметика с `interval`.
- **COALESCE** — подстановка значения вместо `NULL`.
- Выражения с `NULL` обычно дают `NULL`; исключения — функции вроде `concat`.

В [§3.3](chapter-03-03.md) мы рассмотрим фильтрацию: WHERE с условиями сравнения, логическими операторами, IS NULL и шаблонами LIKE.
