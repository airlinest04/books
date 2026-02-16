# §2.3 Ограничения столбцов и таблиц

В [§2.2](chapter-02-02.md) мы рассмотрели типы данных — они ограничивают вид хранимых значений. Типов недостаточно, чтобы задать правила вроде «цена положительная» или «email уникален». Для этого в SQL используются **ограничения (constraints)**: NOT NULL, DEFAULT, UNIQUE, PRIMARY KEY, CHECK и FOREIGN KEY. В этом разделе мы разберём каждое ограничение, синтаксис на уровне столбца и таблицы, а также **внешние ключи** и ссылочную целостность. Изменение структуры таблицы (добавление и удаление ограничений) рассматривается в [§2.4](chapter-02-04.md). См. [документацию по ограничениям](https://www.postgresql.org/docs/current/ddl-constraints.html).

---

## 2.3.1. Ограничения: столбцовые и табличные

Ограничения делятся на два вида:

- **Столбцовые (column constraints)** — задаются рядом с определением столбца.
- **Табличные (table constraints)** — задаются отдельно в теле `CREATE TABLE` и могут затрагивать несколько столбцов.

Оба вида проверяются при вставке и обновлении. Нарушение ограничения приводит к ошибке и откату операции. Любое столбцовое ограничение можно переписать как табличное; обратное не всегда возможно (например, ограничение на несколько столбцов). См. [Глоссарий](glossary.md).

---

## 2.3.2. NOT NULL: запрет пустых значений

Ограничение **NOT NULL** запрещает значение `NULL` в столбце. Без него столбец по умолчанию допускает `NULL`. По [документации](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-NOT-NULL):

```sql
CREATE TABLE users (
  id integer NOT NULL,
  name text NOT NULL,
  email text
);
```

Столбцы `id` и `name` обязательны; `email` может быть `NULL`. При вставке без указания `email` будет подставлен `NULL`.

Имя ограничения (опционально):

```sql
name text CONSTRAINT users_name_not_null NOT NULL
```

В PostgreSQL NOT NULL эффективнее, чем `CHECK (column IS NOT NULL)`.

---

## 2.3.3. DEFAULT: значение по умолчанию

**DEFAULT** задаёт значение, которое подставляется, если при вставке столбец не указан:

```sql
CREATE TABLE orders (
  id integer,
  status text DEFAULT 'pending',
  created_at timestamptz DEFAULT now(),
  total numeric(10, 2) DEFAULT 0
);
```

При `INSERT INTO orders (id) VALUES (1)` для `status`, `created_at` и `total` будут использованы значения по умолчанию. Выражение `now()` вычисляется в момент вставки.

DEFAULT можно комбинировать с другими ограничениями:

```sql
CREATE TABLE logs (
  id serial PRIMARY KEY,
  message text NOT NULL,
  level text DEFAULT 'info' CHECK (level IN ('debug', 'info', 'warning', 'error'))
);
```

Порядок ограничений и DEFAULT не важен.

---

## 2.3.4. UNIQUE: уникальность

**UNIQUE** запрещает дублирование значений в одном или нескольких столбцах. `NULL` по умолчанию не считается равным другому `NULL`, поэтому в столбце с UNIQUE может быть несколько строк с `NULL`. По [документации](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-UNIQUE-CONSTRAINTS):

Столбцовое ограничение:

```sql
CREATE TABLE users (
  id integer,
  email varchar(255) UNIQUE,
  name text
);
```

Табличное (одна или несколько колонок):

```sql
CREATE TABLE order_items (
  order_id integer,
  product_id integer,
  quantity integer,
  UNIQUE (order_id, product_id)
);
```

Комбинация `(order_id, product_id)` должна быть уникальной; каждый столбец по отдельности может повторяться.

С UNIQUE автоматически создаётся уникальный B-tree-индекс, ускоряющий поиск и проверку уникальности. Именованное ограничение:

```sql
email varchar(255) CONSTRAINT users_email_unique UNIQUE
```

Для учёта `NULL` в уникальности (PostgreSQL 15+):

```sql
email varchar(255) UNIQUE NULLS NOT DISTINCT
```

---

## 2.3.5. PRIMARY KEY: первичный ключ

**PRIMARY KEY** объединяет уникальность и NOT NULL. Обычно используется для идентификатора строки. По [документации](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-PRIMARY-KEYS):

```sql
CREATE TABLE products (
  product_no integer PRIMARY KEY,
  name text NOT NULL,
  price numeric CHECK (price > 0)
);
```

Эквивалентно `UNIQUE NOT NULL`. Первичный ключ задаёт целевой столбец для внешних ключей и создаёт уникальный индекс.

Составной первичный ключ (несколько столбцов):

```sql
CREATE TABLE order_items (
  order_id integer,
  product_id integer,
  quantity integer,
  PRIMARY KEY (order_id, product_id)
);
```

В таблице может быть только один PRIMARY KEY. Первичный ключ желательно задавать для каждой таблицы.

---

## 2.3.6. CHECK: проверка по условию

**CHECK** требует, чтобы выражение было истинным (или `NULL` — тогда проверка считается пройденной). По [документации](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-CHECK-CONSTRAINTS):

Столбцовое ограничение:

```sql
CREATE TABLE products (
  product_no integer PRIMARY KEY,
  name text,
  price numeric CHECK (price > 0),
  discount numeric CHECK (discount >= 0 AND discount <= 100)
);
```

Табличное (связь нескольких столбцов):

```sql
CREATE TABLE products (
  product_no integer PRIMARY KEY,
  price numeric,
  discounted_price numeric,
  CHECK (price > 0),
  CHECK (discounted_price > 0),
  CHECK (price > discounted_price)
);
```

CHECK не может ссылаться на другие строки или таблицы; для таких правил используются триггеры или внешние ключи.

Именованное ограничение:

```sql
CONSTRAINT valid_discount CHECK (price > discounted_price)
```

---

## 2.3.7. FOREIGN KEY: внешний ключ и ссылочная целостность

**FOREIGN KEY** (внешний ключ) связывает столбец (или группу столбцов) со столбцами другой таблицы. Это обеспечивает **ссылочную целостность**: нельзя сослаться на несуществующую строку. См. [Глоссарий](glossary.md).

Таблица, на которую ссылаются, — **ссылаемая**; таблица со ссылкой — **ссылающаяся**.

Синтаксис:

```sql
CREATE TABLE products (
  product_no integer PRIMARY KEY,
  name text,
  price numeric
);

CREATE TABLE orders (
  order_id integer PRIMARY KEY,
  product_no integer REFERENCES products (product_no),
  quantity integer
);
```

Строка в `orders` может быть вставлена только если `product_no` есть в `products`. При ссылке на первичный ключ можно опустить имя столбца:

```sql
product_no integer REFERENCES products
```

Составной внешний ключ:

```sql
CREATE TABLE t1 (
  a integer,
  b integer,
  c integer,
  FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
);
```

---

## 2.3.8. ON DELETE и ON UPDATE: поведение при изменениях

При удалении или изменении строки в ссылаемой таблице нужно определить, что делать со ссылающимися строками. За это отвечают `ON DELETE` и `ON UPDATE`:

| Действие | Поведение |
|----------|-----------|
| `NO ACTION` (по умолчанию) | Проверка после операции; при нарушении целостности — ошибка |
| `RESTRICT` | Запрет удаления/обновления, если есть ссылающиеся строки |
| `CASCADE` | Каскадное удаление/обновление ссылающихся строк |
| `SET NULL` | Установить ссылающийся столбец в `NULL` |
| `SET DEFAULT` | Установить значение по умолчанию |

Пример:

```sql
CREATE TABLE order_items (
  product_no integer REFERENCES products ON DELETE RESTRICT,
  order_id integer REFERENCES orders ON DELETE CASCADE,
  quantity integer,
  PRIMARY KEY (product_no, order_id)
);
```

- Удаление продукта запрещено, если на него есть заказы (`RESTRICT`).
- При удалении заказа связанные строки в `order_items` удаляются (`CASCADE`).

`SET NULL` и `SET DEFAULT` применяются, когда связь необязательна (столбец допускает `NULL`). По [документации](https://www.postgresql.org/docs/current/ddl-constraints.html#DDL-CONSTRAINTS-FK).

---

## 2.3.9. Самоссылающийся внешний ключ

Внешний ключ может ссылаться на ту же таблицу — для иерархий (дерево категорий, сотрудники и руководители):

```sql
CREATE TABLE categories (
  id integer PRIMARY KEY,
  name text NOT NULL,
  parent_id integer REFERENCES categories (id)
);
```

Корневые элементы имеют `parent_id = NULL`; остальные ссылаются на существующие строки.

---

## 2.3.10. Имена ограничений

Любое ограничение можно именовать через `CONSTRAINT имя`:

```sql
CREATE TABLE users (
  id integer CONSTRAINT users_pkey PRIMARY KEY,
  email varchar(255) CONSTRAINT users_email_unique UNIQUE NOT NULL,
  age integer CONSTRAINT users_age_check CHECK (age >= 0 AND age <= 150)
);
```

Именованные ограничения легче находить в сообщениях об ошибках и изменять через `ALTER TABLE`.

---

## 2.3.11. Комбинированный пример

Полный пример таблиц с ограничениями:

```sql
CREATE TABLE customers (
  id serial PRIMARY KEY,
  name text NOT NULL,
  email varchar(255) UNIQUE NOT NULL,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE orders (
  id serial PRIMARY KEY,
  customer_id integer NOT NULL REFERENCES customers (id) ON DELETE CASCADE,
  status text NOT NULL DEFAULT 'new' CHECK (status IN ('new', 'paid', 'shipped', 'cancelled')),
  total numeric(10, 2) NOT NULL DEFAULT 0 CHECK (total >= 0),
  created_at timestamptz DEFAULT now()
);
```

(Среда: PostgreSQL 14+, psql)

---

## 2.3.12. Типичные ошибки

- **CHECK и NULL:** выражение CHECK считается выполненным при `NULL`; для обязательности используйте NOT NULL.
- **UNIQUE и несколько NULL:** по умолчанию несколько `NULL` допустимы; при необходимости — `UNIQUE NULLS NOT DISTINCT`.
- **Внешний ключ без индекса на ссылающейся таблице:** при удалении/обновлении в ссылаемой таблице требуется поиск по ссылающейся; индекс на столбцах FK улучшает производительность (PostgreSQL не создаёт его автоматически).
- **CASCADE без понимания:** каскадное удаление может затронуть много строк; применяйте осознанно.
- **Циклические внешние ключи:** две таблицы, ссылающиеся друг на друга, создают зависимости; иногда нужно отложить проверку (DEFERRABLE) или разнести создание таблиц.

---

## Ключевое

- **NOT NULL** — запрет `NULL`; **DEFAULT** — значение по умолчанию при вставке.
- **UNIQUE** — уникальность значений; создаётся индекс; по умолчанию несколько `NULL` допустимы.
- **PRIMARY KEY** — уникальность и NOT NULL; в таблице один; целевой столбец для внешних ключей.
- **CHECK** — проверка по булевому выражению; не может ссылаться на другие строки.
- **FOREIGN KEY** — ссылка на столбец другой (или той же) таблицы; поддерживает ссылочную целостность.
- **ON DELETE / ON UPDATE:** `RESTRICT`, `CASCADE`, `SET NULL`, `SET DEFAULT` — поведение при удалении или изменении ссылаемой строки.
- Ограничения можно именовать через `CONSTRAINT имя` для удобства отладки и изменения.

В [§2.4](chapter-02-04.md) мы рассмотрим изменение структуры таблиц: ALTER TABLE для добавления, удаления и переименования столбцов и ограничений.
