# §7.1 Lookup: назначение и типы

В [§6.4](chapter-06-04.md) мы рассмотрели Sorter и Union. **Lookup** — трансформация для обогащения данных из справочника: по значению в потоке ищется соответствующая строка в таблице или файле и возвращаются связанные поля. Lookup может быть активной или пассивной; поддерживает **Connected** и **Unconnected** режимы, **Cached** и **Uncached**. В этом разделе разберём назначение Lookup, типы источников, Lookup Condition, Connected vs Unconnected и Cached vs Uncached. Кеш и производительность — в [§7.2](chapter-07-02.md). См. [Глоссарий](glossary.md).

---

## 7.1.1. Назначение Lookup

**Lookup** — трансформация для поиска данных в справочнике (relational table, view, flat file) по условию. Integration Service выполняет запрос к источнику Lookup на основе входных портов и Lookup Condition; возвращает найденные значения. Источник: [Lookup Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/lookup-transformation-overview.html).

**Типичные сценарии:**
- **Обогащение:** источник содержит `customer_id`; Lookup возвращает `customer_name` из справочника клиентов.
- **Расчёт:** Lookup возвращает ставку налога по региону; Expression вычисляет сумму налога.
- **Проверка существования:** Lookup по target table для SCD или определения insert/update.
- **Множественные значения:** Lookup может возвращать несколько строк (например, все сотрудники отдела).

---

## 7.1.2. Источники Lookup

Lookup может использовать: relational table/view/synonym, flat file, source qualifier (pipeline lookup). Источник: [Lookup Source Types](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/lookup-source-types.html).

Для relational — импорт через ODBC; для flat file — Flat File Wizard. PowerCenter Client и Integration Service должны иметь доступ к источнику Lookup.

---

## 7.1.3. Lookup Condition и порты

**Lookup Condition** — условие поиска, аналог WHERE в SQL. Сравнивает значения входных портов с колонками источника Lookup. Источник: [Lookup Condition](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/lookup-condition.html).

**Пример:** источник содержит `employee_number`; Lookup — таблица с `employee_ID`, `first_name`, `last_name`. Условие: `employee_ID = employee_number`. Lookup возвращает `first_name`, `last_name` для каждой строки.

**Правила:**
- Типы данных в условии должны совпадать.
- Lookup Condition обязателен во всех Lookup.
- Несколько условий объединяются через AND.
- Порядок условий для оптимизации: `=` → `<`, `>`, `<=`, `>=` → `!=`.
- NULL = NULL считается совпадением.

**Порты Lookup:**
- **Lookup ports** — колонки из источника, участвующие в условии.
- **Input ports** — получают значения из потока; связываются с Lookup ports в условии.
- **Output ports** — возвращают значения из источника (колонки, не входящие в условие, или все нужные).

---

## 7.1.4. Connected vs Unconnected

**Connected Lookup** — входные и выходные порты связаны с другими трансформациями в потоке. Получает данные напрямую из pipeline; возвращает несколько колонок в ту же строку. Источник: [Connected and Unconnected Lookups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/connected-and-unconnected-lookups.html).

**Unconnected Lookup** — не связан с потоком; вызывается из выражения `:LKP.LookupName( arg1, arg2, ... )` в Expression, Filter, Router, Aggregator и др. Возвращает **одно** значение через return port. Источник: [Connected and Unconnected Lookups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/connected-and-unconnected-lookups.html).

| Критерий | Connected | Unconnected |
|----------|-----------|-------------|
| Связь с потоком | Input/Output порты связаны | Вызов через :LKP в выражении |
| Возврат | Несколько колонок | Одна колонка (return port) |
| Кеш | Static или Dynamic | Только Static |
| При отсутствии совпадения | Default value (настраивается) | NULL |
| User-defined default | Поддерживается | Не поддерживается |

**Когда Unconnected:** Lookup нужен условно (в ветке IIF, DECODE); один Lookup вызывается из нескольких мест; возвращается одно значение. См. [§5.3](chapter-05-03.md).

---

## 7.1.5. Cached vs Uncached

**Cached Lookup** — Integration Service при первом запросе загружает данные источника в кеш (память и/или файлы); последующие строки ищутся в кеше без обращения к БД. По умолчанию кеш static — не обновляется во время сессии. Источник: [Lookup Caches](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/lookup-caches.html).

**Uncached Lookup** — для каждой строки выполняется запрос к источнику; кеш не используется. Подходит для больших справочников, когда кеш не помещается в память, или при частых изменениях данных во время сессии.

| Режим | Обращение к источнику | Производительность | Применение |
|-------|----------------------|--------------------|------------|
| **Cached** | Один раз при построении кеша | Высокая при повторных совпадениях | Справочники малого/среднего размера |
| **Uncached** | На каждую строку | Низкая при большом объёме | Большие справочники, редкие совпадения |

Подробнее о полном/частичном кеше, persistent cache и настройках — в [§7.2](chapter-07-02.md).

---

## 7.1.6. Active vs Passive

Lookup может быть **активной** или **пассивной** в зависимости от настроек. При возврате одной строки на входную — пассивная (количество строк не меняется). При возврате нескольких строк (Return Multiple Rows) — активная: одна входная строка может породить несколько выходных.

---

## 7.1.7. Типичные ошибки

- **Забыть Lookup Condition:** Lookup без условия невалиден.
- **Несовпадение типов в условии:** типы Input и Lookup портов должны совпадать.
- **Unconnected без return port:** Unconnected Lookup возвращает одно значение; должен быть настроен return port.
- **Unconnected с Dynamic cache:** Unconnected поддерживает только static cache.
- **Слишком большой Uncached Lookup:** при большом объёме данных Uncached создаёт нагрузку на БД; рассмотреть Cached или Joiner.

---

## Ключевое

- **Lookup** — обогащение данными из справочника (таблица, файл) по Lookup Condition.
- **Connected** — в потоке, несколько выходных колонок; **Unconnected** — вызов через :LKP, одна колонка.
- **Cached** — кеш в памяти; **Uncached** — запрос к источнику на каждую строку.
- **Lookup Condition** — обязателен; типы должны совпадать; условия объединяются через AND.
- **Источники:** relational, flat file, source qualifier.

В [§7.2](chapter-07-02.md) мы разберём кеш Lookup: полный и частичный кеш, persistent cache и настройки производительности.
