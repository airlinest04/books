# §5.2 Активные и пассивные трансформации

В [§5.1](chapter-05-01.md) мы ввели три оси классификации трансформаций. **Active** (активная) и **Passive** (пассивная) — классификация по влиянию на поток данных: меняет ли трансформация количество строк, границы транзакций или тип строк. Это определяет правила связывания в маппинге. В этом разделе мы разберём определение Active и Passive, примеры, ограничения на подключение к одному входу и исключение (Sequence Generator). См. [Глоссарий](glossary.md).

---

## 5.2.1. Активная трансформация (Active)

**Active transformation** (активная трансформация) может выполнять одно или несколько действий. Источник: [Active Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/active-transformations.html).

| Действие | Описание | Примеры |
|----------|----------|---------|
| **Изменение количества строк** | Добавляет или удаляет строки в потоке | Filter (удаляет по условию), Aggregator (сворачивает в группы), Router (разветвляет), Joiner (объединяет), Union (объединяет) |
| **Изменение границ транзакции** | Определяет commit или rollback | Transaction Control |
| **Изменение типа строки** | Помечает строку для insert/update/delete/reject | Update Strategy |

Все **multi-group** трансформации (несколько входных или выходных групп) — активные, так как могут менять количество строк. Примеры: Filter, Router, Aggregator, Joiner, Union, Sorter, Rank, Source Qualifier, Normalizer, Update Strategy.

---

## 5.2.2. Пассивная трансформация (Passive)

**Passive transformation** (пассивная трансформация) **не меняет** количество строк, сохраняет границы транзакций и тип строки. Источник: [Passive Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/passive-transformations.html).

Пассивная трансформация передаёт каждую входную строку на выход (возможно, с изменёнными значениями полей). Количество строк на входе равно количеству на выходе.

Примеры: Expression (вычисляет поля), Lookup в режиме pass-through (подставляет значения, не отфильтровывает), Sequence Generator (генерирует значения), Stored Procedure (при определённых настройках).

---

## 5.2.3. Сравнение

| Признак | Active | Passive |
|---------|--------|---------|
| Количество строк | Может измениться | Не меняется |
| Границы транзакций | Может измениться | Сохраняются |
| Тип строки (insert/update/delete) | Может измениться | Сохраняется |
| Примеры | Filter, Aggregator, Router, Joiner, Union, Sorter, Rank | Expression, Lookup (pass-through), Sequence Generator |

---

## 5.2.4. Правила связывания: ограничения для Active

Designer **не позволяет** подключать к одному входу нижестоящей трансформации (или к одной входной группе): Источник: [Active Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/active-transformations.html), [Rules and Guidelines for Connecting Mapping Objects](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/connecting-mapping-objects/rules-and-guidelines-for-connecting-mapping-objects.html).

- **Несколько активных трансформаций** — нельзя. Причина: Integration Service не может однозначно объединить строки от нескольких активных трансформаций (например, одна ветка помечает строку для delete, другая — для insert).
- **Активную и пассивную** — нельзя. Та же причина: неоднозначность при объединении.

**Разрешено:** любое число **пассивных** трансформаций к одному входу. Пассивные не меняют количество строк и тип; объединение не создаёт конфликтов.

---

## 5.2.5. Исключение: Sequence Generator

**Sequence Generator** — исключение. Designer разрешает подключать Sequence Generator и активную трансформацию к одному входу нижестоящей трансформации. Источник: [Active Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/active-transformations.html).

Причина: Sequence Generator **не получает** данные — он только генерирует уникальные числовые значения. Integration Service не объединяет строки от Sequence Generator с данными от активной трансформации в конфликтном смысле; Sequence Generator добавляет значения к существующим строкам.

---

## 5.2.6. Практические следствия

**Два Aggregator в один Expression — нельзя.** Оба активные; два активных в один вход запрещены. Решение: Union для объединения результатов агрегации, затем дальнейшая обработка. См. [§4.3](chapter-04-03.md).

**Два Expression в один Aggregator — можно.** Оба пассивные; несколько пассивных в один вход разрешено. (На практике чаще один Expression передаёт в Aggregator; два Expression могут быть при разных ветках одного источника.)

**Filter и Aggregator в один Target — нельзя.** Filter — активная, Aggregator — активная; две активные в один target недопустимы. Каждая активная ветка должна вести в отдельный target или в Union.

---

## 5.2.7. Lookup: Active или Passive

**Lookup** может быть Active или Passive в зависимости от настроек. Источник: [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

- **Passive:** при пропуске всех строк (no rows dropped); Lookup только подставляет значения, количество строк не меняется.
- **Active:** при отсечении строк (rows dropped), когда Lookup не находит совпадение и строка не передаётся дальше.

Настройка **Lookup Policy** (например, Lookup Condition on Dynamic Lookup Cache) влияет на Active/Passive. Подробнее — в [§7.1](chapter-07-01.md).

---

## 5.2.8. Типичные ошибки

- **Два Filter в один Aggregator:** оба активные — ошибка валидации. Объединить через Union или изменить логику.
- **Считать Expression активной:** Expression — пассивная; не меняет количество строк.
- **Забыть про Router:** Router — активная; несколько выходных групп; каждая группа — отдельная ветка.
- **Игнорировать Update Strategy:** Update Strategy — активная; меняет тип строки; учитывать при объединении веток.

---

## Ключевое

- **Active** — меняет количество строк, границы транзакций или тип строки. Примеры: Filter, Aggregator, Router, Joiner, Union, Sorter, Rank, Update Strategy.
- **Passive** — не меняет количество строк, границы и тип. Примеры: Expression, Lookup (pass-through), Sequence Generator.
- **Правило:** нельзя подключать несколько активных или активную и пассивную к одному входу; можно — любое число пассивных.
- **Исключение:** Sequence Generator можно подключать вместе с активной к одному входу (не получает данные).
- **Lookup** — Active или Passive в зависимости от настроек (rows dropped или нет).

В [§5.3](chapter-05-03.md) мы разберём Connected и Unconnected трансформации: различие по способу подключения к потоку данных.
