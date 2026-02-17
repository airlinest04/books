# §7.4 Возврат набора из функции

В [§7.3](chapter-07-03.md) мы разобрали динамический SQL. В этом разделе сводим воедино **возврат набора строк** из функции PL/pgSQL: объявление **RETURNS SETOF** и **RETURNS TABLE**, операторы **RETURN QUERY** и **RETURN QUERY EXECUTE**, вызов функции в FROM и проверка **FOUND** после RETURN QUERY. Основы (RETURN NEXT, RETURN QUERY) даны в [§5.3](chapter-05-03.md); здесь — акцент на табличных функциях и связи с запросами и динамическим SQL. С гл. 8 начинаются триггеры. См. [Глоссарий](glossary.md).

---

## 7.4.1. RETURNS SETOF и RETURNS TABLE

Функция, возвращающая **набор строк**, объявляется как **RETURNS SETOF тип** или **RETURNS TABLE ( столбец тип [, ...] )** (по документации PostgreSQL). См. [Глоссарий](glossary.md).

**RETURNS SETOF тип** — тип элемента набора: скаляр (integer, text), составной тип (имя таблицы, тип записи) или **record**. При вызове `SELECT * FROM func()` столбцы результата именуются по полям типа (для таблицы — имена столбцов таблицы; для record при одном OUT — имя OUT, при нескольких — имена OUT).

**RETURNS TABLE ( имя_столбца тип [, ...] )** — явно задаёт имена и типы столбцов результата. Удобно, когда структура не совпадает с существующей таблицей или нужно задать свои имена. Количество и типы столбцов в RETURN QUERY (или RETURN NEXT) должны соответствовать объявлению TABLE.

Примеры:

```sql
-- строки как тип таблицы foo
CREATE FUNCTION get_foo() RETURNS SETOF foo AS $$
  BEGIN RETURN QUERY SELECT * FROM foo; RETURN; END;
$$ LANGUAGE plpgsql;

-- явные имена столбцов
CREATE FUNCTION get_stats(p_id int) RETURNS TABLE (order_count bigint, total_sum numeric) AS $$
  BEGIN
    RETURN QUERY SELECT count(*), sum(amount) FROM orders WHERE user_id = p_id;
    RETURN;
  END;
$$ LANGUAGE plpgsql;
```

Вызов — в FROM, как у таблицы или подзапроса: `SELECT * FROM get_foo();`, `SELECT * FROM get_stats(1) WHERE total_sum > 100;`.

---

## 7.4.2. RETURN QUERY и RETURN QUERY EXECUTE

**RETURN QUERY запрос;** выполняет запрос и **добавляет все его строки** к результату функции; выполнение функции продолжается (по документации PostgreSQL, раздел 41.6.1.2). Запрос пишется как обычный SQL; переменные PL/pgSQL подставляются. После RETURN QUERY можно проверить **FOUND** — true, если запрос вернул хотя бы одну строку. См. [Глоссарий](glossary.md).

Пример с проверкой FOUND (по документации PostgreSQL):

```sql
CREATE FUNCTION get_available_flightid(d date) RETURNS SETOF integer AS $$
BEGIN
  RETURN QUERY SELECT flightid FROM flight
    WHERE flightdate >= d AND flightdate < (d + 1);
  IF NOT FOUND THEN
    RAISE EXCEPTION 'No flight at %.', d;
  END IF;
  RETURN;
END;
$$ LANGUAGE plpgsql;
```

**RETURN QUERY EXECUTE строка_запроса [ USING выражение, ... ]** — запрос задаётся строкой (динамический SQL); параметры подставляются через USING, как в EXECUTE ([§7.3](chapter-07-03.md)). Имена объектов в строке собирают через quote_ident или format(%I). Пример:

```sql
CREATE FUNCTION query_table(tbl text, status_val text) RETURNS SETOF record AS $$
BEGIN
  RETURN QUERY EXECUTE format('SELECT id, name FROM %I WHERE status = $1', tbl)
    USING status_val;
  RETURN;
END;
$$ LANGUAGE plpgsql;

-- вызов: указать структуру результата
SELECT * FROM query_table('orders', 'new') AS t(id int, name text);
```

RETURN NEXT и RETURN QUERY можно комбинировать в одной функции — строки добавляются к общему набору; завершает функцию **RETURN;** без аргумента.

---

## 7.4.3. Накопление результата и вызов в запросе

Текущая реализация PL/pgSQL **сначала накапливает весь набор** результата (в памяти или на диске при нехватке work_mem), и только затем передаёт его вызывающему. Функция не завершится, пока не сформирует все строки (документация PostgreSQL, раздел 41.6.1.2). Для очень больших наборов это может сказываться на памяти и времени до первой строки. См. [Глоссарий](glossary.md).

Функцию, возвращающую набор, вызывают в **FROM** (или в LATERAL подзапросе): она ведёт себя как таблица. Можно фильтровать, соединять, ограничивать:

```sql
SELECT * FROM get_foo() WHERE fooname LIKE 'a%';
SELECT * FROM get_stats(1) AS s WHERE s.total_sum > 0;
```

Псевдоним таблицы (AS s) при необходимости задаёт имена столбцов для конфликтующих или неоднозначных имён. Для RETURNS SETOF record при вызове нужно задать список столбцов с типами: `... AS alias(col1 type, col2 type)`.

---

## Ключевое

- **RETURNS SETOF тип** или **RETURNS TABLE ( столбец тип, ... )** — функция возвращает набор строк; вызов в FROM: `SELECT * FROM func(...)`.
- **RETURN QUERY** запрос — добавить все строки запроса к результату; после него **FOUND** показывает, была ли хотя бы одна строка.
- **RETURN QUERY EXECUTE** строка **[ USING ... ]** — динамический запрос; параметры через USING, имена объектов — через quote_ident/format(%I).
- Результат накапливается целиком до выхода из функции; для больших объёмов это может влиять на память и производительность.

В [§8.1](chapter-08-01.md) начнём главу о **триггерах**: события (BEFORE/AFTER, INSERT/UPDATE/DELETE), срабатывание по строке и по оператору.
