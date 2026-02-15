# §8.4 Типичные применения

В [§8.3](chapter-08-03.md) мы разобрали CREATE TRIGGER и условие WHEN. В этом разделе — **типичные применения** триггеров на PL/pgSQL: **аудит** (запись изменений в отдельную таблицу), **заполнение служебных полей** (updated_at, updated_by) и **проверки с модификацией или отменой** операции через NEW. Ограничения, производительность и когда предпочесть ограничения БД или приложение — в [§8.5](chapter-08-05.md). См. [Глоссарий](glossary.md).

---

## 8.4.1. Аудит: логирование изменений

**Аудит** — фиксация факта и содержимого изменений (кто, когда, какая операция, старые/новые значения). Обычно реализуют **AFTER** триггером **FOR EACH ROW**: после INSERT, UPDATE или DELETE в основную таблицу вставляется запись в таблицу аудита (по документации PostgreSQL, пример 41.4). См. [Глоссарий](glossary.md).

В функции по **TG_OP** определяют тип операции и записывают в таблицу аудита метку времени (now()), пользователя (current_user), тип операции ('I'/'U'/'D') и данные строки (NEW.* или OLD.*). Возвращают NULL (для AFTER возврат не используется).

Пример (по документации PostgreSQL, адаптировано):

```sql
CREATE TABLE emp_audit (
  operation char(1) NOT NULL,
  stamp     timestamp NOT NULL,
  userid    text NOT NULL,
  empname   text NOT NULL,
  salary    integer
);

CREATE OR REPLACE FUNCTION process_emp_audit() RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    INSERT INTO emp_audit SELECT 'D', now(), current_user, OLD.*;
  ELSIF TG_OP = 'UPDATE' THEN
    INSERT INTO emp_audit SELECT 'U', now(), current_user, NEW.*;
  ELSIF TG_OP = 'INSERT' THEN
    INSERT INTO emp_audit SELECT 'I', now(), current_user, NEW.*;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER emp_audit
  AFTER INSERT OR UPDATE OR DELETE ON emp
  FOR EACH ROW EXECUTE FUNCTION process_emp_audit();
```

Для массовых операций можно использовать триггер **FOR EACH STATEMENT** с transition tables (REFERENCING OLD TABLE / NEW TABLE), чтобы обрабатывать все затронутые строки одним запросом — см. документацию PostgreSQL. См. [Глоссарий](glossary.md).

---

## 8.4.2. Заполнение полей: updated_at, updated_by

Часто нужно автоматически проставлять **дату и пользователя** последнего изменения (updated_at, updated_by) или создания (created_at, created_by). Подходит **BEFORE INSERT OR UPDATE** триггер **FOR EACH ROW**: в функции присваивают полям NEW значения current_timestamp и current_user и возвращают **RETURN NEW** (по документации PostgreSQL, пример 41.3 — emp_stamp). См. [Глоссарий](glossary.md).

Пример: два столбца — дата и пользователь (при INSERT заполняем оба, при UPDATE только «обновление»):

```sql
CREATE FUNCTION set_updated() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    NEW.created_at := current_timestamp;
    NEW.created_by := current_user;
  END IF;
  NEW.updated_at := current_timestamp;
  NEW.updated_by := current_user;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_set_updated
  BEFORE INSERT OR UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION set_updated();
```

Таблица orders должна содержать столбцы created_at, created_by, updated_at, updated_by (типы timestamp и text или аналоги). Одна и та же функция может использоваться для нескольких таблиц с такой же структурой полей.

---

## 8.4.3. Проверки и модификация NEW

В **BEFORE** триггере можно **проверять** данные и при нарушении правил вызывать **RAISE EXCEPTION** — тогда операция откатывается. Либо **изменить** NEW (исправить значение, подставить по умолчанию) и вернуть NEW; либо **отменить** операцию для строки, вернув **NULL**. См. [Глоссарий](glossary.md).

Пример проверки (из [§8.2](chapter-08-02.md)): запрет NULL в имени и отрицательной зарплаты:

```sql
IF NEW.empname IS NULL THEN
  RAISE EXCEPTION 'empname cannot be null';
END IF;
IF NEW.salary IS NOT NULL AND NEW.salary < 0 THEN
  RAISE EXCEPTION '% cannot have a negative salary', NEW.empname;
END IF;
```

Пример модификации: нормализация значения до записи (например, приведение к верхнему регистру, обрезка пробелов):

```sql
NEW.code := upper(trim(NEW.code));
RETURN NEW;
```

Сложные бизнес-правила и кросс-табличные проверки иногда лучше реализовать ограничениями (CHECK, UNIQUE, FK) или в приложении; триггер уместен, когда логика тесно связана с одной таблицей или требует доступа к OLD/NEW. См. [§8.5](chapter-08-05.md).

---

## Ключевое

- **Аудит**: AFTER FOR EACH ROW, по TG_OP пишем в таблицу логов операцию, время, пользователя и OLD/NEW; возврат NULL.
- **Заполнение полей** (updated_at, created_at и т.п.): BEFORE INSERT OR UPDATE, присваивание NEW и RETURN NEW.
- **Проверки**: BEFORE, при ошибке RAISE EXCEPTION; **модификация** — изменить NEW и RETURN NEW; **отмена для строки** — RETURN NULL.
- Для массового аудита можно использовать триггер по оператору с transition tables.

В [§8.5](chapter-08-05.md) разберём **ограничения и риски** триггеров: производительность, каскадные вызовы, когда предпочесть ограничения БД или проверки в приложении.
