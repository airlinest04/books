# §6.3 Специфичные для Oracle

В [§6.2](chapter-06-02.md) мы рассмотрели общие оконные функции (OVER, PARTITION BY, ROW_NUMBER, RANK, DENSE_RANK, агрегаты с окном). В Oracle есть дополнительные функции для агрегации и анализа, удобные в отчётах и ETL. В этом разделе — **LISTAGG** (сбор строк в одну с разделителем), **NTH_VALUE** (значение N-й строки в окне), а также кратко **FIRST_VALUE** и **LAST_VALUE**. Глава 6 на этом завершается; далее [глава 7](chapter-07-01.md): CTE, множества, представления. См. [Глоссарий](glossary.md).

---

## 6.3.1. LISTAGG: объединение строк в одну

**LISTAGG** — групповая (или оконная) функция Oracle, которая объединяет значения столбца или выражения из нескольких строк в одну строку с указанным разделителем. Типичное применение: «список имён в отделе через запятую», «все коды через точку с запятой». См. [Глоссарий](glossary.md).

Синтаксис (упрощённо):

```sql
LISTAGG(выражение [, разделитель]) [WITHIN GROUP (ORDER BY столбец [ASC|DESC], ...)]
```

В контексте **GROUP BY** LISTAGG собирает значения по каждой группе; порядок элементов задаётся **WITHIN GROUP (ORDER BY ...)**. Разделитель по умолчанию — NULL (конкатенация без разделителя); обычно задают запятую или точку с запятой.

Пример: по каждому отделу вывести список фамилий сотрудников через запятую, отсортированный по фамилии:

```sql
SELECT department_id,
       LISTAGG(last_name, ', ') WITHIN GROUP (ORDER BY last_name) AS employees_list
  FROM employees
 GROUP BY department_id
 ORDER BY department_id;
```

Результат: одна строка на отдел с столбцом employees_list вида `King, Kochhar, De Haan`. LISTAGG также поддерживает **OVER (PARTITION BY ...)** — тогда в каждой строке партиции будет одна и та же собранная строка по этой партиции. Среда: Oracle 11g R2 и новее (в 11g R1 — без ORDER BY в WITHIN GROUP).

Ограничение: результат LISTAGG — строка; максимальная длина в байтах определяется параметром **VARCHAR2** (до 4000 в стандартной кодировке). При большом числе значений возможна ошибка ORA-01489. Решения: подзапрос с ROWNUM и обрезка, или агрегация по подгруппам, или использование в приложении.

---

## 6.3.2. NTH_VALUE: значение N-й строки в окне

**NTH_VALUE(выражение, n)** — оконная функция Oracle: возвращает значение выражения из **N-й строки** в окне (по ORDER BY в OVER). Полезно для запросов вида «вторая по величине зарплата в отделе», «третий по дате заказ». См. [Глоссарий](glossary.md).

Синтаксис:

```sql
NTH_VALUE(выражение, n) [FROM FIRST | FROM LAST]
  OVER ([PARTITION BY ...] ORDER BY ... [окно_кадра])
```

**n** — положительное целое (1 — первая строка, 2 — вторая и т.д.). **FROM FIRST** (по умолчанию) — отсчёт от начала партиции по ORDER BY; **FROM LAST** — от конца. Если в окне меньше n строк, возвращается NULL. Окно кадра (ROWS/RANGE) при необходимости сужает, какие строки входят в партицию для расчёта.

Пример: для каждого сотрудника вывести его зарплату и вторую по величине зарплату в его отделе (в каждой строке отдела одно и то же значение «вторая зарплата»):

```sql
SELECT department_id, last_name, salary,
       NTH_VALUE(salary, 2) FROM FIRST
         OVER (PARTITION BY department_id ORDER BY salary DESC
               ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS second_salary
  FROM employees
 ORDER BY department_id, salary DESC;
```

Кадр UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING нужен, чтобы NTH_VALUE смотрела на всю партицию, а не только на строки «до текущей». Иначе в первой строке партиции была бы только одна строка в кадре и NTH_VALUE(salary, 2) дала бы NULL. Среда: Oracle 11g R2 и новее.

---

## 6.3.3. FIRST_VALUE и LAST_VALUE

**FIRST_VALUE(выражение)** и **LAST_VALUE(выражение)** — оконные функции: возвращают значение выражения из **первой** или **последней** строки в окне (по ORDER BY в OVER). По смыслу FIRST_VALUE(expr) эквивалентна NTH_VALUE(expr, 1) FROM FIRST, LAST_VALUE — NTH_VALUE(expr, 1) FROM LAST, но без явного указания кадра поведение по умолчанию может отличаться: LAST_VALUE часто требует кадра до конца партиции (UNBOUNDED FOLLOWING), иначе «последняя» строка будет только до текущей. См. [Глоссарий](glossary.md).

Пример: в каждом отделе вывести сотрудника, его зарплату и максимальную зарплату в отделе (через LAST_VALUE по партиции с ORDER BY salary и кадром до конца):

```sql
SELECT department_id, last_name, salary,
       LAST_VALUE(salary) OVER (
         PARTITION BY department_id ORDER BY salary
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS max_salary_in_dept
  FROM employees
 ORDER BY department_id, salary;
```

Для «максимум по партиции в каждой строке» проще использовать **MAX(salary) OVER (PARTITION BY department_id)** без ORDER BY. FIRST_VALUE/LAST_VALUE удобны, когда нужно именно значение из первой/последней строки окна (например, первая или последняя дата в группе).

---

## Ключевое

- **LISTAGG(выражение [, разделитель]) WITHIN GROUP (ORDER BY ...)** — объединение значений строк группы в одну строку с разделителем. Групповая с GROUP BY или оконная с OVER. Ограничение по длине результата (VARCHAR2). См. [Глоссарий](glossary.md).
- **NTH_VALUE(выражение, n)** — значение выражения из N-й строки в окне (ORDER BY в OVER). FROM FIRST / FROM LAST; при числе строк меньше n — NULL. Для доступа ко всей партиции задают кадр ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING. См. [Глоссарий](glossary.md).
- **FIRST_VALUE** и **LAST_VALUE** — значение из первой или последней строки окна. LAST_VALUE часто требует явного кадра до конца партиции.
- Функции LISTAGG, NTH_VALUE, FIRST_VALUE, LAST_VALUE — часть аналитических возможностей Oracle; в стандартном SQL их нет или синтаксис иной.

В [§7.1](chapter-07-01.md) мы начнём главу о сложных запросах: CTE (WITH), именованные подзапросы и рекурсия.
