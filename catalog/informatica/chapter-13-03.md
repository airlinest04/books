# §13.3 Pushdown на Snowflake, Databricks и др.

В [§13.2](chapter-13-02.md) мы рассмотрели CLAIRE. **Pushdown** в облачных DWH — выполнение логики трансформаций непосредственно в Snowflake, Databricks, BigQuery, Redshift, Azure Synapse; снижение объёма передаваемых данных и затрат на data egress. В этом разделе — Advanced Pushdown Optimization (APDO), поддержка облачных DWH, ELT vs ETL в облаке, гибридные сценарии. Завершаем главу 13 обзором расширений PowerCenter. См. [Глоссарий](glossary.md).

---

## 13.3.1. ELT и pushdown в облаке

**ELT** (Extract, Load, Transform) — альтернатива ETL: данные сначала загружаются в целевое хранилище (data lake или DWH), затем преобразования выполняются в самом хранилище. В облачных DWH с масштабируемой вычислительной мощностью ELT часто эффективнее: не нужно перемещать данные туда-обратно через Integration Service; логика выполняется в нативном SQL-движке целевой платформы. См. [§1.1](chapter-01-01.md).

**Pushdown optimization** в облаке — механизм переноса логики трансформаций (filter, join, aggregation, sort) в облачный DWH или data lake. Informatica генерирует SQL или нативные команды платформы; выполнение происходит в целевой системе. Преимущества:
- **Снижение data egress:** облачные провайдеры взимают плату за передачу данных; при pushdown данные не покидают экосистему.
- **Производительность:** использование вычислительной мощности DWH; параллельная обработка.
- **Масштабируемость:** облачные DWH масштабируются по требованию.

Источник: [Why Advanced Pushdown Optimization Is a Must-Have](https://www.informatica.com/blogs/why-advanced-pushdown-optimization-is-a-must-have-capability-for-your-cloud-analytics-journey.html).

---

## 13.3.2. Advanced Pushdown Optimization (APDO)

**Advanced Pushdown Optimization (APDO)** — расширенная pushdown-оптимизация Informatica для облачных DWH и data lake. Отличается от классического pushdown PowerCenter ([§11.3](chapter-11-03.md)):

| Аспект | PowerCenter pushdown | APDO (IICS/CDI) |
|--------|----------------------|-----------------|
| **Среда** | On-premise БД (Oracle, Teradata и др.) | Облачные DWH и data lake (Snowflake, Databricks, BigQuery, Redshift, Azure Synapse) |
| **Ограничения** | Обычно source и target в одной БД для full pushdown | Поддержка разных source/target; cross-endpoint |
| **Реализация** | SQL | SQL и нативные команды экосистемы |
| **Фокус** | Производительность, нагрузка на DTM | Снижение затрат (data egress, compute hours) |

APDO доступен в **Cloud Data Integration (IICS)**; PowerCenter использует классический pushdown. При миграции в IICS маппинги могут быть настроены на APDO для облачных target. Источник: [Why Advanced Pushdown Optimization Is a Must-Have](https://www.informatica.com/blogs/why-advanced-pushdown-optimization-is-a-must-have-capability-for-your-cloud-analytics-journey.html).

---

## 13.3.3. Snowflake

**Snowflake** — облачный DWH с отдельным хранилищем и вычислениями; поддерживает масштабирование по требованию. Informatica предоставляет нативный коннектор Snowflake и **Advanced Pushdown Optimization** для Snowflake Data Cloud.

При APDO:
- Логика трансформаций (filter, join, aggregation, sort) переводится в SQL Snowflake.
- Выполнение происходит в Snowflake; данные не передаются в Integration Service.
- **Нулевые data egress charges** при source и target в Snowflake Data Cloud.
- Поддержка сценариев: загрузка из data lake в DWH с трансформацией; SCD и другие классические паттерны DWH.

Пользователь создаёт маппинг в визуальном интерфейсе CDI и включает режим pushdown optimization; движок генерирует оптимизированный SQL или нативные команды Snowflake. Источник: [ELT or Advanced Pushdown Optimization for Snowflake](https://www.informatica.com/blogs/why-you-should-use-informaticas-elt-or-advanced-pushdown-optimization-for-snowflake.html).

---

## 13.3.4. Databricks и другие облачные DWH

**Databricks** — платформа data lakehouse; объединяет хранилище и аналитику. Informatica поддерживает ELT и pushdown с Databricks Lakebase: трансформации выполняются в SQL-движке Databricks; оркестрация и масштабирование — через IICS. Источник: [Creating Trusted Data with IDMC and Databricks Lakebase](https://www.informatica.com/blogs/creating-trusted-data-for-analytics-and-ai-with-idmc-and-databricks-lakebase.html).

**Другие облачные DWH:**
- **Amazon Redshift** — pushdown в Redshift; использование вычислительной мощности кластера.
- **Google BigQuery** — pushdown в BigQuery; serverless-модель.
- **Azure Synapse** — pushdown в Synapse; интеграция с экосистемой Azure.

**Optimization Engine** в IICS выбирает наиболее эффективный режим: pushdown в облачный DWH, pushdown в экосистему (AWS, GCP, Azure), serverless Spark или традиционный ETL (через Secure Agent для малых объёмов). Источник: [Why Advanced Pushdown Optimization Is a Must-Have](https://www.informatica.com/blogs/why-advanced-pushdown-optimization-is-a-must-have-capability-for-your-cloud-analytics-journey.html).

---

## 13.3.5. Гибридные сценарии

**Гибридный сценарий** — сочетание PowerCenter (on-premise) и IICS (облако), или источников on-premise и облачных target. См. [§1.2.6](chapter-01-02.md).

| Сценарий | Реализация |
|----------|------------|
| **On-premise source → Cloud DWH** | Secure Agent читает из on-premise; IICS загружает в Snowflake/BigQuery/Databricks; при возможности — pushdown в target. |
| **Cloud source → On-premise target** | IICS читает из облака; Secure Agent записывает в on-premise. |
| **PowerCenter → Staging, IICS → DWH** | PowerCenter загружает в промежуточный слой (on-premise или гибрид); IICS выполняет загрузку в облачный DWH с pushdown. |
| **Multi-cloud** | Источники и target в разных облаках (AWS, Azure, GCP); IICS оркестрирует; pushdown в каждом target по возможности. |

**Затраты:** при переносе данных между облаками или из облака наружу взимается data egress; pushdown снижает объём передаваемых данных и затраты. Источник: [Why Advanced Pushdown Optimization Is a Must-Have](https://www.informatica.com/blogs/why-advanced-pushdown-optimization-is-a-must-have-capability-for-your-cloud-analytics-journey.html).

---

## 13.3.6. Когда применять pushdown в облаке

**Рекомендуется pushdown/ELT, когда:**
- Целевой DWH — Snowflake, BigQuery, Databricks, Redshift, Azure Synapse.
- Большие объёмы данных; важно снизить data egress и compute hours.
- Логика выразима в SQL целевой платформы (filter, join, aggregation, sort).
- Source и target в одной облачной экосистеме (например, оба в Snowflake) — нулевые или минимальные egress charges.

**Традиционный ETL (без pushdown) остаётся уместен, когда:**
- Малые объёмы; Secure Agent с традиционным ETL может выполнить быстрее при меньших compute hours.
- Сложная логика, не выразимая в SQL целевой БД.
- Строгие требования к размещению данных; обработка только в периметре заказчика.

---

## Ключевое

- **ELT** — загрузка в DWH, затем преобразования в самом DWH; предпочтителен для облачных DWH с масштабируемой мощностью.
- **APDO** — Advanced Pushdown Optimization в IICS; поддержка Snowflake, Databricks, BigQuery, Redshift, Azure Synapse; снижение data egress и затрат.
- **Snowflake:** нативный коннектор, APDO; нулевые egress при source и target в Snowflake Data Cloud.
- **Databricks и др.:** pushdown в SQL-движок; Optimization Engine выбирает режим (pushdown, Spark, ETL).
- **Гибрид:** Secure Agent для on-premise; IICS для облака; pushdown в облачных target при возможности.

Глава 13 завершена. Книга охватывает PowerCenter (Designer, Workflows, трансформации, производительность, практику загрузки в DWH) и обзор Informatica Cloud, CLAIRE и pushdown в облачных DWH. Дальнейшее изучение — [Глоссарий](glossary.md), [Список литературы](sources.md) и документация Informatica.
