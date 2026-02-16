# §12.4 Распространение в витрины

В [§12.3](chapter-12-03.md) мы сформировали золотую запись в Core и маппинг. В этом разделе реализуем **распространение в витрины**: загрузка из Core в измерение `mart.dim_customer`, проверки качества и завершение пайплайна. Глава 12 закрывается полным циклом MDM-пайплайна. См. [Глоссарий](glossary.md).

---

## 12.4.1. Роль измерения dim_customer

**Измерение** `dim_customer` — таблица измерений в витрине; строится из Core эталонов и используется в джойнах с фактами. См. [§10.3](chapter-10-03.md).

Структура (без SCD для упрощения):

| Поле | Тип | Описание |
|------|-----|----------|
| customer_id | BIGINT PK | Суррогатный ключ = golden_id |
| name | VARCHAR(500) | Наименование / ФИО |
| inn | VARCHAR(12) | ИНН |
| email | VARCHAR(255) | Email |
| phone | VARCHAR(50) | Телефон |
| address | TEXT | Адрес |
| registration_date | DATE | Дата регистрации |
| updated_at | TIMESTAMP | Момент обновления записи в Mart |

Для историчности добавляют `effective_from`, `effective_to`, `is_current` (SCD Type 2). См. [§10.4](chapter-10-04.md).

---

## 12.4.2. Загрузка из Core в Mart

Простейший вариант — полная перезагрузка измерения из Core:

```sql
-- PostgreSQL; полная перезагрузка dim_customer
TRUNCATE mart.dim_customer;

INSERT INTO mart.dim_customer (
    customer_id, name, inn, email, phone, address, registration_date, updated_at
)
SELECT
    golden_id AS customer_id,
    name,
    inn,
    email,
    phone,
    address,
    registration_date,
    CURRENT_TIMESTAMP AS updated_at
FROM core.customer;
```

При ежедневной полной перезагрузке Core (как в [§12.3.6](chapter-12-03.md)) измерение каждый раз пересоздаётся из актуального Core. См. [§10.3](chapter-10-03.md).

---

## 12.4.3. Инкрементальное обновление измерения

Если Core обновляется инкрементально, измерение тоже обновляют выборочно:

```sql
-- Upsert: вставка новых, обновление изменённых
INSERT INTO mart.dim_customer (customer_id, name, inn, email, phone, address, registration_date, updated_at)
SELECT golden_id, name, inn, email, phone, address, registration_date, CURRENT_TIMESTAMP
FROM core.customer c
WHERE NOT EXISTS (SELECT 1 FROM mart.dim_customer d WHERE d.customer_id = c.golden_id)
ON CONFLICT (customer_id) DO UPDATE SET
    name = EXCLUDED.name,
    inn = EXCLUDED.inn,
    email = EXCLUDED.email,
    phone = EXCLUDED.phone,
    address = EXCLUDED.address,
    registration_date = EXCLUDED.registration_date,
    updated_at = CURRENT_TIMESTAMP;
```

Требуется `UNIQUE (customer_id)` или `PRIMARY KEY` на `mart.dim_customer`. См. [§10.2](chapter-10-02.md).

---

## 12.4.4. SCD Type 2 (опционально)

Для истории изменений атрибутов применяют SCD Type 2:

1. Сравнить текущее состояние Core с последней версией в измерении (по golden_id, `is_current = 1`).
2. При изменении атрибутов: закрыть текущую версию (`effective_to = CURRENT_DATE`, `is_current = 0`).
3. Вставить новую версию (`effective_from = CURRENT_DATE`, `effective_to = NULL`, `is_current = 1`); новый surrogate_key (отдельная последовательность для версий).
4. Новые записи из Core — INSERT с `effective_from = CURRENT_DATE`.

При загрузке фактов используют surrogate_key версии, актуальной на дату факта. См. [§10.4](chapter-10-04.md), книга по DWH.

Для учебного пайплайна достаточно полной перезагрузки без SCD.

---

## 12.4.5. Проверки качества

Перед открытием витрины для потребления выполняют проверки ([§11.4](chapter-11-04.md)):

| Проверка | Описание |
|----------|----------|
| Row count | Число записей в dim_customer = числу в core.customer. |
| Уникальность | Нет дубликатов по customer_id. |
| Полнота | Обязательные поля (name) не пусты. |
| Сироты | Факты ссылаются только на существующие customer_id; при наличии fact_sales — проверить отсутствие orphan-ключей. |

Пример проверки row count:

```sql
SELECT
    (SELECT COUNT(*) FROM core.customer) AS core_count,
    (SELECT COUNT(*) FROM mart.dim_customer) AS mart_count,
    (SELECT COUNT(*) FROM core.customer) - (SELECT COUNT(*) FROM mart.dim_customer) AS diff;
```

При `diff <> 0` — алерт. Проверка сирот (если есть fact_sales):

```sql
SELECT f.customer_id
FROM fact_sales f
LEFT JOIN mart.dim_customer d ON d.customer_id = f.customer_id
WHERE d.customer_id IS NULL
LIMIT 10;
```

Пустой результат — сирот нет. См. [§10.3.7](chapter-10-03.md), книга «Качество данных» [§8.4](../data-quality/chapter-08-04.md).

---

## 12.4.6. Итоговая цепочка пайплайна

Полный цикл главы 12:

```
1. Raw: загрузка срезов из CRM и ERP (raw.crm_customer, raw.erp_counterparty)
2. Staging: объединение, нормализация, staging.customer
3. Дедупликация + слияние: group_key, merge по приоритетам
4. Core: core.customer (золотая запись), core.customer_mapping
5. Mart: mart.dim_customer из Core
6. Проверки: row count, уникальность, сироты
```

Оркестратор (Airflow, cron, ETL-пайплайн) выполняет шаги по порядку; при провале проверок — алерт, витрина не открывается. См. [§12.1.2](chapter-12-01.md), книга «Качество данных» [§6.4](../data-quality/chapter-06-04.md).

---

## 12.4.7. Использование маппинга при загрузке фактов

При загрузке транзакций (продажи, заказы) факт содержит `source_id` (client_id из CRM или counterparty_id из ERP). Для связи с измерением используют маппинг:

```sql
-- Пример: загрузка fact_sales; транзакция содержит source_system, source_id
INSERT INTO fact_sales (..., customer_id, ...)
SELECT ...,
    COALESCE(m.golden_id, -1) AS customer_id,  -- -1 = Unknown при отсутствии маппинга
    ...
FROM staging.sales s
LEFT JOIN core.customer_mapping m 
    ON m.source_system = s.customer_source AND m.source_id = s.customer_id::text;
```

Эталонный пайплайн должен завершаться **до** загрузки фактов: маппинг и измерение должны быть актуальны. См. [§11.1](chapter-11-01.md), [§10.3.3](chapter-10-03.md).

---

## 12.4.8. Типичные ошибки

- **Загружать факты до эталонов:** при джойне с dim_customer по source_id будут сироты; нужен golden_id из маппинга.
- **Пропустить проверки:** дефекты в измерении (дубликаты, пустые поля) искажают отчёты; проверки обязательны перед открытием витрины.
- **Не синхронизировать порядок задач:** эталонный пайплайн (Raw → Staging → Core → Mart) должен выполняться перед пайплайном фактов.
- **Использовать source_id в факте вместо golden_id:** при нескольких источниках один клиент имеет несколько source_id; факт должен хранить единый ключ (golden_id).

---

## Ключевое

- **Распространение:** загрузка из core.customer в mart.dim_customer; полная перезагрузка или инкрементальный upsert.
- **Структура измерения:** customer_id = golden_id; атрибуты из Core; при необходимости — SCD Type 2.
- **Проверки:** row count Core vs Mart; уникальность; полнота; отсутствие сирот в фактах.
- **Порядок:** эталонный пайплайн выполняется до загрузки фактов; маппинг используется при разрешении source_id → golden_id.
- **Полный цикл:** Raw → Staging → дедупликация → Core + маппинг → Mart → проверки.

Глава 12 завершена. В приложениях — [Глоссарий](glossary.md) и [Список литературы](sources.md).
