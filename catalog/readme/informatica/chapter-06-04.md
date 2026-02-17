# §6.4 Sorter и Union

В [§6.3](chapter-06-03.md) мы рассмотрели Filter и Router. **Sorter** и **Union** — активные трансформации для упорядочивания и объединения потоков. Sorter сортирует данные по ключу (одному или нескольким портам); Union объединяет данные из нескольких входных групп в один поток (аналог SQL UNION ALL). В этом разделе разберём Sort Key, настройки Sorter, структуру Union и требования к совместимости портов. Sorter часто используется перед Aggregator (sorted input) и Joiner; Union — для объединения результатов нескольких веток. См. [Глоссарий](glossary.md).

---

## 6.4.1. Sorter: назначение и Sort Key

**Sorter** — активная трансформация; упорядочивает строки по заданному ключу. Поддерживает relational и flat file источники. Источник: [Sorter Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/sorter-transformation/sorter-transformation-overview.html).

**Sort Key** — один или несколько портов, по которым выполняется сортировка. Порядок портов в Sort Key определяет приоритет: первый порт — первичная сортировка, второй — вторичная при равенстве первого и т.д. Источник: [Sorting Data](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/sorter-transformation/sorting-data.html).

**Настройки:**
- **Ascending / Descending** — направление сортировки для каждого порта ключа.
- **Case-sensitive** — чувствительность к регистру для строковых портов.
- **Distinct** — при включении Sorter удаляет дубликаты (аналог SELECT DISTINCT по ключу).

Sorter содержит только Input/Output порты; все данные проходят через трансформацию и сортируются.

---

## 6.4.2. Sorter: применение

**Aggregator с sorted input:** при настройке Aggregator на sorted input group by порты должны совпадать по порядку с Sort Key в Sorter. Это позволяет Integration Service выполнять агрегацию без полной загрузки группы в память. См. [§6.1.4](chapter-06-01.md), [§8.1](chapter-08-01.md).

**Joiner с sorted input:** Joiner может использовать merge join при отсортированных данных; порядок Sort Key в каждом входе Joiner должен совпадать. См. [§7.3](chapter-07-03.md).

**Альтернатива:** сортировка на уровне источника — Number of Sorted Ports в Source Qualifier (только для relational). См. [§6.1.4](chapter-06-01.md).

---

## 6.4.3. Union: назначение и структура

**Union** — активная трансформация с несколькими входными группами и одной выходной. Объединяет данные из нескольких потоков; аналог SQL UNION ALL — дубликаты не удаляются. Источник: [Union Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/union-transformation/union-transformation-overview.html).

**Структура:**
- **Input groups** — несколько групп; каждая подключается к одному источнику или трансформации.
- **Output group** — одна; порты совпадают с портами входных групп.

Integration Service обрабатывает входные группы параллельно; порядок строк в выходе зависит от порядка поступления блоков данных. Источник: [Working with Groups and Ports](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/union-transformation/working-with-groups-and-ports.html).

---

## 6.4.4. Union: требования к портам

Все входные группы и выходная группа должны иметь **совпадающие порты**: одинаковые datatype, precision и scale. Источник: [Rules and Guidelines for Union Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/union-transformation/union-transformation-overview/rules-and-guidelines-for-union-transformations.html).

| Требование | Описание |
|------------|----------|
| Совпадение типов | Одинаковый datatype, precision, scale для соответствующих портов во всех группах. |
| Порядок портов | Порты в группах должны соответствовать по позиции. |
| Несвязанные порты | Порты, не связанные с источником в какой-либо группе, получают NULL. |

Union не удаляет дубликаты. Для дедупликации — Router, Filter или Distinct (см. [§8.2](chapter-08-02.md)). Union не генерирует транзакции.

---

## 6.4.5. Union vs Joiner

| Критерий | Union | Joiner |
|----------|-------|--------|
| Назначение | Объединение потоков с одинаковой структурой | Соединение по условию (join) |
| Структура | Одинаковые колонки во всех входах | Разные структуры; результат — комбинация |
| Аналог SQL | UNION ALL | INNER/LEFT/RIGHT/FULL JOIN |
| Количество входов | Два и более | Два (master, detail) |

Union — для конкатенации строк (например, заказы из разных регионов в одну таблицу). Joiner — для обогащения данными из связанной таблицы (например, заказы + клиенты). См. [§7.3](chapter-07-03.md), [§7.4](chapter-07-04.md).

---

## 6.4.6. Типичные ошибки

- **Union с несовместимыми портами:** datatype, precision, scale должны совпадать; при несовпадении — Expression для приведения типов перед Union.
- **Неверный порядок Sort Key для Aggregator/Joiner:** group by порты Aggregator и Sort Key Sorter должны совпадать по порядку; иначе ошибка или неверный результат.
- **Ожидать дедупликацию от Union:** Union не удаляет дубликаты; при необходимости — Distinct или Filter после Union.
- **Два активных в один вход без Union:** два Aggregator или два Filter нельзя подключать к одному Expression; объединить через Union. См. [§4.3](chapter-04-03.md).

---

## Ключевое

- **Sorter** — активная трансформация; Sort Key (один или несколько портов); Ascending/Descending; опция Distinct.
- **Sorter перед Aggregator/Joiner** — при sorted input порядок Sort Key должен совпадать с group by или join key.
- **Union** — несколько input groups, один output; аналог UNION ALL; дубликаты не удаляются.
- **Union:** порты во всех группах — одинаковые datatype, precision, scale; несвязанные порты — NULL.
- **Union vs Joiner:** Union — конкатенация потоков; Joiner — соединение по условию.

В [§7.1](chapter-07-01.md) мы разберём Lookup: назначение, типы (Cached/Uncached, Connected/Unconnected) и обогащение данными из справочника.
