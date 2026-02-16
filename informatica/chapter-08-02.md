# §8.2 Rank и Distinct

В [§8.1](chapter-08-01.md) мы рассмотрели Aggregator. **Rank** — активная трансформация для выбора топ-N или bottom-N строк по заданному критерию; **Distinct** в Informatica реализуется через Select Distinct в Source Qualifier или опцию Distinct в Sorter. В этом разделе разберём Rank Index, группы в Rank, настройку top/bottom и способы получения уникальных строк. См. [Глоссарий](glossary.md).

---

## 8.2.1. Rank: назначение

**Rank** — активная трансформация; отбирает строки в верхнем или нижнем диапазоне по заданной мере. В отличие от MAX/MIN возвращает **набор** строк (например, топ-10), а не одну. Источник: [Rank Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/rank-transformation/rank-transformation-overview.html).

**Примеры:**
- Топ-10 продавцов по выручке в регионе.
- Три отдела с наименьшими расходами на зарплаты.
- Топ-5 товаров по количеству продаж.

Integration Service кеширует входные данные до выполнения расчёта ранга. Вход — только из одной трансформации.

---

## 8.2.2. Rank Index и Rank port

**Rank port** — порт (input/output), по которому выполняется ранжирование. Задаёт меру для сравнения (число или строка по session sort order). Источник: [Ports in a Rank Transformation](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/rank-transformation/ports-in-a-rank-transformation.html).

**Rank Index** — выходной порт с номером ранга (1, 2, 3, …). При совпадении значений несколько строк получают один и тот же ранг; следующий ранг пропускается (1, 1, 3, 4 — как в SQL RANK()). Источник: [Defining Groups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/rank-transformation/defining-groups.html).

**Top / Bottom:** настройка — отбор топ-N (наибольшие) или bottom-N (наименьшие) по Rank port.

---

## 8.2.3. Группы в Rank

Как и в Aggregator, в Rank можно задать **Group By** порты. Ранжирование выполняется отдельно внутри каждой группы. Источник: [Defining Groups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/rank-transformation/defining-groups.html).

**Пример:** топ-10 самых дорогих товаров **по каждому** производителю — Group By `manufacturer_id`, Rank port `price`, Top 10.

Без групп — один общий рейтинг по всему входу.

---

## 8.2.4. Rank: ограничения и рекомендации

- Вход — только из **одной** трансформации.
- Rank — блокирующая трансформация; кеширует данные до расчёта.
- Для строк: порядок определяется session sort order.
- Можно использовать Variable порты и неагрегатные выражения.

---

## 8.2.5. Distinct: способы реализации

В PowerCenter нет отдельной трансформации Distinct. Уникальные строки получают так: Источник: [Select Distinct](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/select-distinct.html), [§6.4](chapter-06-04.md).

| Способ | Где | Описание |
|--------|-----|----------|
| **Select Distinct** | Source Qualifier | Добавляет SELECT DISTINCT в default query; дедупликация на стороне источника. |
| **Sorter Distinct** | Sorter | Опция Distinct при сортировке; удаляет дубликаты по Sort Key. |
| **Aggregator** | Aggregator | Group By по всем нужным полям без агрегатов — одна строка на группу. |

**Select Distinct:** Properties Source Qualifier → Select Distinct. При SQL-override настройка игнорируется. Источник: [Select Distinct](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/source-qualifier-transformation/select-distinct.html).

**Sorter Distinct:** при включённой опции Distinct Sorter оставляет по одной строке на каждую уникальную комбинацию Sort Key. См. [§6.4.1](chapter-06-04.md).

---

## 8.2.6. Выбор способа дедупликации

| Сценарий | Рекомендация |
|----------|--------------|
| Дедупликация на этапе извлечения | Select Distinct в Source Qualifier |
| Дедупликация по ключу после преобразований | Sorter с Distinct |
| Дедупликация с агрегацией | Aggregator с Group By |
| Топ-N по группе | Rank с Group By |

---

## 8.2.7. Типичные ошибки

- **Rank с входом из нескольких трансформаций:** Rank принимает данные только из одной.
- **Select Distinct при SQL-override:** Select Distinct не применяется; при необходимости добавить DISTINCT в SQL-override.
- **Sorter Distinct по части полей:** уникальность только по Sort Key; при других дубликатах — Aggregator или расширить Sort Key.

---

## Ключевое

- **Rank** — выбор топ-N или bottom-N по Rank port; Rank Index; группы через Group By.
- **Distinct в PowerCenter** — Select Distinct (Source Qualifier), Sorter Distinct или Aggregator с Group By.
- **Select Distinct** — в default query; при SQL-override не действует.
- **Sorter Distinct** — одна строка на уникальную комбинацию Sort Key.

В [§8.3](chapter-08-03.md) мы разберём Normalizer и другие трансформации: разворот данных, Stored Procedure и прочие типы.
