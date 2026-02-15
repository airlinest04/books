# §10.4 Порядок выполнения и скобки

В [§10.3](chapter-10-03.md) мы рассмотрели EXCEPT. При комбинировании нескольких операций (UNION, INTERSECT, EXCEPT) важен **порядок выполнения** — он определяет, какой результат получится. В этом разделе разберём приоритет операторов, ассоциативность и использование скобок для явного управления порядком. Глава 11 — транзакции и целостность чтения ([§11.1](chapter-11-01.md)). См. [документацию Combining Queries](https://www.postgresql.org/docs/current/queries-union.html) и [SELECT](https://www.postgresql.org/docs/current/sql-select.html).

---

## 10.4.1. Приоритет операторов

По [документации](https://www.postgresql.org/docs/current/queries-union.html) PostgreSQL:

- **INTERSECT** связывает **крепче**, чем UNION и EXCEPT.
- **UNION** и **EXCEPT** выполняются **слева направо** при одинаковом приоритете.

Пример:

```sql
SELECT 1 AS x
UNION
SELECT 2
INTERSECT
SELECT 2;
```

Интерпретация: `SELECT 1 UNION (SELECT 2 INTERSECT SELECT 2)`. Внутренний INTERSECT даёт строку 2; UNION объединяет {1} и {2} — результат {1, 2}. Без учёта приоритета `(SELECT 1 UNION SELECT 2) INTERSECT SELECT 2` дало бы только {2}.

---

## 10.4.2. Ассоциативность UNION и EXCEPT

UNION и EXCEPT объединяются слева направо:

```sql
запрос1 UNION запрос2 EXCEPT запрос3
```

Эквивалентно:

```sql
(запрос1 UNION запрос2) EXCEPT запрос3
```

Сначала объединяются результаты первого и второго запросов, затем из них вычитается результат третьего.

---

## 10.4.3. Скобки для явного порядка

Скобки задают порядок вычисления. Разный порядок — разный результат:

```sql
-- Сначала INTERSECT, затем UNION:
(SELECT id FROM table_a INTERSECT SELECT id FROM table_b)
UNION
SELECT id FROM table_c;

-- Сначала UNION первых двух, затем INTERSECT с третьим:
SELECT id FROM table_a
UNION
(SELECT id FROM table_b INTERSECT SELECT id FROM table_c);
```

Скобки полезны, когда приоритет по умолчанию не совпадает с задуманной логикой, и для читаемости.

---

## 10.4.4. Скобки вокруг отдельных запросов

ORDER BY, LIMIT, OFFSET в конце относятся ко **всему** результату операций над множествами. Чтобы применить их только к одному из запросов, этот запрос нужно взять в скобки:

```sql
-- LIMIT применяется ко всему UNION:
SELECT name FROM customers
UNION
SELECT name FROM employees
LIMIT 5;

-- LIMIT только к employees:
SELECT name FROM customers
UNION
(SELECT name FROM employees LIMIT 5);
```

Без скобок `LIMIT 5` во втором варианте было бы синтаксической ошибкой или трактовалось бы как ограничение всего результата.

---

## 10.4.5. Рекомендация: скобки для ясности

Даже когда приоритет по умолчанию даёт нужный результат, скобки улучшают читаемость:

```sql
(SELECT customer_id FROM orders)
EXCEPT
(SELECT customer_id FROM refunds);
```

Для сложных комбинаций из трёх и более запросов скобки почти обязательны, чтобы не путаться в порядке выполнения.

---

## Ключевое

- **INTERSECT** связывает крепче, чем UNION и EXCEPT; UNION и EXCEPT выполняются слева направо.
- `A UNION B INTERSECT C` = `A UNION (B INTERSECT C)`; `A UNION B EXCEPT C` = `(A UNION B) EXCEPT C`.
- Скобки задают явный порядок и повышают читаемость.
- ORDER BY, LIMIT, OFFSET в конце относятся ко всему результату; для одного запроса — скобки вокруг него.

В [§11.1](chapter-11-01.md) начнём главу о транзакциях — BEGIN, COMMIT, ROLLBACK.
