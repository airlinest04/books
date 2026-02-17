# §12.2 OVER ()

В [§12.1](chapter-12-01.md) мы рассмотрели идею оконных функций. **OVER** задаёт окно — набор строк, по которому вычисляется функция. В этом разделе разберём синтаксис OVER: пустое окно, PARTITION BY и ORDER BY внутри OVER. Ранжирование — в [§12.3](chapter-12-03.md). См. [документацию Value Expressions](https://www.postgresql.org/docs/current/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS) и [туториал](https://www.postgresql.org/docs/current/tutorial-window.html).

---

## 12.2.1. Базовый синтаксис

```sql
функция(...) OVER ([PARTITION BY выражение [, ...]] [ORDER BY выражение [ASC|DESC] [, ...]])
```

В скобках после OVER указывают, как разбить строки и в каком порядке их обрабатывать. ORDER BY в OVER **не задаёт** порядок вывода запроса — только порядок обработки внутри окна.

---

## 12.2.2. OVER () — окно по всей таблице

Пустое **OVER ()** — одно окно из **всех** строк результата запроса (после WHERE, JOIN и т.д.). Разбиения нет.

Пример: сумма зарплат по всей таблице в каждой строке:

```sql
SELECT depname, empno, salary,
  sum(salary) OVER () AS total_salary
FROM empsalary;
```

В `total_salary` во всех строках будет одна и та же общая сумма. Число строк не меняется.

---

## 12.2.3. PARTITION BY — разбиение на части

**PARTITION BY** разбивает строки на **партиции** (группы) по значениям выражений. Для каждой строки функция считается только по строкам **её** партиции.

```sql
SELECT depname, empno, salary,
  avg(salary) OVER (PARTITION BY depname) AS avg_in_dept
FROM empsalary;
```

Строки с одинаковым `depname` попадают в одну партицию. `avg(salary)` в каждой строке — среднее по отделу этой строки. Без PARTITION BY окно — вся таблица.

---

## 12.2.4. ORDER BY внутри OVER

**ORDER BY** в OVER задаёт порядок обработки строк в партиции. От него зависит:

- для ранжирующих функций (ROW_NUMBER, RANK и т.д.) — порядок присвоения номеров;
- для агрегатов — границы **рамки** (frame): по умолчанию от начала партиции до текущей строки включительно, что даёт накопительный итог.

Пример без ORDER BY — одно и то же среднее во всех строках партиции:

```sql
avg(salary) OVER (PARTITION BY depname)
```

С ORDER BY — накопительное среднее (от первой до текущей строки):

```sql
avg(salary) OVER (PARTITION BY depname ORDER BY empno)
```

ORDER BY в OVER **не определяет** порядок вывода всего запроса. Порядок вывода задаётся только в основном ORDER BY в конце запроса.

---

## 12.2.5. Комбинация PARTITION BY и ORDER BY

Оба предложения можно использовать вместе:

```sql
SELECT depname, empno, salary,
  sum(salary) OVER (PARTITION BY depname ORDER BY salary) AS running_sum_in_dept
FROM empsalary;
```

Для каждой строки `running_sum_in_dept` — сумма зарплат в отделе от самой маленькой до текущей (включительно). В каждом отделе свой нарастающий итог.

---

## 12.2.6. Именованные окна (WINDOW)

Если одно и то же окно нужно в нескольких функциях, его можно задать в **WINDOW** и ссылаться по имени:

```sql
SELECT sum(salary) OVER w, avg(salary) OVER w
FROM empsalary
WINDOW w AS (PARTITION BY depname ORDER BY salary);
```

`OVER w` эквивалентно повторению `OVER (PARTITION BY depname ORDER BY salary)`.

---

## Ключевое

- **OVER ()** — окно по всем строкам результата.
- **PARTITION BY** разбивает строки на партиции; функция считается по партиции текущей строки.
- **ORDER BY** в OVER задаёт порядок внутри партиции и (для агрегатов) рамку «от начала до текущей строки».
- ORDER BY в OVER не задаёт порядок вывода запроса — только обработку в окне.
- **WINDOW** позволяет вынести общее описание окна и переиспользовать его.

В [§12.3](chapter-12-03.md) рассмотрим функции ранжирования: ROW_NUMBER, RANK, DENSE_RANK, NTILE.
