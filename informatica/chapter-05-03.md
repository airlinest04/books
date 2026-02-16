# §5.3 Connected и Unconnected

В [§5.1](chapter-05-01.md) и [§5.2](chapter-05-02.md) мы рассмотрели определение трансформаций и классификацию Active/Passive. **Connected** (подключённая) и **Unconnected** (неподключённая) — классификация по способу подключения к потоку данных: в цепочке или по вызову из выражения. В этом разделе мы разберём различие Connected и Unconnected, какие трансформации поддерживают оба режима, вызов Unconnected через выражение и когда выбирать тот или иной вариант. Подробнее Lookup — в [§7.1](chapter-07-01.md). См. [Глоссарий](glossary.md).

---

## 5.3.1. Connected трансформация

**Connected transformation** (подключённая трансформация) — получает данные **напрямую из потока**: входные порты связаны с выходными портами вышестоящих трансформаций или Source Qualifier. Выходные порты связаны с нижестоящими трансформациями или Target. Источник: [Unconnected Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/unconnected-transformations.html), [Connected and Unconnected Lookups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/connected-and-unconnected-lookups.html).

Большинство трансформаций — Connected. Данные проходят через них последовательно: Source → SQ → Expression → Filter → Target. Каждая строка обрабатывается в порядке потока.

---

## 5.3.2. Unconnected трансформация

**Unconnected transformation** (неподключённая трансформация) — **не связана** с другими трансформациями в потоке. Она **вызывается из выражения** другой трансформации (Expression, Aggregator и др.) и **возвращает одно значение** в точку вызова. Источник: [Unconnected Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/unconnected-transformations.html).

Unconnected трансформация присутствует в маппинге, но не имеет связей (links) с потоком. Вызов выполняется через специальный синтаксис в выражении: для Lookup — `:LKP.LookupName(argument1, argument2, ...)`; для Stored Procedure — `:SP.ProcedureName(...)`.

---

## 5.3.3. Какие трансформации поддерживают Unconnected

По документации PowerCenter поддерживают режим Unconnected: Источник: [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

| Трансформация | Connected | Unconnected |
|---------------|-----------|-------------|
| **Lookup** | Да | Да |
| **Stored Procedure** | Да | Да |
| **External Procedure** | Да | Да |

Остальные трансформации — только Connected. Expression, Filter, Aggregator, Joiner и др. всегда в потоке.

---

## 5.3.4. Lookup: Connected vs Unconnected

Сводная таблица различий. Источник: [Connected and Unconnected Lookups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/connected-and-unconnected-lookups.html).

| Признак | Connected Lookup | Unconnected Lookup |
|---------|------------------|---------------------|
| **Вход** | Напрямую из потока (связанные порты) | Из выражения `:LKP.LookupName(...)` |
| **Кеш** | Dynamic или Static | Только Static |
| **Возврат** | Несколько колонок в несколько портов | Одна колонка в return port |
| **При отсутствии совпадения** | Default value (настраивается) | NULL |
| **Связи** | Lookup/output порты связаны с нижестоящей трансформацией | Return port передаёт значение в порт с выражением :LKP |

**Unconnected Lookup** возвращает **одно значение** на вызов. Для нескольких колонок — несколько Unconnected Lookup или один Connected Lookup. Подробнее — в [§7.1](chapter-07-01.md).

---

## 5.3.5. Вызов Unconnected Lookup через :LKP

Синтаксис вызова Unconnected Lookup в Expression (или Aggregator): Источник: [Connected and Unconnected Lookups](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/lookup-transformation/connected-and-unconnected-lookups.html).

```
:LKP.LookupName(argument1, argument2, ...)
```

`LookupName` — имя Unconnected Lookup в маппинге. Аргументы — значения для lookup condition (поиск по ключу). Результат — значение из return port Lookup. Пример: `IIF(ISNULL(:LKP.LKP_GET_STATUS(ID)), 'NEW', :LKP.LKP_GET_STATUS(ID))` — условный lookup: если нет совпадения, подставить 'NEW'.

---

## 5.3.6. Когда использовать Unconnected

**Unconnected Lookup** полезен, когда:

- **Условный вызов:** Lookup вызывается только при определённом условии (IIF, DECODE); при Connected Lookup выполняется для каждой строки.
- **Несколько вызовов одного Lookup:** один Unconnected Lookup можно вызывать из разных портов Expression с разными аргументами.
- **Одна колонка:** нужен результат только одной колонки; Unconnected возвращает одно значение.

**Connected Lookup** предпочтителен, когда:

- Нужно несколько колонок из одной строки справочника.
- Lookup выполняется для каждой строки без условий.
- Требуется dynamic cache (вставка в кеш при отсутствии совпадения).

---

## 5.3.7. Stored Procedure и External Procedure

**Stored Procedure** и **External Procedure** также поддерживают Unconnected. Вызов — через выражение `:SP.ProcedureName(...)` или аналог. Unconnected Stored Procedure вызывается по требованию из Expression; может использоваться для условного вызова или для получения одного значения. Подробнее — в документации по Stored Procedure.

---

## 5.3.8. Типичные ошибки

- **Связывать Unconnected Lookup с потоком:** Unconnected не имеет связей; вызов только через выражение.
- **Ожидать несколько колонок от Unconnected Lookup:** возвращается одно значение; для нескольких колонок — несколько Unconnected или Connected.
- **Использовать dynamic cache с Unconnected:** Unconnected Lookup — только static cache.
- **Забыть return port:** Unconnected Lookup должен иметь настроенный return port; его значение возвращается в выражение :LKP.

---

## Ключевое

- **Connected** — в потоке данных; вход и выход связаны с другими объектами; большинство трансформаций.
- **Unconnected** — не в потоке; вызывается из выражения; возвращает одно значение. Поддерживают: Lookup, Stored Procedure, External Procedure.
- **Lookup Connected vs Unconnected:** Connected — несколько колонок, dynamic/static cache; Unconnected — одна колонка, static cache, вызов через :LKP.
- **Синтаксис вызова:** `:LKP.LookupName(arg1, arg2, ...)` для Unconnected Lookup.
- **Когда Unconnected:** условный вызов, несколько вызовов одного Lookup, нужна одна колонка.

В [§5.4](chapter-05-04.md) мы разберём порты и выражения: Input, Output, Variable порты; выражения в Expression; IIF, DECODE и другие функции.
