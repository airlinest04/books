# §11.3 Pushdown и SQL-override

В [§11.2](chapter-11-02.md) мы рассмотрели Lookup cache и buffer. **Pushdown optimization** — перенос логики трансформаций в источник или приёмник (БД); Integration Service генерирует SQL и выполняет его на стороне БД, что снижает объём данных и нагрузку на DTM. **SQL-override** в Source Qualifier — ручной способ задать запрос к источнику. В этом разделе — типы pushdown (source-side, target-side, full), настройка сессии, SQL-override в контексте pushdown и ограничения. Подробнее профилирование — в [§11.4](chapter-11-04.md). См. [Глоссарий](glossary.md).

---

## 11.3.1. Pushdown optimization: обзор

**Pushdown optimization** — механизм, при котором Integration Service переводит логику трансформаций в SQL и отправляет запросы в source или target database. БД выполняет обработку вместо Integration Service; передаётся только результат. Источник: [Pushdown Optimization Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/pushdown-optimization-overview.html).

Объём логики, которую можно перенести, зависит от типа БД, логики трансформаций и конфигурации маппинга и сессии. Всё, что не удаётся перенести, выполняется в Integration Service. Источник: [Pushdown Optimization Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/pushdown-optimization-overview.html).

**Pushdown Optimization Viewer** — инструмент для предпросмотра SQL и логики, которую Integration Service может перенести в БД, а также сообщений о причинах непереносимости. Источник: [Pushdown Optimization Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/pushdown-optimization-overview.html).

---

## 11.3.2. Типы pushdown

| Тип | Описание |
|-----|----------|
| **Source-side** | Integration Service переносит максимум логики в source database; генерирует SELECT; читает результат и обрабатывает оставшиеся трансформации. |
| **Target-side** | Перенос логики в target database. |
| **Full** | Попытка перенести всю логику в target; при невозможности — source-side и target-side. |

Источник: [Pushdown Optimization Types](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/pushdown-optimization-types.html).

**Source-side:** Integration Service анализирует маппинг от источника до target или до первой трансформации, которую нельзя перенести; генерирует и выполняет SELECT; читает результат и обрабатывает остальное. Источник: [Running Source-Side Pushdown Optimization Sessions](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/pushdown-optimization-types/running-source-side-pushdown-optimization-sessions.html).

Source-side применим, когда source и target в разных БД (например, Teradata → Oracle). Источник: [Pushdown Optimization Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/pushdown-optimization-overview.html).

---

## 11.3.3. Настройка сессии

Pushdown настраивается в **Session properties**. Может потребоваться правка трансформаций, маппинга или Config Object для переноса большей логики. Источник: [Configuring Sessions for Pushdown Optimization](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/configuring-sessions-for-pushdown-optimization.html).

Session → Properties → Pushdown Optimization (или Config Object → Pushdown Options). Выбор типа: Source, Target или Full. Источник: [Configuring Sessions for Pushdown Optimization](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization/configuring-sessions-for-pushdown-optimization.html).

---

## 11.3.4. SQL-override и pushdown

SQL-override в Source Qualifier описан в [§6.1](chapter-06-01.md). При pushdown Integration Service создаёт view на основе SQL-override и выполняет запрос к этой view. Источник: [Source Qualifier Transformation with an SQL Override](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization-and-transformations/source-qualifier-transformation/source-qualifier-transformation-with-an-sql-override.html).

**Ограничения при pushdown с SQL-override:**

| Ограничение | Описание |
|-------------|----------|
| **Порядок портов** | SELECT должен перечислять колонки в порядке портов Source Qualifier; иначе — сбой или неверные данные. |
| **View** | Сессия должна быть настроена на pushdown с view. |
| **Sequence** | SQL-override не должен ссылаться на database sequence — сессия завершится ошибкой. |
| **ORDER BY** | При ORDER BY в SQL-override и pushdown в DB2, SQL Server, Sybase ASE, Teradata — сессия завершится ошибкой. |
| **Select Distinct** | При SQL-override настройка Select Distinct игнорируется. |
| **Партиции** | При нескольких партициях SQL-override нужно задать для всех. |
| **Teradata** | Для Teradata при SQL-override отключить создание temporary views; используются derived tables. |

PowerCenter не проверяет синтаксис SQL-override; тестировать запрос на источнике до использования. Источник: [Source Qualifier Transformation with an SQL Override](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/pushdown-optimization-and-transformations/source-qualifier-transformation/source-qualifier-transformation-with-an-sql-override.html).

---

## 11.3.5. Когда применять

**Source Filter и Source Qualifier:** фильтрация и join на уровне источника (default query, Source Filter, User-Defined Join) — всегда выполняются в БД; это «естественный» pushdown без отдельной настройки. См. [§6.1](chapter-06-01.md).

**Pushdown optimization:** когда нужно перенести Expression, Filter, Aggregator и другие трансформации в БД. Эффективно при:
- большом объёме данных;
- source и target в одной БД (full pushdown) или разных (source-side);
- логика выразима в SQL целевой БД.

**SQL-override:** когда default query недостаточен — произвольный SQL (сложные join, подзапросы, специфичные функции БД). При pushdown — учитывать ограничения выше.

---

## 11.3.6. Типичные ошибки

- **ORDER BY в SQL-override при pushdown в DB2/SQL Server/Sybase/Teradata:** сессия падает; убрать ORDER BY или отключить pushdown для Source Qualifier.
- **Sequence в SQL-override при pushdown:** сессия падает; sequence нельзя переносить в view.
- **Несовпадение порядка колонок в SELECT:** неверные данные или сбой.
- **Pushdown без проверки:** Pushdown Optimization Viewer показывает, что переносится; проверять перед production.

---

## Ключевое

- **Pushdown optimization** — перенос логики в source/target БД; Integration Service генерирует SQL; БД выполняет обработку.
- **Типы:** Source-side, Target-side, Full; настройка в Session properties.
- **Pushdown Optimization Viewer** — предпросмотр переносимой логики и сообщений.
- **SQL-override при pushdown:** порядок портов; ограничения по ORDER BY, sequence, Select Distinct; тестировать на источнике.
- **Source Filter и default query** — выполняются в БД по умолчанию; pushdown — для дополнительных трансформаций.

В [§11.4](chapter-11-04.md) мы разберём профилирование и узкие места: Session Log, Performance, Bottleneck, рекомендации.
