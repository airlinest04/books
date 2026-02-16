# §7.2 Оконные функции

В [§7.1](chapter-07-01.md) мы разобрали группировку и агрегацию (groupBy, agg). Иногда нужно не «схлопнуть» строки в одну на группу, а вычислить для **каждой строки** значение, зависящее от соседних строк или от всей группы (ранг, накопленная сумма, сдвиг на одну строку). Для этого служат **оконные функции**: задаётся **окно** (partitionBy + orderBy), и функция применяется в рамках этого окна. В этом разделе рассмотрим **Window.partitionBy().orderBy()**, функции **row_number()**, **rank()**, **dense_rank()**, **lag** и **lead**, а также агрегаты **over** окно. В [§7.3](chapter-07-03.md) перейдём к соединениям. См. [Глоссарий](glossary.md).

---

## 7.2.1. Окно: partitionBy и orderBy

**Оконная функция** — функция, которая для каждой строки вычисляет значение по подмножеству строк (окну), связанных с этой строкой. Окно задаётся через **Window**: разбиение по ключу (**partitionBy**) и порядок строк внутри партиции (**orderBy**). Опционально можно задать рамки кадра (range/rows between); по умолчанию — от начала партиции до текущей строки для агрегатов. См. [Глоссарий](glossary.md).

Импорт и пример спецификации окна:

```python
from pyspark.sql import Window
from pyspark.sql.functions import col

# Окно: по region, внутри региона сортировка по date
window_spec = Window.partitionBy("region").orderBy("date")
```

**partitionBy(*cols)** — аналог GROUP BY по ключу: строки с одинаковым ключом образуют одну партицию (логическое разбиение; не путать с партициями Spark). **orderBy(*cols)** задаёт порядок строк внутри партиции; от него зависят row_number, rank, lag, lead и рамка кадра для агрегатов. См. [§2.3](chapter-02-03.md), [Глоссарий](glossary.md).

---

## 7.2.2. row_number(), rank(), dense_rank()

Три функции присваивают **порядковый номер** строке внутри партиции в соответствии с orderBy:

| Функция | Поведение |
|---------|-----------|
| **row_number()** | Уникальный номер 1, 2, 3, … при равных значениях orderBy порядок недетерминирован. |
| **rank()** | Одинаковый ранг для равных; следующие номера пропускаются (1, 2, 2, 4, …). |
| **dense_rank()** | Одинаковый ранг для равных; номера идут подряд (1, 2, 2, 3, …). |

Использование с ранее определённым окном:

```python
from pyspark.sql.functions import row_number, rank, dense_rank

df.withColumn("rn", row_number().over(window_spec)) \
  .withColumn("rk", rank().over(window_spec)) \
  .withColumn("dr", dense_rank().over(window_spec))
```

Типичное применение: «последняя запись по ключу» — partitionBy(key), orderBy(date.desc()) и row_number() = 1; дедупликация или выбор одного представителя группы. См. [Глоссарий](glossary.md).

---

## 7.2.3. lag и lead

**lag(column, offset, default)** — значение столбца в строке, отстоящей на **offset** строк **назад** в порядке orderBy; если такой строки нет (например, первая в партиции), возвращается **default** (по умолчанию null). **lead(column, offset, default)** — значение в строке на **offset** строк **вперёд**. См. [Глоссарий](glossary.md).

```python
from pyspark.sql.functions import lag, lead

df.withColumn("prev_amount", lag("amount", 1).over(window_spec)) \
  .withColumn("next_amount", lead("amount", 1).over(window_spec))
```

Применение: разность «текущее — предыдущее» (дельта), скользящие сравнения, доступ к соседним событиям по времени. См. [§6.2](chapter-06-02.md).

---

## 7.2.4. Агрегаты over окно

Любую агрегатную функцию (sum, avg, count, min, max) можно вызвать **по окну**: не группируя строки в одну, а вычислив для каждой строки агрегат по кадру (например, «все строки от начала партиции до текущей»). Синтаксис — **функция(column).over(window_spec)**. См. [§7.1](chapter-07-01.md), [Глоссарий](glossary.md).

```python
from pyspark.sql.functions import sum as spark_sum

df.withColumn("running_total", spark_sum("amount").over(window_spec))
```

По умолчанию кадр для агрегатов — **rows between unbounded preceding and current row** (от первой строки партиции до текущей). Чтобы задать другое окно (например, «последние 3 строки»), к спецификации добавляют **.rowsBetween(-2, 0)** или **.rangeBetween(...)**. См. [Глоссарий](glossary.md).

---

## 7.2.5. Кадр: rowsBetween и rangeBetween

**Кадр (frame)** — подмножество строк внутри партиции, по которому считается агрегат или некоторые функции. Задаётся методами **rowsBetween(start, end)** и **rangeBetween(start, end)** у Window. См. [Глоссарий](glossary.md).

- **rowsBetween(start, end)** — по номерам строк: **Window.unboundedPreceding**, **Window.currentRow**, целые числа (отрицательные — назад от текущей строки). Пример: «текущая и две предыдущие» — `.orderBy("date").rowsBetween(-2, 0)`.
- **rangeBetween(start, end)** — по значениям orderBy (логический диапазон); единицы в тех же единицах, что и столбец orderBy (например, дни для типа date).

Пример: накопленная сумма по партиции — окно с orderBy и кадром по умолчанию; скользящее среднее по последним 3 строкам — orderBy + rowsBetween(-2, 0). См. [§7.1](chapter-07-01.md).

---

## Ключевое

- **Оконная функция** вычисляет значение для каждой строки по окну (партиция + порядок + опционально кадр). Спецификация — **Window.partitionBy(...).orderBy(...)** и при необходимости **.rowsBetween()** / **.rangeBetween()**.
- **row_number()**, **rank()**, **dense_rank()** — нумерация внутри партиции; row_number уникален, rank с пропусками, dense_rank без пропусков при равенстве.
- **lag(col, n)** и **lead(col, n)** — значение в строке на n позиций назад/вперёд по orderBy; для доступа к соседним строкам и дельтам.
- Агрегаты **sum**, **avg**, **count** и т.д. с **.over(window_spec)** дают для каждой строки агрегат по кадру (по умолчанию — от начала партиции до текущей строки).
- **Кадр** задаётся **rowsBetween** (по номерам строк) или **rangeBetween** (по значениям orderBy); по умолчанию — unbounded preceding to current row.

В [§7.3](chapter-07-03.md) мы разберём соединения: join(other, on, how), типы inner, left, right, full и по нескольким столбцам.
