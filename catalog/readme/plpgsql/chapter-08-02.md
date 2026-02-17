# §8.2 Создание триггерной функции

В [§8.1](chapter-08-01.md) мы разобрали события триггера (BEFORE/AFTER, FOR EACH ROW/STATEMENT). Триггер вызывает **функцию**, которую нужно создать до привязки триггера. В этом разделе — объявление триггерной функции на PL/pgSQL: **RETURNS trigger**, отсутствие аргументов, специальные переменные **NEW** и **OLD** (и кратко TG_OP, TG_WHEN и др.), а также **что возвращать** — NEW, OLD или NULL в зависимости от типа триггера. CREATE TRIGGER и условие WHEN — в [§8.3](chapter-08-03.md). См. [Глоссарий](glossary.md).

---

## 8.2.1. Сигнатура: без аргументов, RETURNS trigger

Триггерная функция для **изменения данных** (INSERT, UPDATE, DELETE, TRUNCATE) объявляется как функция **без аргументов** с возвращаемым типом **trigger** (по документации PostgreSQL, раздел 41.10). Даже если в CREATE TRIGGER заданы аргументы, в объявлении функции параметров не указывают — аргументы передаются через массив **TG_ARGV**. См. [Глоссарий](glossary.md).

Пример объявления:

```sql
CREATE FUNCTION my_trigger_func() RETURNS trigger
AS $$
BEGIN
  -- тело
  RETURN NEW;  -- или OLD, или NULL
END;
$$ LANGUAGE plpgsql;
```

Язык — **LANGUAGE plpgsql** (или другой процедурный язык с поддержкой триггеров). Функция создаётся до CREATE TRIGGER; одну и ту же функцию можно привязать к разным триггерам.

---

## 8.2.2. NEW и OLD — данные строки

В построчном триггере (FOR EACH ROW) в верхнем блоке функции автоматически доступны переменные **NEW** и **OLD** типа **record** (по документации PostgreSQL). См. [Глоссарий](glossary.md).

- **NEW** — строка **после** изменения: для **INSERT** и **UPDATE** содержит новую версию строки. Для **DELETE** не определена (NULL). Обращение к полям: `NEW.имя_столбца`. В триггере **BEFORE** можно **изменять** NEW (присваивать полям); возвращённая строка станет той, что будет записана (для INSERT/UPDATE).
- **OLD** — строка **до** изменения: для **UPDATE** и **DELETE** содержит старую версию строки. Для **INSERT** не определена (NULL). Только для чтения; изменять OLD нельзя.

В триггере **по оператору** (FOR EACH STATEMENT) NEW и OLD в обычном варианте не заданы (NULL); доступ к наборам изменённых строк — через transition tables в CREATE TRIGGER (отдельная тема). В триггерах на TRUNCATE нет построчных данных.

---

## 8.2.3. Что возвращать из триггерной функции

Возвращаемое значение должно быть **NULL** или **строка (record)** с той же структурой, что и таблица триггера (по документации PostgreSQL). См. [Глоссарий](glossary.md).

**Построчный BEFORE:**
- **RETURN NULL** — отменить операцию для этой строки (вставка/обновление/удаление не выполняется, последующие триггеры для этой строки не вызываются).
- **RETURN NEW** — продолжить операцию; для INSERT/UPDATE подставляется эта строка (можно вернуть изменённый NEW). Для DELETE возвращаемое значение не меняет строку, но должно быть не NULL, чтобы операция прошла; обычно **RETURN OLD**.

**Построчный AFTER** и **триггер по оператору** (BEFORE или AFTER): возврат **игнорируется**; можно писать **RETURN NULL**. Прервать операцию можно только вызовом исключения (RAISE EXCEPTION).

**INSTEAD OF** (только представления): RETURN NULL — «ничего не сделано» для этой строки; иначе RETURN NEW (для INSERT/UPDATE) или RETURN OLD (для DELETE), при необходимости изменив NEW.

Кратко: для **BEFORE INSERT/UPDATE** — изменить при необходимости NEW и **RETURN NEW**; для **BEFORE DELETE** — **RETURN OLD**; для **AFTER** — **RETURN NULL**.

---

## 8.2.4. Дополнительные переменные: TG_OP, TG_WHEN, TG_TABLE_NAME и др.

В теле триггерной функции доступны и другие переменные (по документации PostgreSQL): См. [Глоссарий](glossary.md).

- **TG_OP** (text) — операция: 'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE'.
- **TG_WHEN** (text) — момент: 'BEFORE', 'AFTER', 'INSTEAD OF'.
- **TG_LEVEL** (text) — уровень: 'ROW' или 'STATEMENT'.
- **TG_TABLE_NAME** (name) — имя таблицы, на которой сработал триггер.
- **TG_TABLE_SCHEMA** (name) — схема таблицы.
- **TG_NAME** (name) — имя триггера.
- **TG_NARGS** (integer), **TG_ARGV** (text[]) — число и массив аргументов из CREATE TRIGGER (индексация с 0).

По **TG_OP** в одной функции можно разветвить логику для INSERT, UPDATE и DELETE. Пример:

```sql
IF TG_OP = 'INSERT' THEN
  NEW.created_at := current_timestamp;
ELSIF TG_OP = 'UPDATE' THEN
  NEW.updated_at := current_timestamp;
END IF;
RETURN NEW;
```

---

## 8.2.5. Пример: проверка и заполнение полей (BEFORE)

Пример (по документации PostgreSQL, адаптировано): триггер BEFORE INSERT OR UPDATE проверяет обязательные поля и заполняет last_date и last_user.

```sql
CREATE FUNCTION emp_stamp() RETURNS trigger AS $$
BEGIN
  IF NEW.empname IS NULL THEN
    RAISE EXCEPTION 'empname cannot be null';
  END IF;
  IF NEW.salary IS NOT NULL AND NEW.salary < 0 THEN
    RAISE EXCEPTION '% cannot have a negative salary', NEW.empname;
  END IF;
  NEW.last_date := current_timestamp;
  NEW.last_user := current_user;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Таблица emp должна содержать столбцы empname, salary, last_date, last_user. Привязка триггера — в [§8.3](chapter-08-03.md). См. [Глоссарий](glossary.md).

---

## Ключевое

- Триггерная функция: **без аргументов**, **RETURNS trigger**; аргументы из CREATE TRIGGER доступны через **TG_ARGV**.
- **NEW** — новая строка (INSERT/UPDATE); **OLD** — старая (UPDATE/DELETE); в построчном BEFORE можно изменять NEW и вернуть его.
- **BEFORE по строке:** RETURN NULL — отменить операцию; RETURN NEW (или изменённый NEW) — выполнить INSERT/UPDATE; RETURN OLD — для DELETE. **AFTER** и по оператору — возврат не используется, обычно RETURN NULL.
- **TG_OP**, **TG_WHEN**, **TG_TABLE_NAME** и др. — тип операции, момент, таблица; по TG_OP удобно разветвлять логику.

В [§8.3](chapter-08-03.md) разберём **CREATE TRIGGER**: привязка функции к таблице, указание события и уровня, условие **WHEN** для построчных триггеров.
