# §12.2 Загрузка и слияние из источников

В [§12.1](chapter-12-01.md) мы зафиксировали постановку: эталон «клиент» из CRM и ERP, единая структура Staging. В этом разделе реализуем **загрузку и слияние**: ETL из обоих источников в Raw и Staging, приведение к единой структуре, нормализацию полей. Дедупликация и золотая запись — в [§12.3](chapter-12-03.md); распространение в витрины — в [§12.4](chapter-12-04.md). См. [Глоссарий](glossary.md).

---

## 12.2.1. Схема Raw и Staging

**Raw** — срезы «как есть» для трассировки. Таблицы: `raw.crm_customer`, `raw.erp_counterparty` (или файлы в landing-зоне). Структура повторяет источник.

**Staging** — единая таблица `staging.customer` со структурой из [§12.1.4](chapter-12-01.md):

| Поле | Тип | Описание |
|------|-----|----------|
| source_system | varchar | 'CRM' или 'ERP' |
| source_id | varchar | client_id / counterparty_id |
| name | varchar | наименование / ФИО |
| inn | varchar | ИНН (только ERP) |
| email | varchar | email (только CRM) |
| phone | varchar | телефон |
| address | varchar | адрес (только ERP) |
| registration_date | date | дата регистрации (только CRM) |
| loaded_at | timestamp | момент загрузки в Staging |

См. [§10.1](chapter-10-01.md), [§8.2](chapter-08-02.md).

---

## 12.2.2. Загрузка из CRM в Staging

Извлечение из CRM (таблица или файл) и вставка в Staging с маппингом полей и нормализацией:

```sql
-- PostgreSQL; источник — таблица crm_ext.clients
INSERT INTO staging.customer (
    source_system, source_id, name, inn, email, phone, address, registration_date, loaded_at
)
SELECT
    'CRM'::varchar AS source_system,
    client_id::varchar AS source_id,
    TRIM(COALESCE(full_name, '')) AS name,
    NULL::varchar AS inn,
    LOWER(TRIM(COALESCE(email, ''))) AS email,
    regexp_replace(COALESCE(phone, ''), '[^0-9+]', '', 'g') AS phone,  -- только цифры
    NULL::varchar AS address,
    registration_date,
    CURRENT_TIMESTAMP AS loaded_at
FROM raw.crm_customer
WHERE load_date = CURRENT_DATE - 1
  AND client_id IS NOT NULL
  AND TRIM(COALESCE(full_name, '')) <> '';
```

Условие `TRIM(COALESCE(full_name, '')) <> ''` — проверка полноты обязательного поля name ([§12.1.6](chapter-12-01.md)). Невалидные записи можно логировать или помещать в кварантин. См. [§11.4](chapter-11-04.md).

---

## 12.2.3. Загрузка из ERP в Staging

Аналогично для ERP:

```sql
-- PostgreSQL; источник — raw.erp_counterparty
INSERT INTO staging.customer (
    source_system, source_id, name, inn, email, phone, address, registration_date, loaded_at
)
SELECT
    'ERP'::varchar AS source_system,
    counterparty_id::varchar AS source_id,
    TRIM(COALESCE(name, '')) AS name,
    regexp_replace(COALESCE(inn, ''), '[^0-9]', '', 'g') AS inn,  -- только цифры
    NULL::varchar AS email,
    regexp_replace(COALESCE(phone, ''), '[^0-9+]', '', 'g') AS phone,
    TRIM(COALESCE(address, '')) AS address,
    NULL::date AS registration_date,
    CURRENT_TIMESTAMP AS loaded_at
FROM raw.erp_counterparty
WHERE load_date = CURRENT_DATE - 1
  AND counterparty_id IS NOT NULL
  AND TRIM(COALESCE(name, '')) <> '';
```

ИНН оставляем только цифры; валидация длины (10/12) — отдельная проверка. См. [§11.4](chapter-11-04.md).

---

## 12.2.4. Объединение в одной задаче (UNION ALL)

Если оба источника доступны в одной БД, можно загружать одной вставкой:

```sql
INSERT INTO staging.customer (
    source_system, source_id, name, inn, email, phone, address, registration_date, loaded_at
)
SELECT * FROM (
    SELECT 'CRM', client_id::varchar, TRIM(full_name), NULL, LOWER(TRIM(email)),
           regexp_replace(COALESCE(phone,''), '[^0-9+]', '', 'g'), NULL, registration_date, CURRENT_TIMESTAMP
    FROM raw.crm_customer WHERE load_date = CURRENT_DATE - 1 AND client_id IS NOT NULL
    UNION ALL
    SELECT 'ERP', counterparty_id::varchar, TRIM(name), regexp_replace(COALESCE(inn,''), '[^0-9]', '', 'g'), NULL,
           regexp_replace(COALESCE(phone,''), '[^0-9+]', '', 'g'), TRIM(address), NULL, CURRENT_TIMESTAMP
    FROM raw.erp_counterparty WHERE load_date = CURRENT_DATE - 1 AND counterparty_id IS NOT NULL
) AS combined
WHERE TRIM(COALESCE(name, '')) <> '';
```

Порядок вставки не влияет на дедупликацию: Staging содержит все записи в единой структуре; дедупликация — следующий этап. См. [§11.3](chapter-11-03.md).

---

## 12.2.5. Нормализация полей

| Поле | Нормализация |
|------|--------------|
| name | TRIM, запрет пустой строки; опционально — приведение регистра (initcap для ФИО). |
| inn | только цифры; длина 10 (юрлицо) или 12 (физлицо) — проверка при валидации. |
| phone | только цифры и «+»; единый формат для сопоставления. |
| email | LOWER, TRIM; проверка формата — при валидации. |
| address | TRIM; опционально — нормализация через справочники. |

Нормализация уменьшает расхождения при дедупликации (одинаковые номера с разным форматированием не разойдутся). См. [§3.1](chapter-03-01.md), [§5.2](chapter-05-02.md).

---

## 12.2.6. Проверки на этапе Staging

Перед дедупликацией выполняют проверки качества ([§11.4](chapter-11-04.md)):

- **Полнота:** `name` и `source_id` не null и не пустые.
- **Валидность:** ИНН — 10 или 12 цифр; email — допустимый формат (регулярка или проверка на `@`).
- **Повтор в батче:** в рамках одной загрузки нет дублей по (source_system, source_id).

Пример проверки полноты:

```sql
SELECT COUNT(*) AS incomplete_count
FROM staging.customer
WHERE loaded_at::date = CURRENT_DATE
  AND (name IS NULL OR TRIM(name) = '' OR source_id IS NULL OR source_id = '');
```

При `incomplete_count > 0` — алерт или блокировка перехода к дедупликации. См. книга «Качество данных», [§5.2](../data-quality/chapter-05-02.md).

---

## 12.2.7. Инкрементальная vs полная загрузка

**Полная загрузка:** каждый прогон загружает весь срез из источника; Staging перед вставкой очищают (`TRUNCATE staging.customer` или `DELETE WHERE loaded_at::date = CURRENT_DATE`).

**Инкрементальная:** загружают только изменённые записи (CDC, `updated_at > last_load`); в Staging добавляют новые и обновляют существующие по (source_system, source_id). См. [§10.2](chapter-10-02.md).

Для практики достаточно полной загрузки: Raw обновляется ежедневно; Staging заполняется за текущий прогон; дедупликация обрабатывает актуальный срез.

---

## 12.2.8. Реализация в ETL-инструментах

**Informatica:** Source — CRM и ERP; два пайплаина или один с Union; Expression — нормализация; Target — staging.customer. См. книга «Informatica».

**dbt:** два staging-модели (`stg_crm_customer`, `stg_erp_counterparty`) и одна объединяющая (`stg_customer` = union двух). См. [§5.4](chapter-05-04.md).

**Python/pandas:** чтение из источников, приведение к единому DataFrame, нормализация, запись в Staging. Подходит при файловых источниках (CSV, Parquet).

---

## Ключевое

- **Raw:** срезы «как есть»; Staging — единая структура с source_system, source_id, name, inn, email, phone, address.
- **Загрузка:** INSERT из каждого источника с маппингом полей; UNION ALL при едином запросе.
- **Нормализация:** TRIM, только цифры для phone/inn, LOWER для email; подготовка к дедупликации.
- **Проверки:** полнота (name, source_id), валидность (ИНН, email), отсутствие дублей в батче.
- **Реализация:** SQL, Informatica, dbt, Python — по доступным инструментам.

В [§12.3](chapter-12-03.md) реализуем **дедупликацию и золотую запись**: обнаружение групп, слияние по правилам, загрузка в Core и маппинг.
