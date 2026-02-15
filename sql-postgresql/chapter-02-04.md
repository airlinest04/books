# §2.4 Изменение структуры

В [§2.3](chapter-02-03.md) мы рассмотрели ограничения, задаваемые при создании таблицы. Часто структуру нужно менять после создания: добавлять и удалять столбцы, менять типы, переименовывать объекты, добавлять или снимать ограничения. Для этого используется команда **ALTER TABLE**. В этом разделе мы разберём основные варианты: ADD COLUMN, DROP COLUMN, RENAME COLUMN, RENAME TABLE, изменение типа и ограничений столбцов. В [§3.1](chapter-03-01.md) мы перейдём к выборке данных — списку столбцов и псевдонимам в SELECT. См. [документацию ALTER TABLE](https://www.postgresql.org/docs/current/sql-altertable.html).

---

## 2.4.1. ADD COLUMN: добавление столбца

Чтобы добавить новый столбец к существующей таблице:

```sql
ALTER TABLE имя_таблицы ADD COLUMN имя_столбца тип_данных;
```

Можно указать ограничения и значение по умолчанию:

```sql
ALTER TABLE users ADD COLUMN created_at timestamptz DEFAULT now();
ALTER TABLE products ADD COLUMN is_active boolean NOT NULL DEFAULT true;
```

Если столбец с таким именем уже есть, команда завершится ошибкой. С PostgreSQL 9.6+ можно использовать `IF NOT EXISTS`:

```sql
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone text;
```

При добавлении столбца с `NOT NULL` без `DEFAULT` таблица должна быть пустой; иначе PostgreSQL не сможет заполнить существующие строки. При наличии `DEFAULT` значение подставляется во все существующие строки (в современных версиях — без полной перезаписи таблицы). См. [документацию](https://www.postgresql.org/docs/current/sql-altertable.html).

---

## 2.4.2. DROP COLUMN: удаление столбца

Удаление столбца:

```sql
ALTER TABLE имя_таблицы DROP COLUMN имя_столбца;
```

Пример:

```sql
ALTER TABLE users DROP COLUMN phone;
```

При удалении столбца автоматически удаляются индексы и ограничения, которые его используют. Если на столбец ссылаются внешние ключи, представления и т.п., потребуется `CASCADE`:

```sql
ALTER TABLE users DROP COLUMN department_id CASCADE;
```

`CASCADE` удаляет зависимые объекты (внешние ключи, представления). Использовать осторожно.

Без ошибки при отсутствии столбца (PostgreSQL 9.0+):

```sql
ALTER TABLE users DROP COLUMN IF EXISTS obsolete_column;
```

---

## 2.4.3. RENAME COLUMN: переименование столбца

Переименование столбца:

```sql
ALTER TABLE имя_таблицы RENAME COLUMN старое_имя TO новое_имя;
```

Ключевое слово `COLUMN` можно опустить:

```sql
ALTER TABLE users RENAME COLUMN full_name TO name;
```

Индексы, ограничения и зависимости сохраняются. Переименование не затрагивает данные. Представления и функции, ссылающиеся на старый столбец по имени, перестанут работать — их нужно обновить.

---

## 2.4.4. RENAME TO: переименование таблицы

Переименование таблицы:

```sql
ALTER TABLE старое_имя RENAME TO новое_имя;
```

Пример:

```sql
ALTER TABLE user_accounts RENAME TO users;
```

При переименовании сохраняются данные, индексы и ограничения. Представления, триггеры, правила и другие объекты, ссылающиеся на таблицу по имени, нужно обновить вручную.

---

## 2.4.5. Изменение типа столбца

Смена типа данных столбца:

```sql
ALTER TABLE имя_таблицы ALTER COLUMN имя_столбца SET DATA TYPE новый_тип;
```

Ключевые слова `SET DATA` можно опустить:

```sql
ALTER TABLE products ALTER COLUMN price TYPE numeric(12, 2);
```

Если типы несовместимы, PostgreSQL может не выполнить неявное приведение. В этом случае используется `USING`:

```sql
ALTER TABLE events ALTER COLUMN event_date TYPE date USING event_date::date;
```

В `USING` указывается выражение, по которому вычисляется новое значение из старого. Без `USING` PostgreSQL пробует стандартное приведение; при невозможности — ошибка.

При смене типа выполняется полное сканирование таблицы; на больших таблицах операция может быть долгой.

---

## 2.4.6. SET DEFAULT и DROP DEFAULT

Установка и удаление значения по умолчанию:

```sql
ALTER TABLE orders ALTER COLUMN status SET DEFAULT 'pending';
ALTER TABLE orders ALTER COLUMN status DROP DEFAULT;
```

`SET DEFAULT` действует только на последующие операции `INSERT`; существующие строки не меняются. `DROP DEFAULT` эквивалентен установке `NULL` в качестве значения по умолчанию.

---

## 2.4.7. SET NOT NULL и DROP NOT NULL

Включение и снятие ограничения NOT NULL:

```sql
ALTER TABLE users ALTER COLUMN email SET NOT NULL;
ALTER TABLE users ALTER COLUMN email DROP NOT NULL;
```

`SET NOT NULL` допустим только если во всех строках в этом столбце нет `NULL`. Иначе будет ошибка. PostgreSQL проверяет это полным сканированием таблицы (если нет подходящего CHECK, исключающего NULL).

---

## 2.4.8. ADD CONSTRAINT: добавление ограничений

Ограничения можно добавлять к уже созданной таблице:

```sql
ALTER TABLE products ADD CONSTRAINT products_price_positive CHECK (price > 0);
ALTER TABLE orders ADD CONSTRAINT orders_customer_fk
  FOREIGN KEY (customer_id) REFERENCES customers (id);
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);
```

Синтаксис такой же, как в `CREATE TABLE`. PostgreSQL проверяет все существующие строки на соответствие новому ограничению; при нарушении команда завершается ошибкой.

Именованные ограничения упрощают последующее изменение и удаление.

---

## 2.4.9. DROP CONSTRAINT: удаление ограничений

Удаление ограничения по имени:

```sql
ALTER TABLE products DROP CONSTRAINT products_price_positive;
ALTER TABLE orders DROP CONSTRAINT orders_customer_fk;
```

Если ограничение не существует, будет ошибка. Вариант с проверкой:

```sql
ALTER TABLE products DROP CONSTRAINT IF EXISTS products_price_positive;
```

Если на ограничение ссылаются другие объекты (например, внешние ключи), может понадобиться `CASCADE`:

```sql
ALTER TABLE products DROP CONSTRAINT products_pkey CASCADE;
```

---

## 2.4.10. SET SCHEMA: перенос таблицы в другую схему

Перенос таблицы в другую схему:

```sql
ALTER TABLE users SET SCHEMA other_schema;
```

После выполнения таблица будет доступна как `other_schema.users`. Индексы, ограничения и триггеры переносятся вместе с таблицей.

---

## 2.4.11. Несколько действий в одной команде

В одной команде `ALTER TABLE` можно указать несколько действий через запятую:

```sql
ALTER TABLE users
  ADD COLUMN phone text,
  ADD COLUMN updated_at timestamptz DEFAULT now(),
  ALTER COLUMN name SET NOT NULL;
```

Это удобно и может быть эффективнее, чем несколько отдельных команд, так как таблица блокируется один раз.

---

## 2.4.12. IF EXISTS и блокировки

Для `ALTER TABLE` можно использовать `IF EXISTS` (PostgreSQL 9.0+):

```sql
ALTER TABLE IF EXISTS old_table RENAME TO new_table;
```

При отсутствии таблицы ошибки не будет, только notice.

`ALTER TABLE` обычно блокирует таблицу на время выполнения. Настройка структуры больших активных таблиц лучше выполнять в период низкой нагрузки. Для добавления столбца с `DEFAULT` в новых версиях PostgreSQL часто не требуется перезапись всей таблицы; детали зависят от версии.

---

## 2.4.13. Типичные ошибки

- **SET NOT NULL при наличии NULL:** если в столбце есть `NULL`, `SET NOT NULL` завершится ошибкой. Сначала обновите или удалите такие строки.
- **Смена типа без USING:** при несовместимых типах добавьте `USING` с выражением преобразования.
- **DROP COLUMN с зависимостями:** при наличии внешних ключей, представлений и т.д. используйте `CASCADE` или сначала удалите зависимости.
- **Забыть обновить представления:** после переименования таблицы или столбца проверьте представления, функции и приложения.

---

## Ключевое

- **ADD COLUMN** — добавление столбца; с `IF NOT EXISTS` — без ошибки при существующем столбце.
- **DROP COLUMN** — удаление столбца; `CASCADE` — удаление зависимых объектов.
- **RENAME COLUMN** — переименование столбца; **RENAME TO** — переименование таблицы.
- **ALTER COLUMN ... SET DATA TYPE** — смена типа; при несовместимости — выражение `USING`.
- **ALTER COLUMN ... SET DEFAULT / DROP DEFAULT** — установка и сброс значения по умолчанию.
- **ALTER COLUMN ... SET NOT NULL / DROP NOT NULL** — добавление и снятие NOT NULL.
- **ADD CONSTRAINT** — добавление ограничения; **DROP CONSTRAINT** — удаление по имени.
- **SET SCHEMA** — перенос таблицы в другую схему.
- В одной команде `ALTER TABLE` можно указать несколько действий через запятую.

В [§3.1](chapter-03-01.md) мы перейдём к выборке данных: список столбцов в SELECT, использование `*` и псевдонимы (AS).
