# §10.3 Статистические функции

В [§10.2](chapter-10-02.md) мы разобрали регрессию и кластеризацию в MADlib. Помимо моделей машинного обучения, MADlib предоставляет **описательную статистику** и **корреляцию** — функции, которые строят сводки по таблицам и матрицы связей между числовыми столбцами. В этом разделе — **madlib.summary** (сводная статистика по столбцам), **madlib.correlation** и **madlib.covariance** (матрицы корреляции и ковариации Пирсона), а также **интеграция с SQL**: результаты в таблицах, использование в запросах и связь с обычными агрегатами Greenplum. По [MADlib — Summary](https://madlib.apache.org/docs/latest/group__grp__summary.html), [MADlib — Covariance and Correlation](https://madlib.apache.org/docs/latest/group__grp__correlation.html), [§10.1](chapter-10-01.md).

***

## 10.3.1. Summary: сводная статистика по таблице

Функция **madlib.summary**(source\_table, output\_table \[, target\_cols] \[, grouping\_cols] \[, get\_distinct] \[, get\_quartiles] \[, ntile\_array] \[, how\_many\_mfv] \[, get\_estimates] \[, n\_cols\_per\_run]) вычисляет **описательную статистику** по указанным столбцам таблицы и записывает результат в **output\_table**. Возвращает составной тип: имя выходной таблицы, число обработанных столбцов, длительность (секунды). См. документацию MADlib.

**Параметры:** source\_table — исходная таблица; output\_table — таблица для результата. **target\_cols** (необязательно) — список столбцов через запятую; если NULL, обрабатываются все столбцы. **grouping\_cols** — столбец(ы) для группировки: статистика считается отдельно по каждой группе (по каждому столбцу группировки независимо, не как комбинированный GROUP BY). **get\_distinct** (по умолчанию TRUE) — считать число уникальных значений; **get\_quartiles** (TRUE) — квартили (25%, медиана, 75%); **ntile\_array** — массив квантилей (например, ARRAY\[0.33, 0.66]); **how\_many\_mfv** — сколько наиболее частых значений выводить (по умолчанию 10). **get\_estimates** (TRUE по умолчанию) — использовать оценки для distinct (Flajolet–Martin) и для наиболее частых значений (быстрее, но приближённо); FALSE — точный подсчёт (медленнее). **n\_cols\_per\_run** — сколько столбцов обрабатывать за один проход по данным (влияет на память и число проходов). См. документацию MADlib.

**Столбцы выходной таблицы** (сокращённо): group\_by, group\_by\_value, target\_column, column\_number, data\_type, row\_count, distinct\_values, missing\_values, blank\_values, fraction\_missing, fraction\_blank; для числовых столбцов — positive\_values, negative\_values, zero\_values, **mean**, **variance**, **confidence\_interval** (95%), **min**, **max**, first\_quartile, **median**, third\_quartile, quantile\_array; most\_frequent\_values, mfv\_frequencies. Для нечисловых столбцов часть полей NULL; min/max для строк — длина кратчайшей/длиннейшей строки. См. документацию MADlib.

Пример (по документации MADlib):

```sql
SELECT * FROM madlib.summary('iris', 'iris_summary');
-- по группам:
SELECT * FROM madlib.summary('iris', 'iris_summary', 'sepal_length, sepal_width', 'class_name');
```

***

## 10.3.2. Корреляция и ковариация

**Корреляция Пирсона** — мера линейной связи двух переменных; значение от -1 (полная отрицательная связь) до 1 (полная положительная); 0 — отсутствие линейной связи. **Ковариация** — не нормированная мера совместной изменчивости. MADlib строит **матрицу** попарных корреляций (или ковариаций) по числовым столбцам таблицы. См. документацию MADlib.

**madlib.correlation**(source\_table, output\_table \[, target\_cols] \[, verbose] \[, grouping\_cols] \[, n\_groups\_per\_run]) — заполняет **output\_table** матрицей корреляций. **target\_cols** — список числовых столбцов через запятую (по умолчанию '\*' — все числовые). В выходной таблице: column\_position, variable (имя переменной), столбцы с коэффициентами для каждой пары; матрица нижнетреугольная, диагональ 1.0. Создаётся также таблица \*\_summary (method, source\_table, output\_table, column\_names, grouping\_cols, mean\_vector, total\_rows\_processed). **grouping\_cols** — группировка: отдельная матрица по каждой группе. NULL в данных при корреляции подставляются средним по столбцу (mean imputation); при необходимости предварительно отфильтруйте или обработайте пропуски во view. См. документацию MADlib.

**madlib.covariance**(source\_table, output\_table \[, target\_cols] \[, ...]) — тот же синтаксис, результат — матрица ковариаций. См. документацию MADlib.

Пример (по документации MADlib):

```sql
SELECT madlib.correlation('example_data', 'example_data_output', 'temperature, humidity');
SELECT * FROM example_data_output ORDER BY column_position;
```

***

## 10.3.3. Интеграция с SQL

Результаты MADlib — **таблицы** в той же базе: модели регрессии, центроиды k-means, сводки summary, матрицы correlation/covariance. С ними можно работать обычным SQL: JOIN, WHERE, представления, экспорт. Статистику summary или корреляцию удобно использовать для первичного разведочного анализа перед построением моделей ([§10.2](chapter-10-02.md)) или для отчётов. См. [§7.4](chapter-07-04.md).

**Стандартные агрегаты Greenplum** (AVG, SUM, COUNT, STDDEV, VARIANCE, MIN, MAX и др.) тоже выполняются распределённо и подходят для простых сводок по одному столбцу или по группам. MADlib summary даёт расширенный набор (квартили, медиана, оценки distinct, наиболее частые значения, доверительный интервал) за один вызов по многим столбцам; correlation/covariance — попарные матрицы по числовым столбцам, что стандартным SQL без MADlib реализовать громоздко. Выбор: если достаточно AVG/COUNT по группам — используйте обычный SQL; если нужны сводка по всей таблице, квартили, корреляционная матрица — MADlib. См. документацию MADlib и Greenplum.

***

## Ключевое

* **madlib.summary** — сводная статистика по столбцам таблицы: mean, variance, min, max, квартили, медиана, distinct, пропуски, наиболее частые значения; опционально группировка; результат в таблице.
* **madlib.correlation** и **madlib.covariance** — матрицы Пирсона и ковариаций по числовым столбцам; опционально группировка; NULL обрабатываются подстановкой среднего (mean imputation).
* Результаты MADlib хранятся в таблицах и используются в обычном SQL; для простых агрегатов достаточно стандартных функций Greenplum, для развёрнутых сводок и матриц связей — MADlib.

На этом мы завершаем главу 10 и часть V книги. Итоги по Greenplum: архитектура MPP ([гл. 1](chapter-01-01.md)–[2](chapter-02-01.md)), установка и управление ([гл. 3](chapter-03-01.md)–[4](chapter-04-01.md)), типы таблиц и распределение ([гл. 5](chapter-05-01.md)–[6](chapter-06-01.md)), SQL и оптимизация ([гл. 7](chapter-07-01.md)–[8](chapter-08-01.md)), внешние данные PXF ([гл. 9](chapter-09-01.md)) и in-database аналитика MADlib ([гл. 10](chapter-10-01.md)). Термины — в [глоссарии](glossary.md), источники — в [списке литературы](../../greenplum/sources.md).
