# §12.3 Дедупликация и золотая запись

В [§12.2](chapter-12-02.md) мы загрузили данные из CRM и ERP в Staging. В этом разделе реализуем **дедупликацию и золотую запись**: обнаружение групп дубликатов, слияние по правилам из [§12.1.5](chapter-12-01.md), загрузка в Core и маппинг. Распространение в витрины — в [§12.4](chapter-12-04.md). См. [Глоссарий](glossary.md).

---

## 12.3.1. Ключ группировки

По [§12.1.4](chapter-12-01.md): дедупликация по ИНН (если заполнен) или по name + phone; при отсутствии — запись остаётся в отдельной группе.

Ключ группировки (`group_key`):

- Если `inn` не пустой и длина 10 или 12 цифр — `group_key = inn`.
- Иначе, если `name` и `phone` не пустые — `group_key = name || '|' || phone`.
- Иначе — уникальный ключ: `source_system || '|' || source_id` (запись без дубликатов).

См. [§5.1](chapter-05-01.md).

---

## 12.3.2. Вычисление group_key и нумерация групп

```sql
-- PostgreSQL; staging.customer заполнена за текущий прогон
WITH with_key AS (
    SELECT *,
        COALESCE(
            CASE WHEN inn IS NOT NULL AND LENGTH(TRIM(inn)) IN (10, 12) 
                 THEN TRIM(inn) ELSE NULL END,
            CASE WHEN TRIM(COALESCE(name,'')) <> '' AND TRIM(COALESCE(phone,'')) <> ''
                 THEN TRIM(name) || '|' || TRIM(phone) ELSE NULL END,
            source_system || '|' || source_id
        ) AS group_key
    FROM staging.customer
    WHERE loaded_at::date = CURRENT_DATE
),
groups AS (
    SELECT group_key, ROW_NUMBER() OVER (ORDER BY MIN(source_system), MIN(source_id)) AS group_num
    FROM with_key
    GROUP BY group_key
)
SELECT w.*, g.group_num
FROM with_key w
JOIN groups g ON g.group_key = w.group_key;
```

`group_num` — стабильный идентификатор группы для последующего слияния. См. [§5.4](chapter-05-04.md).

---

## 12.3.3. Слияние по правилам приоритета

По [§12.1.5](chapter-12-01.md): name, inn, address — приоритет ERP; email, phone, registration_date — приоритет CRM. Используем `MAX(...) FILTER (WHERE source_system = ...)`:

```sql
WITH with_key AS (
    SELECT *,
        COALESCE(
            CASE WHEN inn IS NOT NULL AND LENGTH(TRIM(inn)) IN (10, 12) THEN TRIM(inn) ELSE NULL END,
            CASE WHEN TRIM(COALESCE(name,'')) <> '' AND TRIM(COALESCE(phone,'')) <> ''
                 THEN TRIM(name) || '|' || TRIM(phone) ELSE NULL END,
            source_system || '|' || source_id
        ) AS group_key
    FROM staging.customer
    WHERE loaded_at::date = CURRENT_DATE
),
merged AS (
    SELECT
        group_key,
        COALESCE(
            MAX(name) FILTER (WHERE source_system = 'ERP' AND TRIM(COALESCE(name,'')) <> ''),
            MAX(name) FILTER (WHERE source_system = 'CRM' AND TRIM(COALESCE(name,'')) <> '')
        ) AS name,
        MAX(inn) FILTER (WHERE source_system = 'ERP' AND TRIM(COALESCE(inn,'')) <> '') AS inn,
        MAX(email) FILTER (WHERE source_system = 'CRM' AND TRIM(COALESCE(email,'')) <> '') AS email,
        COALESCE(
            MAX(phone) FILTER (WHERE source_system = 'CRM' AND TRIM(COALESCE(phone,'')) <> ''),
            MAX(phone) FILTER (WHERE source_system = 'ERP' AND TRIM(COALESCE(phone,'')) <> '')
        ) AS phone,
        MAX(address) FILTER (WHERE source_system = 'ERP' AND TRIM(COALESCE(address,'')) <> '') AS address,
        MAX(registration_date) FILTER (WHERE source_system = 'CRM') AS registration_date,
        array_agg(DISTINCT (source_system, source_id)) AS source_records
    FROM with_key
    GROUP BY group_key
)
SELECT * FROM merged;
```

`source_records` — список (source_system, source_id) для маппинга. См. [§4.2](chapter-04-02.md), [§5.3](chapter-05-03.md).

---

## 12.3.4. Загрузка в Core и маппинг

Структура Core:

```sql
-- core.customer
CREATE TABLE core.customer (
    golden_id    BIGSERIAL PRIMARY KEY,
    name         VARCHAR(500) NOT NULL,
    inn          VARCHAR(12),
    email        VARCHAR(255),
    phone        VARCHAR(50),
    address      TEXT,
    registration_date DATE,
    updated_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- core.customer_mapping
CREATE TABLE core.customer_mapping (
    golden_id    BIGINT NOT NULL REFERENCES core.customer(golden_id),
    source_system VARCHAR(50) NOT NULL,
    source_id    VARCHAR(255) NOT NULL,
    PRIMARY KEY (source_system, source_id)
);
```

Идея вставки: (1) сформировать merged по group_key; (2) INSERT в core.customer и получить golden_id; (3) связать каждый (source_system, source_id) из группы с соответствующим golden_id и INSERT в core.customer_mapping. Для связи merged с golden_id удобно использовать временную таблицу и RETURNING — см. пошаговую реализацию ниже. См. [§7.2](chapter-07-02.md), [§8.2](chapter-08-02.md).

---

## 12.3.5. Пошаговая реализация (рекомендуется)

Разбить на два шага: сначала слияние и вставка в Core, затем маппинг по `group_key`.

**Шаг 1.** Временная таблица с объединёнными записями и `group_key`:

```sql
CREATE TEMP TABLE tmp_merged AS
WITH with_key AS (
    SELECT *, COALESCE(
        CASE WHEN inn IS NOT NULL AND LENGTH(TRIM(inn)) IN (10, 12) THEN TRIM(inn) ELSE NULL END,
        CASE WHEN TRIM(COALESCE(name,'')) <> '' AND TRIM(COALESCE(phone,'')) <> ''
             THEN TRIM(name) || '|' || TRIM(phone) ELSE NULL END,
        source_system || '|' || source_id
    ) AS group_key
    FROM staging.customer WHERE loaded_at::date = CURRENT_DATE
)
SELECT group_key,
    COALESCE(MAX(name) FILTER (WHERE source_system = 'ERP' AND TRIM(COALESCE(name,'')) <> ''),
             MAX(name) FILTER (WHERE source_system = 'CRM' AND TRIM(COALESCE(name,'')) <> '')) AS name,
    MAX(inn) FILTER (WHERE source_system = 'ERP') AS inn,
    MAX(email) FILTER (WHERE source_system = 'CRM') AS email,
    COALESCE(MAX(phone) FILTER (WHERE source_system = 'CRM'), MAX(phone) FILTER (WHERE source_system = 'ERP')) AS phone,
    MAX(address) FILTER (WHERE source_system = 'ERP') AS address,
    MAX(registration_date) FILTER (WHERE source_system = 'CRM') AS registration_date
FROM with_key
GROUP BY group_key;
```

**Шаг 2.** Вставка в Core и формирование `tmp_golden` (group_key, golden_id). Порядок RETURNING совпадает с порядком SELECT; связь через `ROW_NUMBER`:

```sql
CREATE TEMP TABLE tmp_golden (golden_id BIGINT, group_key TEXT);

WITH inserted AS (
    INSERT INTO core.customer (name, inn, email, phone, address, registration_date)
    SELECT name, inn, email, phone, address, registration_date 
    FROM tmp_merged ORDER BY group_key
    RETURNING golden_id
),
num_inserted AS (
    SELECT golden_id, ROW_NUMBER() OVER () AS rn FROM inserted
),
num_merged AS (
    SELECT group_key, ROW_NUMBER() OVER (ORDER BY group_key) AS rn FROM tmp_merged
)
INSERT INTO tmp_golden (golden_id, group_key)
SELECT i.golden_id, m.group_key
FROM num_inserted i
JOIN num_merged m ON i.rn = m.rn;
```

**Шаг 3.** Маппинг: для каждой записи в Staging находим group_key и соответствующий golden_id:

```sql
WITH with_key AS (
    SELECT source_system, source_id,
        COALESCE(
            CASE WHEN inn IS NOT NULL AND LENGTH(TRIM(inn)) IN (10, 12) THEN TRIM(inn) ELSE NULL END,
            CASE WHEN TRIM(COALESCE(name,'')) <> '' AND TRIM(COALESCE(phone,'')) <> ''
                 THEN TRIM(name) || '|' || TRIM(phone) ELSE NULL END,
            source_system || '|' || source_id
        ) AS group_key
    FROM staging.customer WHERE loaded_at::date = CURRENT_DATE
)
INSERT INTO core.customer_mapping (golden_id, source_system, source_id)
SELECT g.golden_id, w.source_system, w.source_id
FROM with_key w
JOIN tmp_golden g ON g.group_key = w.group_key;
```

См. [§5.3](chapter-05-03.md).

---

## 12.3.6. Идемпотентность

Повторный прогон с теми же данными не должен создавать дубликаты. Варианты:

1. **Полная перезагрузка Core:** `TRUNCATE core.customer CASCADE` (включая маппинг) перед вставкой; подходит для демо и ночных перезагрузок.
2. **Upsert по group_key:** если Core уже содержит запись по `inn` или (name, phone), выполнять `UPDATE` вместо `INSERT`; маппинг — `ON CONFLICT (source_system, source_id) DO UPDATE`.
3. **Инкремент:** обрабатывать только новые/изменённые записи из Staging; для существующих `(source_system, source_id)` обновлять золотую запись по правилам. См. [§11.3.7](chapter-11-03.md), [§10.2](chapter-10-02.md).

Для учебного пайплайна достаточно полной перезагрузки с предварительной очисткой.

---

## 12.3.7. Проверки после загрузки

- **Уникальность:** в `core.customer` нет дубликатов по `inn` (при заполненном ИНН) и по бизнес-ключу.
- **Целостность маппинга:** для каждой строки в Staging существует ровно одна строка в `core.customer_mapping` с соответствующим `golden_id`.
- **Полнота:** `name` в Core не пустой. См. [§11.4](chapter-11-04.md).

---

## 12.3.8. Fuzzy matching (при отсутствии ключа)

Если ИНН и name+phone не дают надёжной группировки, можно использовать fuzzy matching по `name` в PostgreSQL (модуль `fuzzystrmatch`):

```sql
-- Только для иллюстрации; блокировка по первым символам
SELECT a.source_id AS id1, b.source_id AS id2,
       similarity(a.name, b.name) AS sim
FROM staging.customer a
JOIN staging.customer b ON a.source_system < b.source_system 
    AND LEFT(LOWER(a.name), 3) = LEFT(LOWER(b.name), 3)
WHERE a.loaded_at::date = CURRENT_DATE AND b.loaded_at::date = CURRENT_DATE
  AND similarity(a.name, b.name) > 0.8;
```

Для production-объёмов fuzzy matching лучше выносить в Python (rapidfuzz) или отдельный ETL. См. [§5.2](chapter-05-02.md), [§11.3.5](chapter-11-03.md).

---

## Ключевое

- **Ключ группировки:** ИНН (10/12 цифр), иначе name+phone, иначе source_system|source_id.
- **Слияние:** по приоритетам: name, inn, address — ERP; email, phone, registration_date — CRM; `MAX(...) FILTER (WHERE source_system = ...)` в SQL.
- **Core:** core.customer (золотая запись), core.customer_mapping (source_system, source_id → golden_id).
- **Реализация:** CTE с group_key, merge-агрегация, INSERT в Core, INSERT маппинга; пошагово через временные таблицы.
- **Идемпотентность:** полная перезагрузка (TRUNCATE) или upsert по ключу.

В [§12.4](chapter-12-04.md) реализуем **распространение в витрины**: загрузка из Core в mart.dim_customer, проверки качества.
