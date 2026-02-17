# §12.2 Маппинг: извлечение и преобразование

В [§12.1](chapter-12-01.md) мы сформулировали постановку задачи. **Маппинг извлечения и преобразования** — цепочка от Source Qualifier до Target: чтение из источника, фильтрация, обогащение справочниками, вычисления. В этом разделе — типовые паттерны с Source Qualifier, Expression, Lookup и Filter в контексте загрузки в Staging или DWH. Подробнее загрузка (Target, Update Strategy) — в [§12.3](chapter-12-03.md). См. [Глоссарий](glossary.md).

---

## 12.2.1. Общая структура маппинга

Типовая цепочка для загрузки в Staging или DWH:

```
Source → Source Qualifier → [Filter] → [Expression] → [Lookup] → ... → Target
```

**Source Qualifier** — обязателен; определяет, что читать из источника. **Filter** — отбор строк по условию; размещать ближе к источнику. **Expression** — вычисления, приведение типов, производные поля. **Lookup** — обогащение по справочнику (код → название, ID → атрибуты). См. [§6.1](chapter-06-01.md), [§6.2](chapter-06-02.md), [§6.3](chapter-06-03.md), [§7.1](chapter-07-01.md).

Порядок может меняться: Lookup до Filter (если фильтр по полю из справочника), Expression после Lookup (если вычисления используют результат Lookup).

---

## 12.2.2. Source Qualifier: извлечение

**Source Qualifier** связывает Source с потоком. По умолчанию — `SELECT * FROM table`. Для оптимизации:

| Настройка | Применение |
|-----------|------------|
| **Source Filter** | Условие WHERE; отбор по дате, статусу; уменьшает объём данных. |
| **SQL-override** | Произвольный SQL; join нескольких таблиц, подзапросы, специфичные функции БД. |
| **Number of Sorted Ports** | ORDER BY для Aggregator с sorted input или Joiner с merge join. |
| **User-Defined Join** | Join нескольких источников на стороне БД. |

**Инкрементальная загрузка:** Source Filter или SQL-override с условием по watermark (например, `MODIFIED_DATE >= $$LastRunDate`). Mapping parameter `$$LastRunDate` задаётся в Session или parameter file. См. [§6.1](chapter-06-01.md), [§12.1.2](chapter-12-01.md).

**Пример Source Filter:** `ORDERS.STATUS = 'ACTIVE' AND ORDERS.ORDER_DATE >= $$StartDate`

---

## 12.2.3. Filter: отбор строк

**Filter** — активная трансформация; пропускает только строки, где условие TRUE. Размещать после Source Qualifier (или после Lookup, если условие по полю из справочника). См. [§6.3](chapter-06-03.md).

**Примеры:**
- `AMOUNT > 0` — отбросить нулевые суммы.
- `STATUS <> 'CANCELLED'` — исключить отменённые.
- `CUSTOMER_ID IS NOT NULL` — отбросить строки без клиента.

**Альтернатива:** фильтрация в Source Qualifier (Source Filter или SQL-override) — выполняется на стороне БД; меньше данных передаётся в Integration Service. Использовать, когда условие выразимо в SQL.

---

## 12.2.4. Expression: вычисления и приведение типов

**Expression** — пассивная трансформация; вычисления по строке. См. [§6.2](chapter-06-02.md).

**Типовые задачи:**
- **Приведение типов:** `TO_CHAR(date_col)`, `TO_DECIMAL(str_col)`.
- **Производные поля:** `unit_price * quantity` → `total_amount`.
- **Условная логика:** `IIF(status='A', 1, 0)`, `DECODE(region, 'EU', 'Europe', 'US', 'America', 'Other')`.
- **Конкатенация:** `first_name || ' ' || last_name`.
- **Значения по умолчанию:** `IIF(ISNULL(field), 'N/A', field)`.

**Порядок портов:** Input → Variable → Output. Variable — для промежуточных расчётов.

---

## 12.2.5. Lookup: обогащение справочниками

**Lookup** — поиск в справочнике по ключу; возврат связанных полей. См. [§7.1](chapter-07-01.md), [§7.2](chapter-07-02.md).

**Типовые сценарии:**
- **Обогащение:** `customer_id` → Lookup по DIM_CUSTOMER → `customer_name`, `region`.
- **Код → описание:** `product_code` → Lookup по справочнику продуктов → `product_name`.
- **Проверка существования:** Lookup по target для SCD; возврат флага (новый/изменённый) для Update Strategy.

**Рекомендации:** кешировать малые справочники; для больших — persistent cache или join в Source Qualifier, если источник и справочник в одной БД. Индексы на колонках Lookup Condition. См. [§11.2](chapter-11-02.md).

---

## 12.2.6. Пример цепочки: загрузка заказов в Staging

**Сценарий:** таблица ORDERS (order_id, cust_id, product_id, quantity, unit_price, order_date, status) → Staging STG_ORDERS с обогащением по клиенту.

1. **Source** ORDERS → **Source Qualifier** с Source Filter `STATUS <> 'CANCELLED'`.
2. **Lookup** по DIM_CUSTOMER: `cust_id` → `customer_name`, `region`; Cached.
3. **Expression:** `total_amount = quantity * unit_price`; `order_date` pass-through.
4. **Target** STG_ORDERS: order_id, cust_id, customer_name, region, product_id, quantity, unit_price, total_amount, order_date.

При необходимости фильтрации по region — Filter после Lookup: `region = 'EU'`.

---

## 12.2.7. Типичные ошибки

- **Filter до Lookup при условии по справочнику:** Filter не имеет доступа к полям Lookup; разместить Lookup перед Filter.
- **Source Filter и SQL-override одновременно:** при SQL-override Source Filter игнорируется; условие включить в SQL.
- **Lookup без кеша для частых вызовов:** производительность падает; включить Cached.
- **Порядок портов в Expression:** Variable до Output; иначе ошибка.

---

## Ключевое

- **Цепочка:** Source Qualifier → Filter → Expression → Lookup → Target; порядок зависит от логики.
- **Source Qualifier:** Source Filter, SQL-override для фильтрации и инкрементальной загрузки.
- **Filter** — отбор по условию; ближе к источнику или после Lookup при условии по справочнику.
- **Expression** — вычисления, приведение типов, IIF, DECODE.
- **Lookup** — обогащение; кешировать; индексы на Lookup Condition.

В [§12.3](chapter-12-03.md) мы разберём маппинг загрузки: Target, режимы вставки/обновления, bulk load, Update Strategy.
