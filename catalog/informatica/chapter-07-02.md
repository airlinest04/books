# §7.2 Lookup: кеш и производительность

В [§7.1](chapter-07-01.md) мы рассмотрели назначение и типы Lookup. **Кеш** Lookup хранит данные справочника в памяти (и при переполнении — в файлах), что снижает число обращений к источнику. В этом разделе разберём структуру кеша (index cache, data cache), полный и частичный кеш, persistent cache, shared cache и настройки производительности. См. [Глоссарий](glossary.md).

---

## 7.2.1. Структура кеша: index и data

При включённом кешировании Integration Service строит кеш при обработке первой строки в Cached Lookup. Источник: [Lookup Caches Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-caches/lookup-caches-overview.html).

**Index cache** — хранит значения из Lookup Condition (ключи поиска).

**Data cache** — хранит выходные значения (output ports) из справочника.

Объём памяти задаётся в свойствах трансформации или Session. При переполнении Integration Service сохраняет избыток в cache files в `$PMCacheDir`. По завершении сессии память освобождается, файлы удаляются (если не настроен persistent cache).

---

## 7.2.2. Полный и частичный кеш

**Полный кеш (full cache):** все данные справочника помещаются в память. Integration Service выделяет память по настройкам; при достаточном размере весь кеш в RAM. Источник: [Lookup Caches Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-caches/lookup-caches-overview.html).

**Частичный кеш (partial cache):** при нехватке памяти часть данных остаётся в памяти, остальное — в cache files на диске. При промахе (cache miss) Integration Service обращается к источнику, добавляет строку в кеш и возвращает значение. Производительность ниже, чем при полном кеше, из‑за обращений к диску и к источнику.

**Рекомендация:** кеширование эффективно для справочников до ~300 MB. Для очень больших справочников — рассмотреть Joiner или Source Qualifier join. Источник: [Types of Caches](https://docs.informatica.com/data-integration/powercenter/10-5/performance-tuning-guide/optimizing-transformations/optimizing-lookup-transformations/caching-lookup-tables/types-of-caches.html).

---

## 7.2.3. Static и Dynamic cache

**Static cache** — кеш по умолчанию. Загружается при первом запросе и не меняется во время сессии. Источник: [Lookup Caches Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-caches/lookup-caches-overview.html).

**Dynamic cache** — кеш обновляется во время сессии: Integration Service может вставлять и обновлять строки. Используется при Lookup по target table для SCD и определения insert/update. Только для Connected Lookup.

Unconnected Lookup поддерживает только static cache.

---

## 7.2.4. Persistent cache

**Persistent cache** — файлы кеша сохраняются после сессии и переиспользуются в следующих запусках. Integration Service загружает кеш из файлов вместо чтения источника, что ускоряет старт. Источник: [Using a Persistent Lookup Cache](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-caches/using-a-persistent-lookup-cache.html).

**Когда использовать:** справочник редко меняется между сессиями.

**Recache from source:** при изменении справочника можно настроить пересборку кеша из источника; persistent cache будет обновлён.

---

## 7.2.5. Shared cache

**Unnamed cache** — несколько Lookup в одном маппинге с совместимой структурой кеша могут использовать один кеш по умолчанию. Только static unnamed caches. Integration Service строит кеш при первом Lookup и использует его для остальных. Источник: [Sharing the Lookup Cache](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-caches/sharing-the-lookup-cache.html).

**Named cache** — persistent named cache для совместного использования между маппингами или между dynamic и static Lookup. Поддерживает static и dynamic.

---

## 7.2.6. Sequential и Concurrent caches

**Sequential** — кеш строится при поступлении первой строки в Lookup. Источник: [Building Connected Lookup Caches](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-caches/building-connected-lookup-caches.html).

**Concurrent** — Integration Service строит кеши нескольких Lookup параллельно, не дожидаясь данных. Ускоряет сессии с несколькими Lookup в начале потока.

Unconnected Lookup всегда строит кеш последовательно.

---

## 7.2.7. Настройки и рекомендации

| Настройка | Описание |
|-----------|----------|
| **Cache size** | Число строк с уникальными ключами; Integration Service рассчитывает размер index и data cache. |
| **Pre-build lookup cache** | Построение кеша до начала обработки данных (при concurrent caches). |
| **Flat file / Pipeline lookup** | Всегда кешируются; для flat file с sorted input — ограничения по группировке колонок. |

**Рекомендации:**
- Включать кеш для больших справочников.
- Persistent cache — при стабильном справочнике.
- Shared cache — при нескольких Lookup с одной и той же таблицей.
- Concurrent caches — при нескольких Lookup в начале маппинга.
- При cache miss в partial cache — обращение к источнику; минимизировать уникальные ключи при частичном кеше.

---

## 7.2.8. Типичные ошибки

- **Слишком малый cache size:** частые cache miss, обращения к диску и источнику; увеличить размер.
- **Persistent cache при часто меняющемся справочнике:** устаревшие данные; отключить persistent или настроить recache.
- **Unconnected с Dynamic cache:** Unconnected поддерживает только static.
- **Shared cache с несовместимой структурой:** Lookup должны иметь совпадающие Lookup Condition и output ports.

---

## Ключевое

- **Index cache** — ключи; **Data cache** — выходные значения; при переполнении — cache files.
- **Полный кеш** — всё в памяти; **частичный** — часть на диске, при промахе — запрос к источнику.
- **Persistent cache** — сохранение кеша между сессиями; **Recache** — пересборка при изменении источника.
- **Shared cache** — один кеш для нескольких Lookup; unnamed (в маппинге) или named (между маппингами).
- **Concurrent caches** — параллельное построение кешей; ускоряет сессии с несколькими Lookup.

В [§7.3](chapter-07-03.md) мы разберём Joiner: соединение источников по условию (INNER, LEFT, RIGHT, FULL) и настройку join.
