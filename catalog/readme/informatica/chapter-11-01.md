# §11.1 Партиционирование сессий

В [§10.4](chapter-10-04.md) мы рассмотрели обработку сбоев. **Партиционирование** (Partitioning) — механизм PowerCenter для параллельной обработки данных: Integration Service делит поток на несколько партиций, каждая выполняется отдельно. В этом разделе — назначение партиционирования, типы (Pass-through, Round-robin, Hash, Key Range, Database), partition points, Dynamic Partitioning и примеры применения. Подробнее Lookup cache — в [§11.2](chapter-11-02.md). См. [Глоссарий](glossary.md).

---

## 11.1.1. Назначение партиционирования

**Партиционирование** позволяет Integration Service распределять данные между несколькими партициями и обрабатывать их параллельно. Это повышает производительность при больших объёмах данных и нескольких ядрах CPU. Источник: [Partition Types Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview.html).

Партиционирование — опциональная функция PowerCenter (Partitioning option); без неё сессия выполняется в одной партиции. Настройка — в Session → Mapping tab → Partitions view; типы партиций задаются на каждом **partition point** (точке разбиения в пайплайне). Источник: [Partition Types Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview.html).

---

## 11.1.2. Partition types: обзор

| Тип | Описание | Применение |
|-----|----------|------------|
| **Pass-through** | Данные передаются без перераспределения; каждая партиция сохраняет свои строки. | Источники разного размера; дополнительная стадия пайплайна без перераспределения. |
| **Round-robin** | Блоки данных распределяются по партициям циклически. | Балансировка нагрузки при источниках разного размера; группировка не требуется. |
| **Hash auto-keys** | Hash по портам группировки/сортировки (compound key). | Rank, Sorter, unsorted Aggregator — строки с одинаковым ключом в одну партицию. |
| **Hash user keys** | Hash по пользовательским портам (partition key). | Группировка по выбранному ключу; числовой ключ быстрее строкового. |
| **Key Range** | Распределение по диапазонам значений ключа. | Source/target партиционированы по key range; WHERE в Source Qualifier. |
| **Database** | Использование нативного партиционирования БД (Oracle, DB2). | Oracle/DB2 multi-node tablespace; DB2 targets. |

Источник: [Partition Types Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview.html).

---

## 11.1.3. Round-robin

**Round-robin** — Integration Service распределяет блоки строк по партициям циклически. Каждая партиция обрабатывает примерно равный объём. Источник: [Round-Robin Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/round-robin-partition-type.html).

**Пример:** три flat file (80 000, 5 000, 15 000 строк). Без round-robin первая партиция обрабатывает 80%, вторая — 5%, третья — 15%. С round-robin на Filter — каждая партиция обрабатывает примерно треть данных. Источник: [Round-Robin Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/round-robin-partition-type.html).

**Когда использовать:** не нужна группировка; источники разного размера; балансировка нагрузки. Источник: [Round-Robin Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/round-robin-partition-type.html).

---

## 11.1.4. Hash auto-keys и Hash user keys

**Hash auto-keys:** Integration Service использует все порты группировки или сортировки как compound partition key. Источник: [Partition Types Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview.html).

**Hash user keys:** пользователь задаёт порты для partition key. Hash-функция распределяет строки с одинаковым ключом в одну партицию. Источник: [Hash User Keys Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/hash-user-keys-partition-type.html).

**Пример:** Sorter по ITEM_DESC; если описание длинное, можно задать hash user keys с ITEM_ID — числовой ключ обрабатывается быстрее. Источник: [Hash User Keys Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/hash-user-keys-partition-type.html).

**Когда использовать:** Rank, Sorter, unsorted Aggregator — строки одной группы должны быть в одной партиции. Источник: [Setting Partition Types in the Pipeline](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview/setting-partition-types-in-the-pipeline.html).

---

## 11.1.5. Key Range

**Key Range** — распределение по диапазонам значений partition key. Для каждой партиции задаётся Start Range и End Range. Источник: [Key Range Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/key-range-partition-type.html).

**Пример:** target партиционирован по ITEM_ID (0001–2999, 3000–5999, 6000–9999). Key range на target: Partition 1 — < 3000; Partition 2 — 3000–5999; Partition 3 — >= 6000. Integration Service направляет строки в соответствующую партицию. Источник: [Key Range Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/key-range-partition-type.html).

**Source Qualifier:** при key range Integration Service создаёт WHERE в SELECT (например, `CUSTOMER_ID < 135000` для одной партиции, `>= 135000` для другой). Источник: [Key Range Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/key-range-partition-type.html).

**Когда использовать:** source и target партиционированы по key range; оптимизация записи в партиционированные таблицы. Источник: [Key Range Partition Type](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/key-range-partition-type.html).

---

## 11.1.6. Partition points и настройка

**Partition point** — точка в пайплайне, где Integration Service перераспределяет данные. По умолчанию создаётся на каждом partition point; тип можно изменить. Источник: [Partition Types Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview.html).

Настройка: Session → Mapping tab → Partitions view; выбрать partition point → Edit → Partition type, Number of partitions. Для Hash user keys и Key Range — задать ключи и диапазоны. Источник: [Setting Partition Types in the Pipeline](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview/setting-partition-types-in-the-pipeline.html).

**Пример комбинирования:** Source Qualifier — pass-through, 3 партиции (три файла); Filter — round-robin для балансировки; Sorter — hash auto-keys; Target — key range для партиционированной таблицы. Источник: [Setting Partition Types in the Pipeline](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/partition-types/partition-types-overview/setting-partition-types-in-the-pipeline.html).

---

## 11.1.7. Dynamic Partitioning

**Dynamic Partitioning** — количество партиций определяется во время выполнения. Config Object tab → Partitioning Options. Источник: [Partitioning Options Settings](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-configuration-object/partitioning-options-settings.html).

| Настройка | Описание |
|-----------|----------|
| **Disabled** | Количество партиций задаётся на Mapping tab. |
| **Based on number of partitions** | Число из Number of Partitions или `$DynamicPartitionCount`. |
| **Based on number of nodes in grid** | По количеству узлов в grid; без grid — одна партиция. |
| **Based on source partitioning** | По максимуму партиций источника (database partition info). |
| **Based on number of CPUs** | По количеству CPU на узле; на grid — CPU × узлы. |

По умолчанию — Disabled. Источник: [Partitioning Options Settings](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-configuration-object/partitioning-options-settings.html).

---

## 11.1.8. Типичные ошибки

- **Hash без group key:** для Aggregator с group by — hash key должен совпадать с group by; иначе — некорректные агрегаты.
- **Key range без знания распределения:** неравномерные диапазоны могут привести к skew (одна партиция перегружена).
- **Слишком много партиций:** overhead на перераспределение; при малом объёме данных — деградация.
- **Partitioning option:** без лицензии Partitioning доступна только одна партиция.

---

## Ключевое

- **Партиционирование** — параллельная обработка; разделение потока на партиции; требует Partitioning option.
- **Типы:** Pass-through, Round-robin, Hash auto-keys, Hash user keys, Key Range, Database.
- **Round-robin** — балансировка при источниках разного размера; группировка не нужна.
- **Hash** — группировка по ключу; для Rank, Sorter, Aggregator.
- **Key Range** — диапазоны значений; для партиционированных source/target.
- **Dynamic Partitioning** — количество партиций на runtime; $DynamicPartitionCount, CPUs, nodes.

В [§11.2](chapter-11-02.md) мы разберём Lookup cache и buffer для оптимизации производительности.
