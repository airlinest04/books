# §11.2 Lookup и джойн с эталоном

В [§11.1](chapter-11-01.md) мы разобрали обогащение транзакций эталонными данными. В этом разделе рассмотрим **реализацию** обогащения в ETL: Lookup в Informatica и аналогах, Join в SQL, факторы производительности. Подробнее Lookup и Joiner — в книге «Informatica»; здесь — приложение к эталонным данным. В [§11.3](chapter-11-03.md) разберём дедупликацию в пайплайне.

---

## 11.2.1. Lookup в Informatica

**Lookup** — трансформация для обогащения из справочника: по ключу (customer_id, product_code) ищется строка в таблице эталона; возвращаются атрибуты (name, region, category). См. книга «Informatica», [§7.1](../informatica/chapter-07-01.md).

Для эталонов Lookup использует: dim_customer, core.customer, таблицу маппинга или маппинг + эталон. Условие: `customer_id = IN_CUSTOMER_ID` (или `source_id = IN_SOURCE_ID` при двухшаговом обогащении: сначала маппинг → golden_id, затем эталон → атрибуты). См. [§11.1](chapter-11-01.md).

Режимы: **Cached** — справочник загружается в память; эффективно при малых и средних эталонах. **Uncached** — запрос к БД на каждую строку; при больших объёмах транзакций — нагрузка. См. книга «Informatica», [§7.2](../informatica/chapter-07-02.md), [§7.4](../informatica/chapter-07-04.md).

---

## 11.2.2. Joiner и Source Qualifier join

**Joiner** — соединение двух потоков (транзакции и эталон) по условию равенства. При больших эталонах, не помещающихся в кеш Lookup, Joiner может быть эффективнее: не требует полной загрузки справочника в память. См. книга «Informatica», [§7.4](../informatica/chapter-07-04.md).

**Source Qualifier join** — когда транзакции и эталон в **одной БД**, join можно выполнить в SQL на стороне СУБД (в Source Qualifier или через SQL override). Это обычно быстрее, чем Lookup или Joiner в сессии, т.к. СУБД оптимизирует соединение. См. книга «Informatica», [§7.4](../informatica/chapter-07-04.md).

---

## 11.2.3. Аналоги Lookup в других инструментах

**dbt:** обогащение через JOIN в SQL-модели: `staging_sales` JOIN `dim_customer` ON ... JOIN `dim_product` ON ...; результат — обогащённая таблица или view. Join выполняется в СУБД.

**Python (pandas):** `merge()` по ключу — аналог JOIN; `map()` с dict — аналог Lookup при малых справочниках в памяти. Для больших объёмов — PySpark `join()` или запросы к БД.

**Apache Spark:** `join()` на DataFrame; broadcast join при малом эталоне — эффективно. См. книга «Spark и PySpark» при наличии.

**Talend, SSIS, Datagram:** аналоги Lookup/Lookup Table и Join. Выбор по размеру справочника и расположению данных. См. документация соответствующих продуктов.

---

## 11.2.4. Join в SQL

В чистом SQL обогащение — это **JOIN**:

```sql
-- Обогащение транзакций клиентом и продуктом
SELECT 
    t.sale_id,
    t.date_id,
    t.customer_id,      -- уже golden_id или surrogate key
    c.name AS customer_name,
    c.region AS customer_region,
    t.product_id,
    p.name AS product_name,
    p.category AS product_category,
    t.amount
FROM staging_sales t
LEFT JOIN dim_customer c ON t.customer_id = c.customer_id
LEFT JOIN dim_product p ON t.product_id = p.product_id;
```

При двухшаговом обогащении (сначала маппинг):

```sql
-- Шаг 1: source_id → golden_id
SELECT t.*, m.golden_id AS customer_golden_id
FROM staging_sales t
LEFT JOIN mapping m ON m.source_system = 'CRM' AND m.source_id = t.crm_customer_id;

-- Шаг 2: golden_id → атрибуты (или запись surrogate key в факт)
```

LEFT JOIN сохраняет строки транзакций при отсутствии совпадения в эталоне (сироты); INNER JOIN отбросит их. См. [§11.1](chapter-11-01.md).

---

## 11.2.5. Производительность

**Размер эталона:**

- **Малый (тысячи строк):** Lookup Cached, dict/map в Python — эффективны; кеш в памяти.
- **Средний (сотни тысяч):** Lookup Cached при достаточной памяти; иначе Join в СУБД или Joiner.
- **Большой (миллионы):** Join в СУБД (Source Qualifier, dbt, чистый SQL); Joiner при гетерогенных источниках; избегать Uncached Lookup на каждую строку. См. книга «Informatica», [§7.4](../informatica/chapter-07-04.md).

**Индексы:** для JOIN по customer_id, product_id эталонная таблица должна иметь индекс по ключу; иначе полный скан. См. [§10.3](chapter-10-03.md).

**Порядок обогащения:** при нескольких эталонах (клиент, продукт, магазин) порядок Lookup/Join обычно не критичен при Join в SQL; в потоковых ETL — последовательность Lookup или один многопортовый Join.

---

## 11.2.6. Двухшаговое обогащение: маппинг + эталон

При транзакциях с **source_id** обогащение часто в два шага:

1. **Lookup/Join с маппингом:** source_system + source_id → golden_id.
2. **Lookup/Join с эталоном:** golden_id → атрибуты (или поиск surrogate key для записи в факт).

Можно объединить в один Join при наличии view или запроса, объединяющего маппинг и эталон. При раздельных шагах — два Lookup в цепочке или подзапросы в SQL. См. [§7.2](chapter-07-02.md), [§11.1](chapter-11-01.md).

---

## 11.2.7. Типичные ошибки

- **Uncached Lookup при большом потоке транзакций:** запрос к БД на каждую строку — крайне медленно; использовать Cached или Join.
- **Отсутствие индекса на ключе эталона:** JOIN без индекса — полный скан; производительность падает.
- **INNER JOIN при сиротах:** транзакции без совпадения в эталоне теряются; при необходимости сохранять — LEFT JOIN.
- **Путать ключи:** Lookup по source_id в эталон, где ключ — golden_id; нужен маппинг или эталон с маппингом.

---

## Ключевое

- **Lookup** — обогащение из справочника по ключу; в Informatica — Cached/Uncached, Connected/Unconnected; эталон как источник Lookup.
- **Joiner, Source Qualifier join** — альтернативы при больших эталонах или одной БД; Join в СУБД обычно быстрее.
- **SQL:** JOIN транзакций с dim_customer, dim_product; LEFT JOIN при сиротах.
- **Производительность:** размер эталона, кеш, индексы; малый эталон — Lookup Cached; большой — Join в СУБД.
- **Двухшаговое обогащение:** маппинг (source_id → golden_id) + эталон (golden_id → атрибуты).

В [§11.3](chapter-11-03.md) разберём дедупликацию в пайплайне: когда дедуплицировать, алгоритмы в ETL.
