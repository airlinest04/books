# §11.2 Lookup cache и buffer

В [§11.1](chapter-11-01.md) мы рассмотрели партиционирование. **Lookup cache** и **buffer** — ключевые настройки памяти для производительности: кеш снижает обращения к источнику Lookup, buffer — объём памяти DTM для передачи данных между трансформациями. В этом разделе — настройка Lookup cache в контексте оптимизации (связь с [§7.2](chapter-07-02.md)), DTM Buffer Size и Default Buffer Block Size, рекомендации по избежанию узких мест. Подробнее pushdown — в [§11.3](chapter-11-03.md). См. [Глоссарий](glossary.md).

---

## 11.2.1. Lookup cache: настройки для производительности

Lookup cache подробно описан в [§7.2](chapter-07-02.md). В контексте оптимизации — ключевые настройки. Источник: [Tips for Lookup Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/tips-for-lookup-transformations.html).

| Настройка | Рекомендация |
|-----------|--------------|
| **Cache small lookup tables** | Кешировать малые справочники; результат тот же, производительность выше. |
| **Persistent cache** | Для статичных справочников — persistent cache; Integration Service переиспользует файлы, не читая источник. |
| **Cache size** | Задать достаточный размер; при переполнении — cache files на диске, cache miss — запрос к источнику. |
| **Concurrent caches** | При нескольких Lookup в начале маппинга — concurrent; кеши строятся параллельно. |
| **Shared cache** | При нескольких Lookup с одной таблицей — shared cache; один кеш вместо нескольких. |

**Индексы:** если есть доступ к БД — создать индекс на колонках Lookup Condition; важно для uncached и при cache miss в partial cache. Источник: [Tips for Lookup Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/tips-for-lookup-transformations.html).

**Join в БД:** если Lookup и Source в одной БД и кеширование невозможно — рассмотреть join в Source Qualifier вместо Lookup. Источник: [Tips for Lookup Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/tips-for-lookup-transformations.html).

---

## 11.2.2. DTM Buffer Size

**DTM Buffer Size** — объём памяти, выделяемой Integration Service для DTM (Data Transformation Manager) при выполнении сессии. Reader, transformation и writer threads используют buffer blocks для передачи данных от источников к приёмникам. Источник: [Understanding Buffer Memory Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/understanding-buffer-memory/understanding-buffer-memory-overview.html).

Настройка: Session → Properties → Performance → DTM Buffer Size. Увеличение создаёт больше buffer blocks; улучшает производительность при кратковременных замедлениях (например, writer ждёт reader). Источник: [Increasing DTM Buffer Size](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/optimizing-sessions/buffer-memory/increasing-dtm-buffer-size.html).

**Рекомендация:** увеличивать кратными Default Buffer Block Size. Производительность обычно растёт до определённого предела; если роста нет — buffer не является узким местом. Обычно не требуется более 1 GB. Источник: [Increasing DTM Buffer Size](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/optimizing-sessions/buffer-memory/increasing-dtm-buffer-size.html).

---

## 11.2.3. Default Buffer Block Size

**Default Buffer Block Size** — размер одного buffer block. DTM делит выделенную память на блоки; размер блока должен быть больше precision самой большой строки в source или target. Источник: [Understanding Buffer Memory Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/understanding-buffer-memory/understanding-buffer-memory-overview.html).

По умолчанию: 64 000 байт или размер самой большой строки — что больше. Integration Service выделяет минимум два buffer block на каждый source и target в партиции. Источник: [Understanding Buffer Memory Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/understanding-buffer-memory/understanding-buffer-memory-overview.html).

Настройка: Session → Config Object → Advanced → Default Buffer Block Size. Источник: [Optimizing the Buffer Block Size](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/optimizing-sessions/buffer-memory/optimizing-the-buffer-block-size.html).

**Когда увеличивать:** при необычно больших строках — увеличить block size. **Когда уменьшать:** при ограниченной памяти и большом числе sources/targets/партиций. Источник: [Optimizing the Buffer Block Size](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/optimizing-sessions/buffer-memory/optimizing-the-buffer-block-size.html).

**Расчёт:** сложить precision всех колонок target (и source); выбрать максимум по всем source и target. Если precision 33 000, а block size 64 000 — в блок помещается около двух строк. Источник: [Optimizing the Buffer Block Size](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/optimizing-sessions/buffer-memory/optimizing-the-buffer-block-size.html).

---

## 11.2.4. Избежание узких мест

**Lookup:**
- Условия Lookup: сначала `=` (equality), затем `<`, `>`, `<=`, `>=`, затем `!=`. Источник: [Tips for Lookup Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/tips-for-lookup-transformations.html).
- Partial cache при множестве уникальных ключей — частые cache miss; рассмотреть full cache, persistent cache или join в БД.
- Очень большие справочники (>300 MB) — Joiner или Source Qualifier join вместо Lookup. См. [§7.2](chapter-07-02.md).

**Buffer:**
- При ошибке инициализации сессии (недостаточно памяти) — уменьшить DTM Buffer Size или Default Buffer Block Size.
- При XML sources/targets — buffer blocks должны быть минимум в два раза больше числа групп. Источник: [Understanding Buffer Memory Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/understanding-buffer-memory/understanding-buffer-memory-overview.html).

**Общее:** профилирование (Session Log, Performance view) — см. [§11.4](chapter-11-04.md) — помогает выявить узкое место перед настройкой.

---

## 11.2.5. Типичные ошибки

- **Увеличение buffer без профилирования:** buffer редко является узким местом; сначала проверить reader/writer/transformation.
- **Слишком малый cache size при partial cache:** частые cache miss; увеличить или перейти на full cache.
- **Lookup вместо join:** при одном источнике и одной БД — join в Source Qualifier часто быстрее.
- **Buffer block size меньше размера строки:** сессия может не инициализироваться или работать некорректно.

---

## Ключевое

- **Lookup cache:** кешировать малые справочники; persistent cache для статичных; shared cache при нескольких Lookup; индексы на Lookup Condition.
- **Join в БД:** при Lookup и Source в одной БД — рассмотреть join вместо Lookup.
- **DTM Buffer Size** — Properties → Performance; увеличивать кратными block size; обычно до 1 GB.
- **Default Buffer Block Size** — Config Object → Advanced; больше precision самой большой строки.
- **Профилирование** — перед настройкой выявить узкое место.

В [§11.3](chapter-11-03.md) мы разберём pushdown и SQL-override для переноса логики в источник.
