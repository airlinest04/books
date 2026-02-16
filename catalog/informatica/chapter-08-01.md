# §8.1 Aggregator

В [§7.4](chapter-07-04.md) мы сравнили Lookup и Joiner. **Aggregator** — активная трансформация для агрегации по группам: SUM, AVG, COUNT, MIN, MAX и др. В отличие от Expression, работающего построчно, Aggregator вычисляет значения по группам строк. В этом разделе разберём Group By порты, агрегатные функции, sorted input и фильтрацию записей. Rank и Distinct — в [§8.2](chapter-08-02.md). См. [Глоссарий](glossary.md).

---

## 8.1.1. Назначение и принцип работы

**Aggregator** — активная трансформация; выполняет агрегатные расчёты по группам. Integration Service читает данные, сохраняет их в aggregate cache и вычисляет агрегаты для каждой группы. Источник: [Aggregator Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/aggregator-transformation/aggregator-transformation-overview.html).

**Expression vs Aggregator:** Expression — расчёт на уровне одной строки; Aggregator — на уровне группы строк (аналог SQL GROUP BY с агрегатными функциями).

**Условная фильтрация:** в выражениях Aggregator можно использовать условия для отбора строк в агрегации (больше гибкости, чем в стандартном SQL).

---

## 8.1.2. Group By порты

**Group By** — порты, по которым выполняется группировка. Для каждой уникальной комбинации значений Group By портов создаётся одна выходная строка с результатами агрегации. Источник: [Aggregator Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/aggregator-transformation/aggregator-transformation-overview.html).

**Без Group By:** одна строка на весь вход (агрегация по всем данным).

**С Group By:** одна строка на группу. Порядок Group By портов влияет на группировку; при sorted input порядок должен совпадать с Sort Key в Sorter или Number of Sorted Ports в Source Qualifier. См. [§6.1.4](chapter-06-01.md), [§6.4](chapter-06-04.md).

**Пример:** Group By `dept_id` — сумма продаж по каждому отделу; Group By `dept_id`, `year` — по отделу и году.

---

## 8.1.3. Агрегатные функции

Агрегатные функции используются в выражениях Output и Variable портов Aggregator. Источник: [Aggregate Functions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/aggregator-transformation/aggregate-expressions/aggregate-functions.html).

| Функция | Описание |
|---------|----------|
| **SUM** | Сумма значений |
| **AVG** | Среднее |
| **COUNT** | Количество строк |
| **MIN** | Минимальное значение |
| **MAX** | Максимальное значение |
| **FIRST** | Первое значение в группе |
| **LAST** | Последнее значение в группе |
| **MEDIAN** | Медиана |
| **PERCENTILE** | Перцентиль |
| **STDDEV** | Стандартное отклонение |
| **VARIANCE** | Дисперсия |

Функции можно вкладывать друг в друга (nested aggregate functions).

**Пример:** `SUM( amount )`, `AVG( price )`, `COUNT( * )`, `MAX( order_date )`.

---

## 8.1.4. Фильтрация записей в агрегации

В выражении агрегатной функции можно задать условие отбора строк. Синтаксис: `AGG_FUNC( port, condition )` — агрегация только по строкам, где condition TRUE.

**Пример:** `SUM( amount, status = 'COMPLETED' )` — сумма только по завершённым заказам.

Это даёт возможность, аналогичную `SUM(CASE WHEN ... THEN ... END)` в SQL, но через встроенный синтаксис Transformation Language.

---

## 8.1.5. Sorted input

При **sorted input** данные должны быть отсортированы по Group By портам в том же порядке. Integration Service выполняет агрегацию без полной загрузки группы в память — потоковая обработка. Источник: [§6.4.2](chapter-06-04.md).

**Настройка:**
1. Sorter (или Source Qualifier с Number of Sorted Ports) — сортировка по ключу группировки.
2. В Aggregator — включить Sorted Input и указать sort origin ports.
3. Порядок Group By = порядок Sort Key.

При несоблюдении порядка — неверный результат или ошибка.

---

## 8.1.6. Порты Aggregator

Aggregator поддерживает Input, Output, Variable порты. Порядок вычисления: Input → Variable → Output (см. [§5.4.2](chapter-05-04.md)).

- **Input** — данные из потока.
- **Variable** — промежуточные расчёты; могут использоваться в других Variable и Output.
- **Output** — результат; Group By порты передаются как есть; агрегатные порты содержат выражения с SUM, AVG и т.д.

Group By порты могут быть Input или Input/Output; они проходят в выход без изменения (по одному значению на группу).

---

## 8.1.7. Incremental Aggregation

**Incremental Aggregation** — опция Session: Integration Service использует исторический кеш и доагрегирует только новые данные. Подходит для инкрементальной загрузки, когда добавляются новые строки к уже обработанным. Источник: [Aggregator Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/aggregator-transformation/aggregator-transformation-overview.html).

---

## 8.1.8. Типичные ошибки

- **Неверный порядок Group By при sorted input:** порядок Group By должен совпадать с Sort Key; иначе неверная агрегация.
- **Агрегатная функция в не-Aggregator:** SUM, AVG и т.д. допустимы только в Aggregator.
- **Variable ссылается на Output:** Variable может ссылаться только на Input и Variable.
- **Два Aggregator в один вход:** нельзя; объединить через Union. См. [§4.3](chapter-04-03.md).

---

## Ключевое

- **Aggregator** — активная трансформация; агрегация по группам; аналог SQL GROUP BY.
- **Group By** — порты группировки; одна строка на группу; без Group By — одна строка на весь вход.
- **Функции:** SUM, AVG, COUNT, MIN, MAX, FIRST, LAST, MEDIAN, STDDEV, VARIANCE и др.
- **Фильтрация в агрегации:** `SUM( port, condition )` — только строки, где condition TRUE.
- **Sorted input** — порядок Group By = порядок Sort Key; потоковая агрегация.

В [§8.2](chapter-08-02.md) мы разберём Rank и Distinct: ранжирование и выбор уникальных строк.
