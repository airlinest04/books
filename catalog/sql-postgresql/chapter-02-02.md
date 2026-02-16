# §2.2 Типы данных в PostgreSQL

В [§2.1](chapter-02-01.md) мы создали таблицы, задав для каждого столбца тип данных. Тип определяет, какие значения можно хранить и какие операции с ними возможны. В этом разделе мы рассмотрим **основные типы данных PostgreSQL**: целые числа, строки, числа с плавающей точкой и точной арифметикой, даты и время, логический тип, JSON и другие. Выбор подходящего типа влияет на корректность, производительность и переносимость приложений. Ограничения столбцов (NOT NULL, DEFAULT и др.) разбираются в [§2.3](chapter-02-03.md). См. [документацию по типам данных](https://www.postgresql.org/docs/current/datatype.html).

---

## 2.2.1. Целочисленные типы

Целые числа хранятся без дробной части. PostgreSQL поддерживает три типа (по [документации](https://www.postgresql.org/docs/current/datatype-numeric.html)):

| Тип | Синонимы | Размер | Диапазон |
|-----|----------|--------|----------|
| `smallint` | `int2` | 2 байта | от -32 768 до +32 767 |
| `integer` | `int`, `int4` | 4 байта | от -2 147 483 648 до +2 147 483 647 |
| `bigint` | `int8` | 8 байт | от -9 223 372 036 854 775 808 до +9 223 372 036 854 775 807 |

Пример:

```sql
CREATE TABLE products (
  id integer,
  stock smallint,
  views_count bigint
);
```

**Рекомендации:** `integer` — обычный выбор для счётчиков и идентификаторов. `smallint` — если важен минимальный объём (ограниченный диапазон). `bigint` — когда возможны значения за пределами `integer` (например, счётчики просмотров, большие суммы).

---

## 2.2.2. Serial: автоинкремент

Типы `smallserial`, `serial` и `bigserial` — удобное обозначение для столбцов с автоувеличением. `serial` эквивалентен `integer` с последовательностью и значением по умолчанию `nextval()`. По [документации](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-SERIAL):

```sql
CREATE TABLE users (
  id serial PRIMARY KEY,
  name text
);
```

Эквивалентно:

```sql
CREATE SEQUENCE users_id_seq;
CREATE TABLE users (
  id integer NOT NULL DEFAULT nextval('users_id_seq'),
  name text
);
ALTER SEQUENCE users_id_seq OWNED BY users.id;
```

При вставке можно не указывать `id` — будет подставлено следующее значение. Для очень больших объёмов записей используйте `bigserial`. Подробнее об ограничениях и `IDENTITY` — в [§2.3](chapter-02-03.md).

---

## 2.2.3. Символьные типы (VARCHAR, CHAR, TEXT)

Для хранения текста используются типы (по [документации](https://www.postgresql.org/docs/current/datatype-character.html)):

| Тип | Описание |
|-----|----------|
| `varchar(n)` | Строка переменной длины до n символов |
| `character(n)` или `char(n)` | Строка фиксированной длины n; дополняется пробелами справа |
| `text` | Строка переменной длины без ограничения |

Примеры:

```sql
CREATE TABLE articles (
  title varchar(200),
  code char(10),
  content text
);
```

**Рекомендации:** в PostgreSQL **`text`** обычно предпочтителен для произвольного текста — ограничения по длине почти нет (до ~1 ГБ), проверка длины немного замедляет запись. `varchar(n)` — когда нужно жёстко ограничить длину. `char(n)` — только при необходимости фиксированной ширины (коды, шифры). По производительности между `text` и `varchar` разница минимальна.

`varchar` без `(n)` эквивалентен `text`.

---

## 2.2.4. Точные числа: NUMERIC и DECIMAL

Тип `numeric` (или `decimal`) хранит числа с заданной точностью и масштабом. Подходит для денежных сумм и расчётов, где важна точность. По [документации](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-NUMERIC):

```sql
numeric( precision, scale )
```

- **precision** — общее число значащих цифр.
- **scale** — число цифр после запятой.

Примеры:

```sql
numeric(10, 2)   -- до 8 цифр до запятой, 2 после (например, 12345678.99)
numeric(5, 2)    -- от -999.99 до 999.99
```

```sql
CREATE TABLE orders (
  id integer,
  total numeric(10, 2),
  discount numeric(5, 4)
);
```

`decimal` и `numeric` в PostgreSQL эквивалентны. Для денег рекомендуется `numeric(p, s)` вместо `real` или `double precision`.

---

## 2.2.5. Числа с плавающей точкой: REAL и DOUBLE PRECISION

Типы `real` (4 байта) и `double precision` (8 байт) — приближённые числа с плавающей точкой. Они быстрее `numeric`, но дают погрешности округления. По [документации](https://www.postgresql.org/docs/current/datatype-numeric.html#DATATYPE-FLOAT):

| Тип | Синонимы | Размер | Точность |
|-----|----------|--------|----------|
| `real` | `float4` | 4 байта | ~6 десятичных знаков |
| `double precision` | `float8`, `float` | 8 байт | ~15 десятичных знаков |

Использовать для денег **не рекомендуется** — возможны ошибки округления. Подходят для измерений, научных расчётов, координат.

```sql
CREATE TABLE sensors (
  id integer,
  temperature real,
  latitude double precision,
  longitude double precision
);
```

---

## 2.2.6. DATE: дата без времени

Тип `date` хранит календарную дату (год, месяц, день) без времени. Размер — 4 байта. Диапазон — примерно 4713 г. до н.э. до 5874897 г. н.э. По [документации](https://www.postgresql.org/docs/current/datatype-datetime.html):

```sql
CREATE TABLE events (
  id integer,
  event_date date,
  description text
);

INSERT INTO events VALUES (1, '2024-03-15', 'Конференция');
```

Формат ISO 8601 (`YYYY-MM-DD`) однозначен при любых настройках. Поддерживаются и другие форматы: `15.03.2024`, `March 15, 2024` и т.д.

---

## 2.2.7. TIMESTAMP и TIMESTAMPTZ: дата и время

Типы для даты и времени:

| Тип | Описание | Часовой пояс |
|-----|----------|--------------|
| `timestamp` | Дата и время без учёта часового пояса | Не хранится |
| `timestamptz` | Дата и время с учётом часового пояса | Сохраняется в UTC |

`timestamp without time zone` и `timestamp with time zone` — полные имена; `timestamptz` — краткая форма. По умолчанию `timestamp` трактуется как `timestamp without time zone`.

```sql
CREATE TABLE logs (
  id integer,
  created_at timestamp,
  created_at_tz timestamptz
);
```

**Рекомендация:** для времени событий в реальном мире обычно используют **`timestamptz`** — значение хранится в UTC и показывается в часовом поясе сессии. `timestamp` — когда часовой пояс не важен (например, расписание «по местному времени» без привязки к зоне).

---

## 2.2.8. TIME и INTERVAL

- **`time`** — время суток без даты (часы, минуты, секунды). Есть вариант `time with time zone`, но он редко нужен.
- **`interval`** — длительность (например, «2 часа 30 минут»). Используется в арифметике дат и для хранения интервалов.

```sql
CREATE TABLE schedules (
  id integer,
  start_time time,
  duration interval
);

INSERT INTO schedules VALUES (1, '09:00', '2 hours 30 minutes');
INSERT INTO schedules VALUES (2, '14:00', interval '1 day');
```

---

## 2.2.9. BOOLEAN: логический тип

Тип `boolean` (синоним `bool`) хранит значения true/false. Размер — 1 байт. По [документации](https://www.postgresql.org/docs/current/datatype-boolean.html):

Допустимые значения «истина»: `true`, `yes`, `on`, `1` (регистр не важен).  
Допустимые значения «ложь»: `false`, `no`, `off`, `0`.  
Третий вариант — `NULL` (неизвестно).

При выводе PostgreSQL показывает `t` или `f`.

```sql
CREATE TABLE users (
  id integer,
  name text,
  is_active boolean DEFAULT true
);

INSERT INTO users VALUES (1, 'Alice', true);
INSERT INTO users VALUES (2, 'Bob', 'yes');
SELECT * FROM users WHERE is_active;
```

---

## 2.2.10. JSON и JSONB

PostgreSQL поддерживает хранение JSON (по [документации](https://www.postgresql.org/docs/current/datatype-json.html)):

- **`json`** — хранение текста «как есть»; проверка при каждой обработке.
- **`jsonb`** — хранение в бинарном виде; быстрее при поиске и индексировании, не сохраняет пробелы и порядок ключей.

**Рекомендация:** в большинстве случаев использовать **`jsonb`**.

```sql
CREATE TABLE configs (
  id integer,
  settings jsonb
);

INSERT INTO configs VALUES (1, '{"theme": "dark", "notifications": true}');
INSERT INTO configs VALUES (2, '[1, 2, 3]');
```

Для выборки полей используются операторы `->`, `->>`, `@>` и др.; подробнее — в документации по [функциям JSON](https://www.postgresql.org/docs/current/functions-json.html).

---

## 2.2.11. Другие полезные типы

| Тип | Назначение |
|-----|------------|
| `uuid` | Уникальный идентификатор (RFC 4122); 16 байт |
| `bytea` | Двоичные данные (например, файлы) |
| `inet` | IPv4 или IPv6 адрес |
| `cidr` | IPv4/IPv6 сеть |
| `money` | Сумма в валюте (зависит от локали; для приложений часто лучше `numeric`) |

Пример с `uuid`:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE TABLE sessions (
  id uuid DEFAULT uuid_generate_v4(),
  user_id integer,
  created_at timestamptz DEFAULT now()
);
```

---

## 2.2.12. Сводная таблица типов

| Категория | Типы | Когда использовать |
|-----------|------|--------------------|
| Целые | `smallint`, `integer`, `bigint` | Счётчики, идентификаторы, количество |
| Автоинкремент | `serial`, `bigserial` | Первичные ключи с автоувеличением |
| Строки | `varchar(n)`, `text`, `char(n)` | Текст; `text` — основной выбор |
| Точные числа | `numeric(p,s)` | Деньги, точные расчёты |
| Приближённые | `real`, `double precision` | Измерения, координаты (не деньги) |
| Дата | `date` | Только дата |
| Дата и время | `timestamp`, `timestamptz` | События; чаще `timestamptz` |
| Время/интервал | `time`, `interval` | Время суток, длительность |
| Логика | `boolean` | Флаги, да/нет |
| JSON | `json`, `jsonb` | Полуструктурированные данные; чаще `jsonb` |

---

## 2.2.13. Приведение типов (CAST)

Преобразование значений между типами:

```sql
SELECT '123'::integer;
SELECT 42::text;
SELECT CAST('2024-01-15' AS date);
```

При несовместимых типах возникает ошибка.

---

## 2.2.14. Типичные ошибки

- **Деньги в `real` или `double precision`:** возможны ошибки округления; использовать `numeric`.
- **`varchar(255)` без необходимости:** в PostgreSQL `text` часто удобнее; длина 255 — legacy из других СУБД.
- **`timestamp` вместо `timestamptz`:** при работе с разными часовыми поясами предпочтительнее `timestamptz`.
- **`char(n)` для обычного текста:** дополнение пробелами и фиксированная длина редко нужны; чаще — `text` или `varchar`.
- **Превышение диапазона:** вставка значения вне диапазона типа приводит к ошибке.

---

## Ключевое

- **Целые:** `smallint`, `integer`, `bigint`; обычно используют `integer`.
- **Строки:** `text` — основной выбор; `varchar(n)` — при ограничении длины.
- **Точные числа:** `numeric(p,s)` для денег и точных расчётов.
- **Плавающая точка:** `real`, `double precision` — не для денег.
- **Дата и время:** `date`, `timestamp`, `timestamptz`; для событий — `timestamptz`.
- **Логика:** `boolean` (true/false).
- **JSON:** `jsonb` для полуструктурированных данных.
- **Serial:** `serial`, `bigserial` для автоинкремента.

В [§2.3](chapter-02-03.md) мы рассмотрим ограничения столбцов: NOT NULL, DEFAULT, UNIQUE, PRIMARY KEY, CHECK и внешние ключи.
