# §11.4 Профилирование и узкие места

В [§11.3](chapter-11-03.md) мы рассмотрели pushdown и SQL-override. **Профилирование** — анализ метрик выполнения сессии для выявления узких мест (bottleneck). В этом разделе — Session Log и метрики, Performance Details (performance counters), порядок поиска bottleneck (target, source, mapping, session, system) и рекомендации по оптимизации. Подробнее практика — в [§12.1](chapter-12-01.md). См. [Глоссарий](glossary.md).

---

## 11.4.1. Session Log и метрики

**Session Log** содержит информацию о задачах Integration Service во время сессии: allocation of heap memory, pre-session commands, SQL для reader/writer, время загрузки target, ошибки, post-session commands, load summary (reader, writer, DTM statistics). Источник: [Session Logs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-and-workflow-logs/session-logs.html).

Integration Service создаёт один session log на сессию; при нескольких сессиях в workflow — отдельный лог на каждую; на grid — один лог на каждый DTM process. Источник: [Session Logs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-and-workflow-logs/session-logs.html).

**Load summary** — ключевые метрики: Source rows read, Target rows loaded, Rejected rows, throughput (rows/sec, bytes/sec). См. [§10.2.6](chapter-10-02.md).

---

## 11.4.2. Performance Details и counters

**Performance Details** — счётчики, помогающие оценить эффективность сессии и маппинга. Отображаются в Workflow Monitor или в performance details file. Источник: [Performance Details](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor-details/performance-details.html).

Source Qualifier, Normalizer и target имеют дополнительные counters, показывающие эффективность передачи данных в/из buffer; используются для поиска bottleneck. Источник: [Understanding Performance Counters](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor-details/performance-details/understanding-performance-counters.html).

**Базовые counters** (все трансформации): input rows, output rows, error rows. Источник: [Understanding Performance Counters](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor-details/performance-details/understanding-performance-counters.html).

| Трансформация | Дополнительные counters |
|---------------|-------------------------|
| **Aggregator, Rank** | readfromcache, writetocache, readfromdisk, writetodisk, newgroupkey, oldgroupkey |
| **Lookup** | rowsinlookupcache |
| **Joiner** | inputMasterRows, inputDetailRows, readfromcache, writetocache, readfromdisk, writetodisk, duplicaterows |
| **Source Qualifier, Target** | efficiency (0–100%); high 80–100%, low 0–20% |

При партиционировании — отдельный набор counters для каждой партиции. Источник: [Understanding Performance Counters](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor-details/performance-details/understanding-performance-counters.html).

---

## 11.4.3. Bottleneck: порядок поиска

**Bottleneck** (узкое место) — компонент, ограничивающий производительность сессии. Стратегия: выявить bottleneck, устранить, повторить до достижения целевой производительности. Источник: [Bottlenecks Overview](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/bottlenecks/bottlenecks-overview.html).

**Порядок поиска:**

1. **Target** — writer медленнее reader; индексы, constraints, bulk load.
2. **Source** — reader медленнее writer; индексы, SQL-override, Source Filter.
3. **Mapping** — трансформации; Lookup cache, Aggregator, Joiner.
4. **Session** — buffer, партиционирование, concurrent sessions.
5. **System** — CPU, I/O, память; мониторинг ОС.

Источник: [Bottlenecks Overview](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/bottlenecks/bottlenecks-overview.html).

**Методы выявления:** test sessions (flat file source/target для изоляции), анализ performance details, thread statistics, мониторинг системы (CPU, I/O, paging). Источник: [Bottlenecks Overview](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/bottlenecks/bottlenecks-overview.html).

---

## 11.4.4. Рекомендации по оптимизации

По [Performance Tuning Overview](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/performance-tuning-overview/performance-tuning-overview.html):

| Область | Действия |
|---------|----------|
| **Target** | Bulk load, отключение индексов при загрузке, партиционирование target. |
| **Source** | Source Filter, SQL-override, индексы, pushdown. |
| **Mapping** | Lookup cache, join в БД вместо Lookup, упрощение логики. |
| **Session** | Партиционирование, buffer size, pushdown. |
| **System** | Ресурсы Integration Service node, concurrent sessions. |

**Важно:** менять по одной переменной за раз; замерять до и после; при отсутствии улучшения — вернуть исходную конфигурацию. После устранения bottleneck — рассмотреть увеличение партиций. Источник: [Performance Tuning Overview](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/performance-tuning-overview/performance-tuning-overview.html).

**Performance Details** помогают настроить buffer block size и index/data cache для Aggregator, Rank, Lookup, Joiner. Источник: [Performance Details](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor-details/performance-details.html).

---

## 11.4.5. Типичные ошибки

- **Оптимизация без профилирования:** сначала выявить bottleneck по counters; не увеличивать buffer или партиции наугад.
- **Игнорирование readfromdisk/writetodisk:** высокие значения — кеш переполнен; увеличить cache size или рассмотреть persistent cache.
- **Source/target efficiency 0–20%:** низкая эффективность — узкое место; оптимизировать источник или приёмник.
- **Менять несколько параметров сразу:** невозможно определить, что помогло; менять по одному.

---

## Ключевое

- **Session Log** — load summary, Source rows read, Target rows loaded, Rejected rows.
- **Performance Details** — counters по трансформациям; input/output/error rows; readfromcache, readfromdisk; efficiency.
- **Bottleneck** — порядок: Target → Source → Mapping → Session → System.
- **Методы:** test sessions, performance details, thread statistics, мониторинг системы.
- **Оптимизация:** одна переменная за раз; замер до и после; Performance Details для cache и buffer.

В [§12.1](chapter-12-01.md) мы перейдём к практике: постановка задачи пайплайна загрузки в DWH.
