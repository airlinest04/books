# §13.1 Informatica Cloud (IICS)

В [§12.4](chapter-12-04.md) мы завершили практику пайплайна загрузки в DWH на PowerCenter. **Informatica Cloud (IICS)** — облачная альтернатива PowerCenter; в этом разделе — архитектура IICS, отличие от PowerCenter, Cloud Data Integration (CDI), Secure Agent, сценарии миграции. Это обзорный раздел: детали реализации IICS см. в официальной документации. Подробнее AI-движок CLAIRE — в [§13.2](chapter-13-02.md). См. [Глоссарий](glossary.md).

---

## 13.1.1. IICS и IDMC: платформа

**Informatica Intelligent Cloud Services (IICS)** — облачная платформа интеграции данных Informatica. Работает как SaaS: веб-интерфейс, API-driven архитектура, эластичное масштабирование. **Informatica Intelligent Data Management Cloud (IDMC)** — единая облачная платформа Informatica, объединяющая IICS с сервисами качества данных, governance, каталогом, MDM и др. IICS — часть IDMC; в контексте ETL чаще говорят IICS или Cloud Data Integration (CDI). См. [§1.2](chapter-01-02.md).

| Аспект | PowerCenter | IICS |
|--------|-------------|------|
| **Интерфейс** | Толстые клиенты (Designer, Workflow Manager) | Веб-интерфейс; доступ из браузера |
| **Размещение** | On-premise; серверы заказчика | SaaS; управление в облаке Informatica |
| **Масштабирование** | Ручное; Grid, узлы | Эластичное; serverless Spark, кластеры по требованию |
| **Лицензирование** | CPU/capacity-based; единовременные платежи | Подписка; consumption-based |
| **Облачные DWH** | Коннекторы; не нативная среда | Нативная поддержка Snowflake, BigQuery, Databricks, Redshift, Azure Synapse |

Источник: [What is Cloud Data Integration](https://www.informatica.com/resources/articles/what-is-cloud-data-integration.html).

---

## 13.1.2. Cloud Data Integration (CDI)

**Cloud Data Integration (CDI)** — сервис ETL/ELT в рамках IICS. Включает:

- **Визуальная разработка:** маппинги, задачи (Tasks), расписание; аналогично PowerCenter, но в веб-интерфейсе.
- **Коннекторы:** нативные подключения к облачным DWH (Snowflake, BigQuery, Databricks, Redshift, Azure Synapse), SaaS (Salesforce, SAP и др.), файловым хранилищам (S3, Azure Blob).
- **Pushdown optimization:** выполнение логики в целевой СУБД; снижение объёма данных, передаваемых через сеть. См. [§11.3](chapter-11-03.md), [§13.3](chapter-13-03.md).
- **Serverless Spark:** Cloud Data Integration-Elastic — тяжёлые нагрузки выполняются на кластерах Spark; масштабирование по требованию; кластеры выключаются после завершения.
- **Mass ingestion:** массовая загрузка из файлов, потоков, БД; multi-latency (batch, streaming, edge).

По данным Informatica: cloud data integration обеспечивает **гибкость** (быстрое развёртывание паттернов, подключение on-premise и облачных источников), **скорость** (шаблоны, переиспользуемые маппинги, интеграции за часы или минуты) и **масштабируемость** (serverless-движок, динамическое масштабирование). Источник: [What is Cloud Data Integration](https://www.informatica.com/resources/articles/what-is-cloud-data-integration.html).

---

## 13.1.3. Secure Agent и доступ к on-premise

При облачной интеграции источники и приёмники могут находиться в разных средах: облако и on-premise. **Secure Agent** — агент IICS, устанавливаемый в периметре заказчика; через него IICS получает доступ к on-premise БД и файлам без выноса данных в облако Informatica. См. [§1.2](chapter-01-02.md).

**Принцип работы:**
- Secure Agent устанавливается на сервер в сети заказчика.
- IICS отправляет агенту инструкции (подключиться к источнику, выполнить запрос, передать данные).
- Данные читаются агентом на стороне заказчика; при pushdown преобразования выполняются в БД заказчика или в целевой облачной БД.
- Credentials и чувствительные данные не хранятся в облаке Informatica; агент использует локальные или защищённые хранилища.

**Сценарии:**
- Источник on-premise (Oracle, SQL Server) → Target облачный (Snowflake, BigQuery): агент читает из источника, передаёт в облако.
- Источник облачный → Target on-premise: агент записывает в целевую БД.
- Гибрид: несколько агентов для разных сетей или регионов.

---

## 13.1.4. Сценарии миграции PowerCenter → IICS

Организации с существующими маппингами PowerCenter могут планировать переход в IICS. Варианты:

| Вариант | Описание |
|---------|----------|
| **PowerCenter Cloud Edition** | PowerCenter в облаке (AWS, Azure, GCP); минимальные изменения в маппингах; инфраструктура в облаке заказчика. |
| **Миграция в IICS** | Конвертация маппингов в формат CDI; полный переход на облачную платформу; инструменты Informatica поддерживают переиспользование до 100% логики при конвертации. |
| **Гибрид** | Часть остаётся на PowerCenter, часть переходит в IICS; сосуществование по слоям (PowerCenter → Staging, IICS → DWH) или доменам. |

**Поэтапная миграция:**
- Выбор пилотных маппингов (простая логика, облачные target).
- Конвертация и тестирование; сверка данных.
- Постепенное перенесение workflow; PowerCenter и IICS работают параллельно до завершения миграции.

По данным Informatica: миграция PowerCenter в IICS даёт ускорение развёртывания (до 8 раз быстрее), переиспользование до 100% активов при конвертации, снижение затрат до 50%. Источник: [PowerCenter to Cloud Modernization](https://www.informatica.com/products/data-integration/powercenter.html).

---

## 13.1.5. Когда выбирать IICS

IICS предпочтителен, если:

- **Целевой DWH — облачный:** Snowflake, BigQuery, Databricks, Redshift, Azure Synapse.
- **Новый проект:** нет legacy PowerCenter; можно сразу строить интеграции в облачной среде.
- **Интеграция SaaS и API:** Salesforce, Workday, SAP Cloud, REST API, потоковые источники.
- **Гибкость и скорость:** быстрые изменения, масштабирование по требованию, подписочная модель без капитальных затрат.
- **Мультиоблачность:** данные распределены по AWS, Azure, GCP и on-premise; единая платформа для интеграции.

PowerCenter остаётся выбором при строгих требованиях к размещению данных, существующих инвестициях в on-premise и интеграции с legacy-системами. См. [§1.2.4](chapter-01-02.md), [§1.2.5](chapter-01-02.md).

---

## Ключевое

- **IICS** — облачная платформа интеграции; веб-интерфейс, эластичное масштабирование, нативная поддержка облачных DWH и SaaS.
- **IDMC** — единая облачная платформа Informatica; IICS — часть IDMC.
- **CDI** — Cloud Data Integration; ETL/ELT, коннекторы, pushdown, serverless Spark.
- **Secure Agent** — агент в периметре заказчика; доступ IICS к on-premise источникам без выноса данных.
- **Миграция PowerCenter → IICS:** PowerCenter Cloud Edition, конвертация в CDI, гибрид; инструменты поддерживают переиспользование логики.

В [§13.2](chapter-13-02.md) мы рассмотрим CLAIRE — AI-движок Informatica для интеллектуальной оптимизации и рекомендаций по маппингам.
