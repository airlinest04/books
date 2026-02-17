# §4.3 Примеры unit-тестов для запросов

В [§4.2](chapter-04-02.md) мы разобрали структуру теста Arrange — Act — Assert и привели полный пример теста агрегации. Здесь даём ещё три типа примеров: **тест агрегации** (кратко, с вариантом на Oracle), **тест джойна** и **тест фильтра** — на PostgreSQL и с указанием отличий для Oracle. Все примеры можно адаптировать под свою схему и тест-раннер.

---

## 4.3.1. Тест агрегации (напоминание и вариант на Oracle)

Тест агрегации проверяет: при заданных строках в таблице запрос с GROUP BY и SUM/COUNT/AVG возвращает ожидаемые группы и значения. В §4.2 уже был пример на PostgreSQL (сумма продаж по region_id). Ниже — тот же сценарий в виде, удобном для Oracle (PL/SQL блок с проверкой через переменные).

Среда: Oracle (SQL*Plus или SQL Developer). Схема теста — отдельный пользователь или префикс таблиц, например `TEST_SALES`.

```sql
-- Oracle: unit-тест агрегации SUM по группе
-- Arrange
TRUNCATE TABLE test_sales;
INSERT INTO test_sales (id, region_id, amount) VALUES (1, 10, 100);
INSERT INTO test_sales (id, region_id, amount) VALUES (2, 10, 200);
INSERT INTO test_sales (id, region_id, amount) VALUES (3, 20, 150);
INSERT INTO test_sales (id, region_id, amount) VALUES (4, 20, 250);
COMMIT;

-- Act: результат во временную таблицу или в курсор
-- Assert
DECLARE
  v_cnt   PLS_INTEGER;
  v_r10   NUMBER;
  v_r20   NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_cnt FROM (
    SELECT region_id, SUM(amount) AS total FROM test_sales GROUP BY region_id
  );
  IF v_cnt != 2 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Expected 2 rows, got ' || v_cnt);
  END IF;

  SELECT total INTO v_r10 FROM (
    SELECT region_id, SUM(amount) AS total FROM test_sales GROUP BY region_id
  ) WHERE region_id = 10;
  SELECT total INTO v_r20 FROM (
    SELECT region_id, SUM(amount) AS total FROM test_sales GROUP BY region_id
  ) WHERE region_id = 20;

  IF v_r10 != 300 OR v_r20 != 400 THEN
    RAISE_APPLICATION_ERROR(-20002, 'Expected 300 and 400, got ' || v_r10 || ' and ' || v_r20);
  END IF;
END;
/
```

В Oracle для прерывания теста при ошибке используют `RAISE_APPLICATION_ERROR`. Подзапрос в SELECT повторяется; в реальном тесте лучше вынести результат агрегации во временную таблицу (CREATE GLOBAL TEMPORARY TABLE или таблицу в тестовой схеме) и проверять её, чтобы не дублировать запрос.

---

## 4.3.2. Тест джойна

Проверяем запрос, который соединяет две таблицы: заказы и клиенты. Ожидание: тип джойна (LEFT) сохраняет заказы без клиента с NULL в полях клиента; ключ соединения верный; количество строк и состав полей совпадают с эталоном.

Среда: PostgreSQL.

```sql
-- Unit-тест: LEFT JOIN orders -> customers
-- Arrange
CREATE TABLE IF NOT EXISTS test.customers (
    customer_id INT PRIMARY KEY,
    name TEXT
);
CREATE TABLE IF NOT EXISTS test.orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    amount NUMERIC(12,2)
);

TRUNCATE test.orders, test.customers;
INSERT INTO test.customers (customer_id, name) VALUES
(1, 'Alice'), (2, 'Bob');
INSERT INTO test.orders (order_id, customer_id, amount) VALUES
(101, 1, 50.00), (102, 2, 100.00), (103, 999, 25.00);  -- 999 нет в customers

-- Act: заказы с именами клиентов; заказ без клиента должен остаться с NULL name
DROP TABLE IF EXISTS test.result;
CREATE TABLE test.result AS
SELECT o.order_id, o.customer_id, c.name AS customer_name, o.amount
FROM test.orders o
LEFT JOIN test.customers c ON c.customer_id = o.customer_id
ORDER BY o.order_id;

-- Assert: 3 строки; order_id=103 -> customer_name IS NULL
DO $$
DECLARE cnt INT; n TEXT;
BEGIN
    SELECT COUNT(*) INTO cnt FROM test.result;
    IF cnt != 3 THEN RAISE EXCEPTION 'Expected 3 rows, got %', cnt; END IF;

    SELECT customer_name INTO n FROM test.result WHERE order_id = 103;
    IF n IS NOT NULL THEN RAISE EXCEPTION 'Expected NULL customer_name for order 103, got %', n; END IF;

    -- Проверка значений для заказов с клиентами
    IF (SELECT customer_name FROM test.result WHERE order_id = 101) != 'Alice' THEN
        RAISE EXCEPTION 'Expected Alice for order 101';
    END IF;
    IF (SELECT amount FROM test.result WHERE order_id = 102) != 100 THEN
        RAISE EXCEPTION 'Expected amount=100 for order 102';
    END IF;
    RAISE NOTICE 'Join test passed.';
END $$;
```

Фикстура намеренно содержит «сироту» (customer_id=999 без строки в customers); тест проверяет, что LEFT JOIN не теряет эту строку и подставляет NULL в customer_name. Для Oracle тот же сценарий: те же INSERT (при необходимости поправить синтаксис типов), запрос с LEFT JOIN и блок PL/SQL с проверками и RAISE_APPLICATION_ERROR при ошибке.

---

## 4.3.3. Тест фильтра

Проверяем отбор строк по условию: только заказы со статусом 'completed' и суммой больше 0. Граничные случаи: amount=0 не должен попадать, status=NULL не должен попадать (если по логике null не считается completed).

Среда: PostgreSQL.

```sql
-- Unit-тест: фильтр status = 'completed' AND amount > 0
-- Arrange
CREATE TABLE IF NOT EXISTS test.orders (
    order_id INT, status TEXT, amount NUMERIC(12,2)
);
TRUNCATE test.orders;
INSERT INTO test.orders (order_id, status, amount) VALUES
(1, 'completed', 100),
(2, 'completed', 0),      -- граница: 0 не должен пройти
(3, 'pending', 50),
(4, 'completed', 200),
(5, NULL, 10);            -- null статус не должен пройти

-- Act
DROP TABLE IF EXISTS test.result;
CREATE TABLE test.result AS
SELECT order_id, status, amount
FROM test.orders
WHERE status = 'completed' AND amount > 0
ORDER BY order_id;

-- Assert: только order_id 1 и 4 (две строки)
DO $$
DECLARE cnt INT; ids INT[];
BEGIN
    SELECT COUNT(*) INTO cnt FROM test.result;
    IF cnt != 2 THEN RAISE EXCEPTION 'Expected 2 rows, got %', cnt; END IF;

    SELECT ARRAY_AGG(order_id ORDER BY order_id) INTO ids FROM test.result;
    IF ids != ARRAY[1, 4] THEN RAISE EXCEPTION 'Expected order_id 1 and 4, got %', ids; END IF;
END $$;
```

В Oracle массив можно заменить проверкой по одной строке: «есть строка с order_id=1», «есть строка с order_id=4», «нет строк с order_id IN (2,3,5)». Альтернатива — эталонная таблица ожидаемых order_id и проверка через MINUS (в Oracle) или EXCEPT (в PostgreSQL): разность между результатом и эталоном должна быть пустой.

---

## 4.3.4. Сводка по примерам и перенос на свою среду

| Пример | Что проверяем | Ключевая фикстура | Assert |
|--------|----------------|-------------------|--------|
| Агрегация | SUM, GROUP BY | Строки с известными region_id и amount | Количество групп и суммы по группам |
| Джойн | LEFT JOIN, сохранение строк без пары | Заказы с «сиротой» (customer_id без строки в customers) | 3 строки, у одной customer_name IS NULL |
| Фильтр | WHERE по статусу и сумме | Строки на границе (amount=0, status=NULL) | Только ожидаемые order_id (2 строки) |

Перенос на свою среду:

- **Схема:** заменить `test` на вашу тестовую схему или префикс; в Oracle при необходимости использовать GLOBAL TEMPORARY TABLE для результата.
- **Запуск:** выполнять скрипт через psql, SQL*Plus, или вызывать из pytest/Java и проверять код возврата или исключение; при RAISE EXCEPTION / RAISE_APPLICATION_ERROR тест считается проваленным.
- **Изоляция:** перед каждым тестом очищать таблицы (TRUNCATE) или пересоздавать схему, чтобы не зависеть от порядка запуска тестов.

В [§4.4](chapter-04-04.md) мы разберём запуск тестов вручную и по расписанию: скрипты, интеграция с Airflow и отчёты о результатах.

---

## Ключевое

- **Тест агрегации:** фикстура с известными группами и значениями; проверка количества групп и агрегатов (SUM/COUNT) по каждой группе; на Oracle — PL/SQL с RAISE_APPLICATION_ERROR при ошибке.
- **Тест джойна:** фикстура с «сиротой» (ключ без пары во второй таблице); проверка, что LEFT JOIN сохраняет строку и даёт NULL в полях правой таблицы; проверка количества строк и выборочных значений.
- **Тест фильтра:** фикстура с граничными значениями (0, NULL, не проходящие условие); проверка, что в результате только ожидаемые строки (по количеству и по ключам).
- Примеры даны для PostgreSQL с возможностью переноса на Oracle (синтаксис блоков и ошибок); схему, имена таблиц и способ запуска можно адаптировать под свой проект.

В [§4.4](chapter-04-04.md) мы разберём запуск тестов вручную и по расписанию: скрипты, интеграция с Airflow, результаты и отчёты.
