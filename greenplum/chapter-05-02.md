# §5.2 Append-Optimized (AO)

В [§5.1](chapter-05-01.md) мы разобрали heap-таблицы. Для больших фактовых таблиц, которые загружаются пачками и в основном читаются, Greenplum предлагает **append-optimized (AO)** хранение: без per-row информации видимости, с оптимизацией под **массовую вставку** и поддержкой **сжатия**. В этом разделе — что такое AO-таблицы, создание **AO row** (построчное), сжатие (ZLIB, ZSTD), ограничения (UPDATE/DELETE, одиночные INSERT) и отличие от **AO columnar** (колоночное хранение разбирается в [§5.3](chapter-05-03.md)). По [Tanzu Greenplum 7 — Choosing the Table Storage Model](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/admin_guide-ddl-ddl-storage.html).

---

## 5.2.1. Зачем нужны append-optimized таблицы

В heap на каждую строку тратится **информация о версиях** (visibility), нужная для MVCC и UPDATE/DELETE — порядка 20 байт на строку. В больших фактовых таблицах DWH данные обычно **добавляются пачками** и редко обновляются или удаляются по строкам; чтение — в основном аналитические запросы (сканы, агрегации). Для таких таблиц overhead версионирования не нужен и лишь раздувает объём и усложняет страницы.

**Append-optimized (AO)** таблицы не хранят per-row visibility: структура страниц упрощена, объём данных меньше, массовая вставка эффективнее. Модель ориентирована на **добавление данных** (append) и **чтение**; UPDATE/DELETE поддерживаются с ограничениями. См. [Глоссарий](glossary.md).

Итог: большие фактовые таблицы, загружаемые батчами и читаемые аналитическими запросами, целесообразно делать **AO**; небольшие и часто обновляемые — heap ([§5.1](chapter-05-01.md)).

---

## 5.2.2. Создание AO-таблицы (row-oriented)

AO задаётся опцией **appendoptimized=true** (или устаревшим **appendonly=true**; в каталоге хранится appendonly) в секции **WITH** команды **CREATE TABLE**. По умолчанию AO-таблица — **row-oriented** (построчное хранение): строки записываются целиком, как в heap, но без информации видимости. См. [Глоссарий](glossary.md) (Row-oriented).

Пример:

```sql
CREATE TABLE fact_sales (
    sale_id     bigint,
    sale_date   date,
    product_id  int,
    quantity    int,
    amount      numeric(12,2)
) WITH (appendoptimized=true)
  DISTRIBUTED BY (sale_id);
```

Колоночное AO (orientation=column) создаётся с **orientation=column** и разбирается в [§5.3](chapter-05-03.md).

---

## 5.2.3. Сжатие AO-таблиц

Для **append-optimized** таблиц (и row, и column) в Greenplum доступно сжатие на уровне таблицы или столбца. Для **AO row** на уровне таблицы поддерживаются алгоритмы **ZLIB** и **ZSTD**. Сжатие уменьшает объём на диске и часто ускоряет скан за счёт меньшего I/O; с другой стороны, увеличивается нагрузка на CPU при сжатии и распаковке. См. документацию Greenplum.

**Уровни сжатия:** для ZLIB — **compresslevel** от 1 до 9 (1 — быстрее, 9 — сильнее сжатие); для ZSTD — от 1 до 19. Чем выше уровень, тем обычно лучше степень сжатия и больше затраты CPU и времени.

Пример AO row с табличным сжатием ZLIB:

```sql
CREATE TABLE fact_events (
    event_id   bigint,
    ts         timestamptz,
    user_id    int,
    action     text
) WITH (appendoptimized=true, compresstype=zlib, compresslevel=5)
  DISTRIBUTED BY (event_id);
```

Пример с ZSTD:

```sql
CREATE TABLE fact_log (
    id bigint, msg text
) WITH (appendoptimized=true, compresstype=zstd, compresslevel=3)
  DISTRIBUTED BY (id);
```

Не создавайте сжатые AO-таблицы на **сжатых файловых системах**: двойное сжатие даёт лишнюю нагрузку и может ухудшить производительность. См. документацию Greenplum.

---

## 5.2.4. Ограничения AO-таблиц

- **Массовая вставка предпочтительна.** Одиночные **INSERT** по одной строке не запрещены, но не рекомендуются: AO оптимизирован под batch-загрузку (COPY, INSERT ... SELECT, загрузка из внешних таблиц). Много мелких INSERT создают много маленьких segment files и ухудшают последующее чтение.
- **UPDATE и DELETE:** на AO-таблицах в транзакциях **REPEATABLE READ** или **SERIALIZABLE** операции UPDATE и DELETE приводят к преждевременному завершению транзакции (по документации Greenplum). В READ COMMITTED поведение может отличаться; детали и актуальные ограничения см. в руководстве по вашей версии. **DECLARE cursor ... FOR UPDATE** и **триггеры** на append-optimized таблицах не поддерживаются.
- **CLUSTER** на AO разрешён только поверх B-tree индекса.

Если нужны частые UPDATE/DELETE по строкам, используйте **heap** ([§5.1](chapter-05-01.md)) или выносите изменяемые данные в отдельные таблицы.

---

## 5.2.5. AO row и AO columnar

**AO row** (appendoptimized=true без orientation=column) — построчное хранение: все столбцы строки лежат рядом, как в heap, но без версионирования и с возможностью сжатия. Подходит для фактовых таблиц со смешанным доступом (чтение целых строк или многих столбцов) и batch-загрузкой.

**AO columnar** (appendoptimized=true, orientation=column) — хранение по столбцам: значения одного столбца хранятся вместе. Выгодно для аналитических запросов с агрегациями по немногим столбцам и фильтрацией по одному столбцу; поддерживаются дополнительные типы сжатия (в том числе RLE). Подробно — в [§5.3](chapter-05-03.md).

Выбор между AO row и AO columnar зависит от паттернов запросов и загрузки; сводка по выбору типа таблицы — в [§5.4](chapter-05-04.md).

---

## 5.2.6. Метаданные и проверка AO

Тип хранения таблицы (heap, ao_row, ao_column) и опции (compresstype, compresslevel, orientation) хранятся в системном каталоге. Для проверки распределения строк AO-таблицы по сегментам можно использовать **get_ao_distribution(имя_таблицы)**; для оценки степени сжатия — **get_ao_compression_ratio(имя_таблицы)**. См. раздел «Checking the Compression and Distribution of an Append-Optimized Table» в документации Greenplum.

---

## 5.2.7. Типичные ошибки

- **Делать AO таблицу с частыми UPDATE/DELETE:** ограничения по изоляции и отсутствие триггеров/FOR UPDATE; для таких сценариев — heap.
- **Заполнять AO множеством одиночных INSERT:** предпочтительна batch-загрузка (COPY, INSERT ... SELECT, внешние таблицы).
- **Включать сжатие AO на уже сжатой ФС:** избегайте двойного сжатия.

---

## Ключевое

- **Append-optimized (AO)** — модель хранения без per-row visibility; оптимизация под **массовую вставку** и чтение; подходит для больших фактовых таблиц DWH.
- **AO row:** WITH (appendoptimized=true); сжатие **compresstype=zlib|zstd**, **compresslevel**; одиночные INSERT не рекомендуются; UPDATE/DELETE с ограничениями по уровню изоляции.
- **AO row** vs **AO columnar:** row — построчное, column — по столбцам ([§5.3](chapter-05-03.md)); выбор по паттернам запросов и загрузки.

В [§5.3](chapter-05-03.md) мы разберём колоночное хранение (AO Columnar): хранение по столбцам, выгода для аналитики, сжатие и когда его использовать.
