# §4.4 Маппинг-визарды

В [§4.1](chapter-04-01.md)–[§4.3](chapter-04-03.md) мы рассмотрели Mapping, его создание и связывание объектов. **Маппинг-визарды** (Mapping Wizards) — пошаговые помощники Designer для быстрого создания типовых маппингов без ручного добавления и связывания каждого объекта. В этом разделе мы разберём Getting Started Wizard (Pass-through, Slowly growing target), Slowly Changing Dimensions Wizard (Type 1, 2, 3), Mapping Template Wizard и когда использовать визарды вместо ручного создания. См. [Глоссарий](glossary.md).

---

## 4.4.1. Назначение визардов

Designer предоставляет **два основных визарда** для создания маппингов загрузки и поддержки схемы «звезда» (star schema): факты и измерения, связанные с центральной таблицей фактов. Источник: [Understanding the Mapping Wizards](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/understanding-the-mapping-wizards.html).

| Визард | Назначение |
|--------|------------|
| **Getting Started Wizard** | Маппинги для статических и медленно растущих таблиц фактов и измерений. |
| **Slowly Changing Dimensions Wizard** | Маппинги для SCD (медленно меняющихся измерений) с учётом истории. |

Сгенерированные маппинги можно редактировать и дорабатывать. Визарды ускоряют создание типовых сценариев; для нестандартной логики — ручное создание или доработка результата визарда.

---

## 4.4.2. Getting Started Wizard: Pass-through (Copy)

**Pass-through mapping** (маппинг «копирование как есть») — вставляет все строки источника в приёмник без преобразований. Эквивалент ручного маппинга Source → Source Qualifier → Target. Источник: [Creating a Pass-Through Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/creating-a-pass-through-mapping.html), [Steps to Create a Pass-Through Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/creating-a-pass-through-mapping/steps-to-create-a-pass-through-mapping.html).

**Когда использовать:** загрузка статической таблицы фактов или измерения, когда историю хранить не нужно. Перед загрузкой целевая таблица обычно очищается (truncate) или пересоздаётся. Пример: справочник поставщиков, обновляемый раз в год полной перезагрузкой.

**Шаги:**
1. **Mappings → Wizards → Getting Started**
2. Ввести имя маппинга (рекомендуется префикс `m_`), выбрать **Simple Pass Through**, Next
3. Выбрать source definition из списка
4. Ввести имя target (рекомендуется префикс `T_`), Finish
5. При необходимости отредактировать маппинг
6. Создать таблицу в БД перед запуском workflow (или использовать Generate/Execute SQL)

**Важно:** в Session включить опцию **Truncate target table** или выполнять truncate/drop через pre-session command, иначе при повторном запуске возможны дубликаты.

---

## 4.4.3. Getting Started Wizard: Slowly growing target

**Slowly growing target** — загрузка **новых** строк в существующую таблицу без обновления старых. Используется, когда данные только добавляются (insert), а не обновляются. Источник: [Using the Getting Started Wizard](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/understanding-the-mapping-wizards/using-the-getting-started-wizard.html).

| Тип | Тип таблицы | Обработка данных |
|-----|-------------|------------------|
| Pass through | Статическая факт/измерение | Вставка всех строк; truncate перед загрузкой |
| Slowly growing target | Медленно растущая факт/измерение | Вставка только новых строк; существующие не трогаются |

Slowly growing target использует флаги и вставку новых записей; подходит для инкрементальной загрузки без обновления.

---

## 4.4.4. Slowly Changing Dimensions Wizard

**Slowly Changing Dimensions (SCD)** — измерение, атрибуты которого меняются со временем. Визард создаёт маппинги для разных стратегий хранения истории. Источник: [Understanding the Mapping Wizards](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/understanding-the-mapping-wizards.html), [Using the Mapping Wizards](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards.html).

| Тип SCD | Описание |
|---------|----------|
| **Type 1** | Перезапись: изменённые атрибуты обновляются на месте; история не сохраняется |
| **Type 2 (Version Data)** | Новая версия записи при изменении; версионность |
| **Type 2 (Flag Current)** | Флаг «текущая» запись; одна активная версия на ключ |
| **Type 2 (Effective Date Range)** | Диапазон дат действия; от и до |
| **Type 3** | Хранение предыдущего значения в отдельной колонке |

Визард настраивает Lookup, Expression, Router, Update Strategy и другие трансформации по выбранному типу. Подробнее SCD — в книге по DWH ([../dwh/](../dwh/)).

---

## 4.4.5. Mapping Template Wizard

**Mapping Template Wizard** (Import Mapping Template Wizard) — создание маппингов из **предопределённых шаблонов** Informatica. Источник: [Creating a Mapping from the Informatica Mapping Templates](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/creating-a-mapping-from-the-informatica-mapping-templates.html).

**Шаблоны:**
- **Slowly Changing Dimensions** — SCD Type 1, 2, 3
- **Remove Duplicates** — удаление дубликатов
- **Incremental Load** — инкрементальная загрузка

**Запуск:** **Mapping → Mapping Template Wizards** → выбор шаблона. Далее — указание имени маппинга, параметров, создание и импорт в репозиторий.

Шаблоны — это рисунки в формате Mapping Architect for Visio; при импорте генерируются маппинги с заданной структурой. Можно использовать существующий parameter file (Use Existing).

---

## 4.4.6. Доработка сгенерированных маппингов

После работы визарда маппинг можно **редактировать**: добавлять Expression (вычисления), Filter (фильтрацию), Lookup (обогащение), Aggregator (агрегацию) и другие трансформации. Источник: [Creating a Pass-Through Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/using-the-mapping-wizards/creating-a-pass-through-mapping.html).

Пример: Pass-through маппинг создан визардом; вручную добавляем Expression для вычисления поля, Filter для отсечения строк — получаем маппинг с фильтрацией и преобразованиями. Визард даёт базовую структуру; детали — ручная настройка.

---

## 4.4.7. Когда использовать визарды

| Сценарий | Рекомендация |
|----------|--------------|
| Копирование таблицы как есть | Getting Started → Pass-through |
| Инкрементальная вставка без обновления | Getting Started → Slowly growing target |
| SCD Type 1, 2, 3 | Slowly Changing Dimensions Wizard или Mapping Template |
| Удаление дубликатов, инкрементальная загрузка | Mapping Template Wizard |
| Сложная логика, фильтрация, агрегация | Ручное создание или доработка результата визарда |

---

## 4.4.8. Создание target в БД

Визарды не создают таблицу в БД автоматически. После создания маппинга нужно:
- создать target table вручную (DDL);
- или использовать **Targets → Generate/Execute SQL** в Target Designer по target definition.

Для Pass-through в Session часто включают **Truncate target table**; для Slowly growing — нет. См. [§3.2](chapter-03-02.md), [§9.2](chapter-09-02.md).

---

## 4.4.9. Типичные ошибки

- **Запустить Pass-through без truncate:** при повторном запуске — дубликаты. Включить Truncate target table или pre-session truncate.
- **Использовать Pass-through для инкрементальной загрузки:** Pass-through вставляет все строки; для инкремента — Slowly growing или Mapping Template Incremental Load.
- **Не проверить сгенерированный маппинг:** визард может не учесть специфику (типы, колонки). Валидировать и при необходимости править.
- **Забыть создать таблицу в БД:** target definition есть, таблицы нет — ошибка при выполнении Session.

---

## Ключевое

- **Getting Started Wizard:** Pass-through (копирование как есть) и Slowly growing target (инкрементальная вставка).
- **Pass-through:** Mappings → Wizards → Getting Started → Simple Pass Through; выбор source, имя target; truncate перед загрузкой.
- **Slowly Changing Dimensions Wizard:** маппинги для SCD Type 1, 2, 3.
- **Mapping Template Wizard:** шаблоны SCD, Remove Duplicates, Incremental Load; Mapping → Mapping Template Wizards.
- **Доработка:** сгенерированные маппинги можно редактировать (Expression, Filter, Lookup, Aggregator и др.).
- **Target в БД:** создавать вручную или через Generate/Execute SQL; визард не создаёт таблицу.

В [§5.1](chapter-05-01.md) мы переходим к трансформациям: определение, типы, классификация активных и пассивных, connected и unconnected.
