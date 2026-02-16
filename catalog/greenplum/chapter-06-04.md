# §6.4 Партиционирование по диапазону и списку

В [§6.3](chapter-06-03.md) мы ввели партиционирование и partition pruning. Здесь — **синтаксис** партиций **RANGE** (по датам и числам), **LIST** (по региону, статусу и т.п.), **подпартиции** (многоуровневое разбиение) и **default-партиция**. Примеры приведены в современном синтаксисе Greenplum 7 (PARTITION BY + CREATE TABLE ... PARTITION OF). По [Tanzu Greenplum 7 — Partitioning Large Tables](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-ddl-ddl-partition.html).

---

## 6.4.1. RANGE по датам

Партиционирование **RANGE** по столбцу типа **date** или **timestamp** удобно для фактов с временной осью: данные разбиваются по месяцам, кварталам или годам; запросы с фильтром по периоду сканируют только нужные партиции. См. [§6.3](chapter-06-03.md).

Границы задаются **FROM ... TO**: нижняя граница **включена**, верхняя **исключена**. Партиция `FROM ('2024-01-01') TO ('2024-02-01')` содержит даты с 2024-01-01 по 2024-01-31 включительно; значение 2024-02-01 попадает в следующую партицию. Соседние партиции могут «стыковаться» по границе (верхняя граница одной = нижняя другой).

Пример: таблица по месяцам.

```sql
CREATE TABLE fact_sales (
    sale_id     bigint,
    sale_date   date NOT NULL,
    customer_id int,
    amount      numeric(12,2)
) DISTRIBUTED BY (sale_id)
  PARTITION BY RANGE (sale_date);

CREATE TABLE fact_sales_2024_01 PARTITION OF fact_sales
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE fact_sales_2024_02 PARTITION OF fact_sales
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE fact_sales_2024_03 PARTITION OF fact_sales
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
-- далее по месяцам
```

Вставка в корневую таблицу `fact_sales` автоматически направляется в партицию по значению `sale_date`. Строки, не попадающие ни в одну партицию, приводят к ошибке вставки; чтобы их принять, добавляют **default-партицию** (см. ниже). Новые месяцы добавляют созданием новых партиций (**CREATE TABLE ... PARTITION OF** или **ALTER TABLE ... ATTACH PARTITION** в зависимости от синтаксиса версии).

---

## 6.4.2. RANGE по числам

**RANGE** можно задать по столбцу **целочисленного** или другого сравниваемого типа: год, идентификатор диапазона, номер периода. Формат тот же — **FROM значение TO значение** (нижняя включительно, верхняя исключительно).

Пример: по году.

```sql
CREATE TABLE events_by_year (
    id    bigint,
    year  int NOT NULL,
    data  text
) DISTRIBUTED BY (id)
  PARTITION BY RANGE (year);

CREATE TABLE events_by_year_2022 PARTITION OF events_by_year FOR VALUES FROM (2022) TO (2023);
CREATE TABLE events_by_year_2023 PARTITION OF events_by_year FOR VALUES FROM (2023) TO (2024);
CREATE TABLE events_by_year_2024 PARTITION OF events_by_year FOR VALUES FROM (2024) TO (2025);
```

В Greenplum 7 современный синтаксис поддерживает **несколько столбцов** в ключе RANGE (например, (year, month)); детали см. в документации CREATE TABLE.

---

## 6.4.3. LIST: регион, статус и default-партиция

При **LIST**-партиционировании каждая партиция задаётся **списком значений** ключа: **FOR VALUES IN (значение1, значение2, ...)**. Подходит для дискретных признаков: регион, статус заказа, тип продукта. См. [Глоссарий](glossary.md) (PARTITION BY).

Пример: по региону.

```sql
CREATE TABLE orders_by_region (
    order_id   bigint,
    region     text NOT NULL,
    amount     numeric(12,2)
) DISTRIBUTED BY (order_id)
  PARTITION BY LIST (region);

CREATE TABLE orders_usa PARTITION OF orders_by_region FOR VALUES IN ('usa');
CREATE TABLE orders_europe PARTITION OF orders_by_region FOR VALUES IN ('europe');
CREATE TABLE orders_asia PARTITION OF orders_by_region FOR VALUES IN ('asia');
CREATE TABLE orders_other PARTITION OF orders_by_region DEFAULT;
```

Партиция **DEFAULT** принимает все строки, **не попавшие** ни в одну из партиций с явным списком. Её полезно иметь, чтобы вставка не падала при появлении нового значения ключа (например, нового региона); затем данные из default можно перенести в новую партицию или оставить там по политике. Учтите: оптимизатор при запросе **всегда сканирует default-партицию**, если она есть (он не может доказать, что подходящих строк там нет), поэтому большая default-партиция может замедлять запросы; старайтесь держать в ней только «остаток» или периодически выделять новые партиции. См. документацию Greenplum.

---

## 6.4.4. Подпартиции (многоуровневое партиционирование)

Партиция может быть объявлена **партиционированной** снова — получается **подпартиционирование**. Например: первый уровень — по году (RANGE), второй — по региону (LIST). Листовыми (хранящими данные) будут только партиции самого нижнего уровня.

Пример: год → регион.

```sql
CREATE TABLE sales (
    trans_id bigint,
    year     int NOT NULL,
    region   text NOT NULL,
    amount   numeric(12,2)
) DISTRIBUTED BY (trans_id)
  PARTITION BY RANGE (year);

CREATE TABLE sales_2023 PARTITION OF sales FOR VALUES FROM (2023) TO (2024)
  PARTITION BY LIST (region);
CREATE TABLE sales_2024 PARTITION OF sales FOR VALUES FROM (2024) TO (2025)
  PARTITION BY LIST (region);

CREATE TABLE sales_2023_usa   PARTITION OF sales_2023 FOR VALUES IN ('usa');
CREATE TABLE sales_2023_eu    PARTITION OF sales_2023 FOR VALUES IN ('europe');
CREATE TABLE sales_2023_asia  PARTITION OF sales_2023 FOR VALUES IN ('asia');
CREATE TABLE sales_2024_usa   PARTITION OF sales_2024 FOR VALUES IN ('usa');
CREATE TABLE sales_2024_eu    PARTITION OF sales_2024 FOR VALUES IN ('europe');
CREATE TABLE sales_2024_asia  PARTITION OF sales_2024 FOR VALUES IN ('asia');
```

Запрос `WHERE year = 2024 AND region = 'usa'` при отсечении затрагивает только партицию `sales_2024_usa`. Количество листовых партиций растёт как произведение уровней (годы × регионы); не создавайте избыточную глубину и число партиций — это усложняет планирование и обслуживание. См. [§6.3](chapter-06-03.md).

---

## 6.4.5. Добавление и отсоединение партиций

**Добавление партиции:** создаётся новая таблица с границами и присоединяется к родителю: **CREATE TABLE ... PARTITION OF родитель FOR VALUES ...** (или в части версий **ALTER TABLE родитель ATTACH PARTITION**). Для RANGE по датам типична автоматизация: скрипт раз в месяц создаёт партицию на следующий период.

**Удаление данных за период (RANGE):** быстрый вариант — **DROP** партиции (данные удаляются) или **ALTER TABLE ... DETACH PARTITION**: партиция отсоединяется и становится самостоятельной таблицей, данные сохраняются. После DETACH с ней можно работать отдельно (архивировать, удалить позже). Синтаксис DETACH/ATTACH уточняйте в справочнике ALTER TABLE вашей версии.

Обычную непартиционированную таблицу нельзя «сделать» партиционированной одной командой: нужно создать новую партиционированную таблицу, перенести в неё данные, удалить старую таблицу и переименовать новую. См. документацию Greenplum.

---

## 6.4.6. Типичные ошибки

- **Путать границы RANGE:** верхняя граница исключающая; значение, равное верхней границе партиции, попадает в **следующую** партицию.
- **Забывать default-партицию при LIST:** при появлении нового значения ключа вставка в корень завершится ошибкой; либо задайте DEFAULT, либо заранее создавайте партиции под все значения.
- **Делать слишком глубокую иерархию подпартиций:** число листовых партиций и файлов (на каждом сегменте) резко растёт; оценивайте произведение размеров по уровням до внедрения.

---

## Ключевое

- **RANGE:** границы **FROM значение TO значение** (нижняя включительно, верхняя исключительно); подходит для дат и чисел (месяц, год, период).
- **LIST:** **FOR VALUES IN ('a', 'b', ...)**; подходит для региона, статуса и т.п.; **DEFAULT** — партиция для всех остальных значений (оптимизатор всегда её сканирует).
- **Подпартиции:** партиция объявляется с собственным **PARTITION BY**; листовые партиции только на нижнем уровне; не раздувайте число уровней и партиций.

В [§7.1](chapter-07-01.md) мы перейдём к SQL в Greenplum: совместимость с PostgreSQL, что работает как в Postgres, и основные ограничения в распределённой среде.
