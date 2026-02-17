# §7.4 Lookup vs Joiner

В [§7.1](chapter-07-01.md)–[§7.3](chapter-07-03.md) мы рассмотрели Lookup и Joiner. Обе трансформации обогащают данные из связанного источника, но работают по-разному. В этом разделе — сравнение по функциональности, производительности и ограничениям; когда выбирать Lookup, когда Joiner; альтернатива — join в Source Qualifier. См. [Глоссарий](glossary.md).

---

## 7.4.1. Сравнение по функциональности

| Критерий | Lookup | Joiner |
|----------|--------|--------|
| **Роль** | Обогащение из справочника; один «основной» поток, второй — справочник | Соединение двух равноправных потоков |
| **Условие** | Lookup Condition: `=`, `<`, `>`, `<=`, `>=`, `!=` | Join Condition: только `=` |
| **Типы join** | По умолчанию — левый outer (все строки основного потока); при отсутствии совпадения — default/NULL | INNER, LEFT, RIGHT, FULL |
| **Возврат нескольких строк** | Поддерживается (Return Multiple Rows) | Да; одна строка master может дать несколько строк при нескольких совпадениях в detail |
| **Unconnected** | Поддерживается; вызов через :LKP | Нет |
| **Условный вызов** | Unconnected Lookup вызывается только при TRUE в IIF/DECODE | Joiner всегда в потоке; нельзя «пропустить» |

Lookup позволяет неравные условия (`>`, `<`, `!=`) и условный вызов через Unconnected; Joiner — только равенство, но полный набор SQL-join. Источник: [Tips for Lookup Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/tips-for-lookup-transformations.html), [Tips for Joiner Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/tips-for-joiner-transformations.html).

---

## 7.4.2. Производительность

**Размер справочника:**
- **Малый/средний (до ~300–800 MB):** Lookup с Cached обычно эффективнее; кеш в памяти, минимум обращений к источнику.
- **Большой (свыше ~1–2 GB):** Joiner часто быстрее; не требует полной загрузки справочника в память; при sorted input — merge join с ограниченным кешем.

**Lookup:** при Uncached — запрос к источнику на каждую строку; при Cached — построение кеша и поиск в памяти; при partial cache — промахи ведут к запросам к источнику и диску.

**Joiner:** блокирующая трансформация; при unsorted — накопление данных; при sorted — merge join с меньшим потреблением памяти.

**Рекомендация Informatica:** если справочник и основной источник в **одной БД** и кеширование Lookup нецелесообразно — выполнять join в Source Qualifier (SQL-override или несколько источников в одном SQ). Join в БД обычно быстрее, чем в сессии. Источник: [Tips for Lookup Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/tips-for-lookup-transformations.html), [Tips for Joiner Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/tips-for-joiner-transformations.html).

---

## 7.4.3. Ограничения

**Lookup:**
- Один источник Lookup на трансформацию; для нескольких справочников — несколько Lookup.
- Unconnected возвращает одно значение.
- Uncached при большом объёме — нагрузка на БД.

**Joiner:**
- Только два входа; для трёх и более источников — цепочка Joiner.
- Только условие равенства (`=`).
- Нельзя использовать при Update Strategy или Sequence Generator во входном pipeline.
- Гетерогенные источники (разные БД, flat file) — основная область применения; для одной БД предпочтителен Source Qualifier join.

---

## 7.4.4. Когда Lookup, когда Joiner

**Выбирать Lookup, если:**
- Справочник малый или средний; кеш помещается в память.
- Нужен Unconnected (условный вызов, одно значение из выражения).
- Нужны неравные условия (`>`, `<`, `BETWEEN` и т.п.).
- Один основной поток, второй — справочник для обогащения.
- Проверка существования в target (dynamic cache для SCD).

**Выбирать Joiner, если:**
- Справочник очень большой (сотни MB и выше); Lookup не помещается в кеш.
- Нужны LEFT/RIGHT/FULL outer join с явным управлением.
- Два равноправных потока из разных источников (гетерогенные).
- Оба источника в одном маппинге; join по равенству.

**Выбирать Source Qualifier join, если:**
- Оба источника в **одной БД**.
- Join по равенству; стандартные INNER/LEFT/RIGHT/FULL.
- Нужна максимальная производительность (выполнение в СУБД).

---

## 7.4.5. Сводная таблица

| Сценарий | Рекомендация |
|----------|--------------|
| Справочник < 300 MB, обогащение по ключу | Lookup Cached |
| Справочник > 1 GB, обогащение | Joiner или Source Qualifier join |
| Одна БД, два источника | Source Qualifier join |
| Разные БД / flat file | Joiner |
| Условный вызов (IIF, DECODE) | Unconnected Lookup |
| Неравные условия | Lookup |
| SCD, проверка в target | Lookup с dynamic cache |

---

## Ключевое

- **Lookup** — обогащение из справочника; Unconnected, неравные условия; эффективен при малом/среднем кеше.
- **Joiner** — соединение двух потоков; INNER/LEFT/RIGHT/FULL; эффективен при больших объёмах и sorted input.
- **Source Qualifier join** — при одной БД предпочтительнее Lookup и Joiner по производительности.
- **Размер справочника** — ключевой фактор: малый — Lookup Cached; большой — Joiner или SQ join.

В [§8.1](chapter-08-01.md) мы разберём Aggregator: группировку и агрегатные функции (SUM, AVG, COUNT, MIN, MAX).
