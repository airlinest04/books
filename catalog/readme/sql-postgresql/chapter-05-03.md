# §5.3 FETCH FIRST ... ROWS ONLY

В [§5.2](chapter-05-02.md) мы рассмотрели LIMIT и OFFSET. **FETCH FIRST ... ROWS ONLY** — это стандартный SQL-синтаксис для ограничения числа строк; он эквивалентен LIMIT, но входит в стандарт SQL:2008. В этом разделе разберём синтаксис FETCH, варианты ONLY и WITH TIES, а также сравнение с LIMIT. Манипулирование данными (INSERT, UPDATE, DELETE) рассматривается в гл. 6. См. [документацию SELECT](https://www.postgresql.org/docs/current/sql-select.html).

---

## 5.3.1. Зачем нужен FETCH

LIMIT — расширение PostgreSQL и многих других СУБД. В стандарте SQL (с SQL:2008) для той же задачи используется конструкция **FETCH FIRST ... ROWS ONLY**. В PostgreSQL обе формы поддерживаются и дают одинаковый результат. FETCH полезен при переносе кода между СУБД или при желании придерживаться стандарта.

---

## 5.3.2. Базовый синтаксис

По [документации](https://www.postgresql.org/docs/current/sql-select.html#SQL-LIMIT):

```sql
SELECT столбцы FROM таблица [WHERE ...] [ORDER BY ...]
FETCH { FIRST | NEXT } [ n ] { ROW | ROWS } ONLY;
```

- **FIRST** и **NEXT** — синонимы, разницы нет.
- **ROW** и **ROWS** — синонимы; при `n > 1` обычно пишут ROWS.
- `n` — целое число; если опущено, по умолчанию 1.

Примеры:

```sql
SELECT id, name, price FROM products ORDER BY price
FETCH FIRST 5 ROWS ONLY;

SELECT id, name FROM users ORDER BY name
FETCH NEXT 10 ROWS ONLY;
```

---

## 5.3.3. FETCH с OFFSET

Стандарт требует, чтобы OFFSET шёл **перед** FETCH:

```sql
ORDER BY ...
OFFSET m { ROW | ROWS }
FETCH { FIRST | NEXT } [ n ] { ROW | ROWS } ONLY;
```

В PostgreSQL порядок OFFSET и FETCH может быть любым, но для совместимости со стандартом лучше придерживаться OFFSET перед FETCH.

Пример — страница 3 при 10 записях на странице:

```sql
SELECT id, name, created_at
FROM orders
ORDER BY created_at DESC
OFFSET 20 ROWS
FETCH FIRST 10 ROWS ONLY;
```

---

## 5.3.4. WITH TIES

Вариант **WITH TIES** вместо **ONLY** включает в результат не только первые `n` строк, но и все строки, которые по столбцам ORDER BY равны последней из этих `n` строк.

```sql
FETCH FIRST 5 ROWS WITH TIES;
```

Пример: «топ-5 по цене, включая все товары с той же ценой, что и пятый». Без WITH TIES вернутся ровно 5 строк. С WITH TIES — 5 или больше, если несколько товаров имеют одинаковую цену с пятым.

WITH TIES требует ORDER BY; причём столбцы ORDER BY определяют, что считать «равным» последней строке.

---

## 5.3.5. Сравнение с LIMIT

| FETCH | LIMIT |
|-------|-------|
| Стандарт SQL:2008 | Расширение PostgreSQL |
| `FETCH FIRST n ROWS ONLY` | `LIMIT n` |
| `OFFSET m ROWS FETCH FIRST n ROWS ONLY` | `LIMIT n OFFSET m` |
| `FETCH FIRST n ROWS WITH TIES` | Нет прямой замены |

Функционально `FETCH FIRST n ROWS ONLY` эквивалентно `LIMIT n`. LIMIT короче и привычнее для многих; FETCH удобнее для кросс-платформенного кода.

---

## 5.3.6. Когда использовать FETCH

- **Переносимость** — при работе с разными СУБД (Oracle, SQL Server, DB2 поддерживают FETCH) стандартный синтаксис упрощает перенос.
- **WITH TIES** — когда нужно включить все «ничьи» на границе топа.
- **Стиль** — при желании следовать стандарту SQL.

В проектах, ориентированных только на PostgreSQL, LIMIT обычно встречается чаще из-за краткости.

---

## 5.3.7. Типичные ошибки

- **FETCH без ORDER BY** — порядок не определён; для воспроизводимого результата ORDER BY обязателен.
- **WITH TIES без ORDER BY** — приведёт к ошибке.
- **Путаница FIRST и NEXT** — семантической разницы нет, можно использовать любой вариант.

---

## Ключевое

- **FETCH FIRST n ROWS ONLY** (или **FETCH NEXT n ROWS ONLY**) — стандартный SQL-способ ограничить число строк; эквивалентен `LIMIT n`.
- OFFSET по стандарту идёт перед FETCH; PostgreSQL допускает любой порядок.
- **WITH TIES** — включает строки, равные последней по ORDER BY; требует ORDER BY.
- FETCH удобен для переносимости и при использовании WITH TIES; LIMIT короче и распространён в PostgreSQL.

В [§6.1](chapter-06-01.md) рассмотрим INSERT — добавление новых строк в таблицу.
