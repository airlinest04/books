# §12.3 Маппинг: загрузка

В [§12.2](chapter-12-02.md) мы рассмотрели извлечение и преобразование. **Маппинг загрузки** — настройка Target и Session для записи данных: режим вставки/обновления, bulk load, Update Strategy. В этом разделе — Target Load Type (Normal/Bulk), опции Insert/Update/Delete, Update Strategy в маппинге, Treat source rows as (Data Driven) и Truncate. Подробнее Workflow и проверка — в [§12.4](chapter-12-04.md). См. [Глоссарий](glossary.md).

---

## 12.3.1. Target и Session: связь

Target definition описывает структуру целевой таблицы; Session настраивает **как** записывать: connection, Target Load Type, Insert/Update/Delete, Truncate. Настройки — Session → Mapping tab → Transformations view → выбрать target под узлом Targets; или Properties tab → General Options. Источник: [Target Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/targets/working-with-relational-targets/target-properties.html).

---

## 12.3.2. Target Load Type: Normal и Bulk

| Режим | Описание |
|-------|----------|
| **Normal** | Стандартные операции БД (INSERT, UPDATE); логирование; recovery возможен. Выбирать при Update Strategy в маппинге. |
| **Bulk** | Вызов bulk-утилиты БД; обход лога; выше производительность; recovery ограничен. Поддерживается для DB2, Sybase, Oracle, Microsoft SQL Server. |

При Bulk для других типов БД Integration Service переключается на Normal. При **data driven** (Update Strategy) и Bulk — Integration Service переключается на Normal. Источник: [Bulk Loading](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/targets/working-with-relational-targets/bulk-loading.html), [Target Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/targets/working-with-relational-targets/target-properties.html).

**Когда Bulk:** только INSERT; большие объёмы; допустимо ограничение recovery. Для Staging с полной перезаписью — типичный сценарий. Источник: [Bulk Loading](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/targets/working-with-relational-targets/bulk-loading.html).

---

## 12.3.3. Insert, Update, Delete

| Опция | Описание | По умолчанию |
|-------|----------|--------------|
| **Insert** | Вставка строк, помеченных как insert | Включено |
| **Update (as Update)** | UPDATE строк, помеченных как update | Включено |
| **Update (as Insert)** | INSERT вместо UPDATE для помеченных update | Выключено |
| **Update (else Insert)** | UPDATE если строка есть; иначе INSERT | Выключено |
| **Delete** | Удаление строк, помеченных как delete | Выключено |
| **Truncate Table** | TRUNCATE target перед загрузкой | Выключено |

Источник: [Target Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/targets/working-with-relational-targets/target-properties.html).

**Truncate Table** — полная перезапись; очистка перед загрузкой. Для идемпотентной полной загрузки в Staging — Truncate + Insert. См. [§12.1.3](chapter-12-01.md).

---

## 12.3.4. Update Strategy в маппинге

**Update Strategy** — трансформация, помечающая строки для Insert (DD_INSERT), Update (DD_UPDATE), Delete (DD_DELETE) или Reject (DD_REJECT). См. [§8.3.5](chapter-08-03.md).

**Выражение:** `IIF(condition, DD_UPDATE, DD_INSERT)` или `DECODE(status, 'NEW', DD_INSERT, 'CHG', DD_UPDATE, 'DEL', DD_DELETE, DD_REJECT)`.

**Логика:** Lookup по target; сравнение с потоком; определение новой/изменённой/удалённой строки; выражение возвращает константу. Для SCD Type 2 — Router или Expression с Lookup по target. См. [§7.1](chapter-07-01.md), [§12.1.4](chapter-12-01.md).

**Session:** для применения Update Strategy необходимо установить **Treat source rows as = Data Driven**. Иначе Integration Service игнорирует пометки и вставляет все строки. Источник: [Setting the Update Strategy](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/update-strategy-transformation/update-strategy-transformation-overview/setting-the-update-strategy.html).

---

## 12.3.5. Treat source rows as

**Treat source rows as** — свойство Session (Properties tab или Config Object), определяющее, как обрабатывать строки. Источник: [Defining the Treat Source Rows As Property](https://docs.informatica.com/data-integration/powercenter/10-4-0/workflow-basics-guide/sources/working-with-relational-sources/defining-the-treat-source-rows-as-property.html).

| Значение | Описание |
|----------|----------|
| **Insert** | Все строки — INSERT |
| **Update** | Все строки — UPDATE (по ключу target) |
| **Delete** | Все строки — DELETE |
| **Data Driven** | Операция определяется Update Strategy в маппинге |

При Update Strategy — обязательно **Data Driven**. При Insert-only (без Update Strategy) — Insert. Источник: [Setting the Update Strategy](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/update-strategy-transformation/update-strategy-transformation-overview/setting-the-update-strategy.html).

---

## 12.3.6. Reject files

**Reject File Directory** и **Reject Filename** — куда писать строки, отклонённые при загрузке (constraint violation, тип данных). По умолчанию — `$PMBadFileDir`, имя `target_name.bad`. Можно использовать session parameter `$BadFileName`. Источник: [Target Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/targets/working-with-relational-targets/target-properties.html).

---

## 12.3.7. Типичные ошибки

- **Update Strategy без Data Driven:** пометки игнорируются; все строки вставляются; установить Treat source rows as = Data Driven.
- **Bulk при Update Strategy:** Integration Service переключается на Normal; Bulk не применяется.
- **Delete без включённой опции:** строки с DD_DELETE не удаляются; включить Delete в target properties.
- **Truncate при инкрементальной загрузке:** Truncate очищает target; для инкрементальной — отключить.

---

## Ключевое

- **Target Load Type:** Normal (INSERT/UPDATE, recovery) или Bulk (только INSERT, выше производительность); при Update Strategy — Normal.
- **Insert, Update, Delete, Truncate** — опции в target properties; Truncate для полной перезаписи.
- **Update Strategy** — DD_INSERT, DD_UPDATE, DD_DELETE, DD_REJECT; логика через Lookup по target.
- **Treat source rows as = Data Driven** — обязательно при Update Strategy.
- **Reject files** — каталог и имя для отклонённых строк.

В [§12.4](chapter-12-04.md) мы разберём Workflow и проверку: Session, Schedule, Monitor, сверка данных.
