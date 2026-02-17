# §4.3 Связывание источников, трансформаций и целей

В [§4.1](chapter-04-01.md) и [§4.2](chapter-04-02.md) мы рассмотрели определение Mapping и его создание. В этом разделе — варианты связывания объектов: **линейный поток** (один источник → одна цепочка трансформаций → один приёмник), **разветвлённый поток** (один ко многим, многие к одному), **несколько источников** (Joiner, Union) и **несколько приёмников** (Router). Подробнее трансформации — в главах 5–8. См. [Глоссарий](glossary.md).

---

## 4.3.1. Линейный поток

**Линейный поток** — простейшая схема: Source → Source Qualifier → [трансформации] → Target. Данные идут в одном направлении, без ветвлений. Источник: [Options for Linking Ports](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/connecting-mapping-objects/options-for-linking-ports.html).

Пример: Source (таблица `orders`) → Source Qualifier → Expression (вычисление поля) → Filter (условие) → Target (таблица `staging_orders`). Каждый объект имеет один вход и один выход (или одну входную и одну выходную группу).

**One-to-one:** один выходной порт связывается с одним входным. Стандартный случай для линейной цепочки.

---

## 4.3.2. One-to-many: один выход ко многим входам

**One-to-many** — один порт (или выходная группа) связывается с **несколькими** входами трансформаций или targets. Одинаковые данные используются в разных ветках. Источник: [Linking One to Many](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/connecting-mapping-objects/options-for-linking-ports/linking-one-to-many.html).

Пример: выход Source Qualifier (поле `salary`) связывается с Expression (расчёт месячной зарплаты) и с Aggregator (средняя зарплата по отделу). Один поток данных разветвляется на несколько потребителей.

**Ограничение:** к одному входу **нижестоящей** трансформации можно подключать **только пассивные** трансформации. Несколько активных трансформаций в один вход — недопустимо (Designer выдаст ошибку). Пассивные (Expression, Filter и др.) не меняют количество строк; активные (Aggregator, Router) — меняют. См. [§5.2](chapter-05-02.md).

---

## 4.3.3. Many-to-one: объединение потоков

**Many-to-one** — несколько трансформаций (или выходных групп) подают данные в **одну** трансформацию или target. Источник: [Linking Many to One](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/connecting-mapping-objects/options-for-linking-ports/linking-many-to-one.html).

**Правило:** в один вход можно подключать **любое число пассивных** трансформаций. **Нельзя** подключать более одной **активной** трансформации к одному входу. Причина: активные трансформации меняют количество строк; объединение двух активных потоков в один вход создаёт неоднозначность.

**Исключения — специализированные трансформации:**
- **Union** — имеет несколько входных групп; предназначена для объединения потоков (UNION ALL). Каждая входная группа получает данные от одного источника или трансформации.
- **Joiner** — имеет два входа (master и detail); объединяет данные из двух источников по условию join.

Union и Joiner спроектированы для many-to-one; обычные трансформации (Expression, Aggregator) — нет. См. [§6.4](chapter-06-04.md), [§7.3](chapter-07-03.md).

---

## 4.3.4. Несколько источников: Joiner

**Joiner** — объединяет данные из **двух** источников по условию (INNER, LEFT, RIGHT, FULL join). Оба источника подключаются к Joiner: один — как master, другой — как detail. Источник: [§7.3](chapter-07-03.md).

Схема: Source1 → SQ1 ─┐
                      ├→ Joiner → [трансформации] → Target
       Source2 → SQ2 ─┘

Каждый Source имеет свой Source Qualifier. Joiner требует отсортированных или совместимых данных; при необходимости перед Joiner добавляют Sorter. Подробнее — в [§7.3](chapter-07-03.md), [§7.4](chapter-07-04.md).

---

## 4.3.5. Несколько источников: Union

**Union** — объединяет данные из **нескольких** потоков с совместимой структурой (одинаковые колонки по типам и порядку). Аналог SQL UNION ALL. Источник: [Using a Union Transformation in a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/union-transformation/using-a-union-transformation-in-a-mapping.html).

Схема: Source1 → SQ1 ─┐
       Source2 → SQ2 ─┼→ Union → [трансформации] → Target
       Source3 → SQ3 ─┘

Union имеет несколько входных групп; каждая группа подключается к одному источнику или трансформации. Порты во всех группах должны быть совместимы; несвязанные порты получают NULL. Union — пассивная (неблокирующая) трансформация.

---

## 4.3.6. Несколько приёмников: Router

**Router** — направляет строки в **разные выходные группы** по условиям. Каждая группа может вести в отдельный Target или в разные трансформации. Источник: [Connecting Router Transformations in a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/router-transformation/connecting-router-transformations-in-a-mapping.html).

Схема: Source → SQ → Expression → Router ─┬→ Target1 (группа «Активные»)
                                          ├→ Target2 (группа «Неактивные»)
                                          └→ Target3 (группа Default)

Router — активная трансформация; каждая строка попадает в одну группу (или в Default, если ни одно условие не выполнено). Выходы Router связываются с разными targets или с разными ветками преобразований. Подробнее — в [§6.3](chapter-06-03.md).

---

## 4.3.7. Комбинированные потоки

Возможны комбинации:

- **Несколько источников → преобразования → один приёмник:** Joiner или Union объединяют данные; далее Expression, Filter, Aggregator и т.д.; один Target.
- **Один источник → преобразования → несколько приёмников:** Router или разветвление one-to-many на несколько targets.
- **Несколько источников → преобразования → несколько приёмников:** Joiner/Union → Expression/Filter → Router → несколько targets.

При проектировании соблюдать правила: не подключать несколько активных трансформаций к одному входу; использовать Union/Joiner для объединения; Router — для разветвления.

---

## 4.3.8. Input group и Output group

Трансформации могут иметь **несколько входных или выходных групп**. Источник: [Rules and Guidelines for Connecting Mapping Objects](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/connecting-mapping-objects/rules-and-guidelines-for-connecting-mapping-objects.html).

- **Input group:** минимум один порт группы должен быть связан с вышестоящим объектом.
- **Output group:** минимум один порт группы должен быть связан с нижестоящим объектом или target.

Router имеет несколько output groups; Union — несколько input groups; Joiner — два input groups (master, detail). При связывании учитывать, к какой группе относится порт.

---

## 4.3.9. Joiner: два output groups при sorted data

Специальный случай: у одной трансформации **два output groups**, оба подключаются к Joiner, настроенному для sorted data. Данные из обеих групп должны быть отсортированы; Joiner выполняет merge join. Редко используется; обычно Joiner получает данные от двух разных Source Qualifier. Источник: [Rules and Guidelines for Connecting Mapping Objects](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/connecting-mapping-objects/rules-and-guidelines-for-connecting-mapping-objects.html).

---

## 4.3.10. Типичные ошибки

- **Два Aggregator в один Expression:** нельзя — Aggregator активная; два активных в один вход запрещены. Использовать Union для объединения результатов агрегации.
- **Router без связей с target:** каждая output group Router должна быть связана с target или трансформацией; иначе данные теряются.
- **Union с несовместимой структурой:** порты во всех input groups должны совпадать по типам и порядку; иначе ошибка валидации.
- **Joiner без master/detail:** оба входа Joiner должны быть подключены; иначе Invalid.
- **Забыть Sorter перед Joiner:** при определённых типах join Joiner требует отсортированных данных; добавить Sorter. См. [§7.3](chapter-07-03.md).

---

## Ключевое

- **Линейный поток:** Source → SQ → [Transform] → Target; one-to-one связи.
- **One-to-many:** один выход — ко многим входам; допустимо для пассивных потребителей; несколько активных в один вход — нельзя.
- **Many-to-one:** много выходов — в один вход; только пассивные, либо Union/Joiner (специализированы).
- **Несколько источников:** Joiner (два источника, join) или Union (несколько, UNION ALL).
- **Несколько приёмников:** Router (группы по условиям) или one-to-many от пассивной трансформации.
- **Input/Output groups:** минимум один порт группы связан; учитывать группы при Joiner, Union, Router.

В [§4.4](chapter-04-04.md) мы разберём маппинг-визарды: Copy, Filter, Aggregation и другие шаблоны для быстрого создания типовых маппингов.
