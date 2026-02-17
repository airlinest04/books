# §4.2 Создание маппинга

В [§4.1](chapter-04-01.md) мы разобрали, что такое Mapping. В этом разделе — пошаговое создание маппинга: открытие Mapping Designer, добавление источников и приёмников, трансформаций, связывание портов, валидация и сохранение. Подробнее разветвлённые потоки и несколько источников — в [§4.3](chapter-04-03.md); визарды для быстрого создания — в [§4.4](chapter-04-04.md). См. [Глоссарий](glossary.md).

---

## 4.2.1. Подготовка: Source и Target

Перед созданием маппинга необходимо иметь **Source definition** и **Target definition** в папке репозитория. Источник: [Developing a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/mappings-overview/developing-a-mapping.html).

- **Source:** импортировать через Source Analyzer (Sources → Import from Database или Import from File) или создать вручную. См. [§3.1](chapter-03-01.md).
- **Target:** импортировать через Target Designer (Targets → Import from Database или Import from File) или создать вручную, Create from Source. См. [§3.2](chapter-03-02.md).

При необходимости создать переиспользуемые трансформации в Transformation Developer или Mapplets в Mapplet Designer. Можно создавать трансформации и в процессе разработки маппинга.

---

## 4.2.2. Создание маппинга

**Способ 1:** Меню **Mappings → Create** (или **Mapping → Create** по версии). Откроется пустое полотно Mapping Designer; введите имя маппинга. Рекомендуемое соглашение имён: префикс `m_` (например, `m_LoadStaging`).

**Способ 2:** Перетащить (drag) Source, Target, Mapplet или переиспользуемую трансформацию из Navigator в рабочую область Mapping Designer. Designer создаст новый маппинг и добавит перетащенный объект. Источник: [Developing a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/mappings-overview/developing-a-mapping.html).

Mapping Designer открывается через **Tools → Mapping Designer** или соответствующую вкладку. В Navigator под узлом **Mappings** отображаются маппинги папки.

---

## 4.2.3. Добавление источников

Развернуть узел **Sources** в Navigator и перетащить нужный source definition в рабочую область. При перетаскивании добавляется **instance** (экземпляр) source в маппинг. Источник: [Working with Sources in a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/working-with-sources-in-a-mapping.html).

**Source Qualifier:** при добавлении реляционного или flat file source Designer **автоматически** создаёт **Source Qualifier** и связывает его с source. Source Qualifier определяет, как Integration Service читает данные (по умолчанию — `SELECT * FROM table` или чтение файла). Для COBOL — Normalizer; для XML — XML Source Qualifier; для application sources — Application Source Qualifier.

**Отключение автосоздания:** если нужно объединить данные из нескольких relational sources в один Source Qualifier (join на стороне источника), автосоздание можно отключить в настройках Designer; затем создать Source Qualifier вручную и подключить к нему несколько источников. См. [§6.1](chapter-06-01.md).

**Свойства instance:** двойной щелчок по source в маппинге → вкладка Properties. Для relational source можно указать table owner; для flat file — datetime format, thousands separator, decimal separator.

---

## 4.2.4. Добавление приёмников

Развернуть узел **Targets** в Navigator и перетащить target definition в рабочую область. Target добавляется как instance; колонки target отображаются как порты. Источник: [Developing a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/mappings-overview/developing-a-mapping.html).

Колонки в target **нельзя копировать** в маппинге; структуру target меняют в Target Designer. В маппинге только **связывают** выходные порты трансформаций с портами target.

---

## 4.2.5. Добавление трансформаций

**Из палитры:** в Mapping Designer доступна палитра трансформаций (Transformations). Выбрать нужную (Expression, Filter, Lookup, Joiner, Aggregator и др.) и перетащить на полотно или создать через меню **Transformation → Create**.

**Переиспользуемые трансформации:** из Transformation Developer — перетащить в маппинг; создаётся instance. Изменения в переиспользуемой трансформации наследуются всеми instance в маппингах.

**Логика трансформации:** двойной щелчок по трансформации → вкладки для настройки портов, выражений, условий. Подробнее — в главах 5–8.

---

## 4.2.6. Связывание портов

Порты связывают для задания потока данных: выходной порт одного объекта — с входным портом другого. Источник: [Linking Ports](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/linking-ports.html).

**Ручное связывание:** перетащить линию от выходного порта (Output) к входному (Input). Принимающий порт должен быть Input или Input/Output; исходящий — Output или Input/Output. Типы данных должны быть совместимы.

**Автоматическое связывание:**
- **By position** (по позиции): порты связываются по порядку (первый с первым, второй со вторым и т.д.).
- **By name** (по имени): порты с совпадающими именами связываются. Можно указать prefix или suffix (например, порт `Name` в Source Qualifier и `FilName` в Filter при prefix `Fil`).

**Source → Source Qualifier:** source связывается **только** с Source Qualifier. Source Qualifier → Target или другие трансформации.

---

## 4.2.7. Минимальный маппинг (pass-through)

Простейший маппинг «копирование как есть»:

1. Добавить Source (перетащить из Navigator).
2. Designer создаёт Source Qualifier и связывает с source.
3. Добавить Target (перетащить из Navigator).
4. Связать порты Source Qualifier с портами Target: вручную или автоматически (by name, если имена совпадают).

Результат: данные читаются из источника и записываются в приёмник без преобразований. Такой маппинг называют pass-through. Визард Copy — в [§4.4](chapter-04-04.md).

---

## 4.2.8. Валидация маппинга

**Validate:** перед сохранением или запуском Session маппинг нужно проверить. Меню **Mapping → Validate** (или аналог). Designer проверяет связи, типы данных, обязательные настройки трансформаций. Источник: [Developing a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/mappings-overview/developing-a-mapping.html).

Ошибки отображаются в окне **Output**. Маппинг с ошибками — **Invalid**; Session против него запустить нельзя до исправления.

---

## 4.2.9. Сохранение

**Save:** **Mapping → Save** или Ctrl+S. При сохранении Designer выполняет валидацию; сообщения — в Output. Маппинг сохраняется в репозиторий в текущей папке.

**View Dependencies:** правый щелчок по маппингу в Navigator → **View Dependencies** (или Mappings → View Dependencies). Показывает, какие Session или shortcuts затронуты изменениями маппинга. Источник: [Editing a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/working-with-mappings/editing-a-mapping.html).

---

## 4.2.10. Редактирование маппинга

После создания маппинг можно редактировать: добавлять, изменять, удалять объекты. При удалении Designer показывает список объектов к удалению. Изменения в source/target definition (в Source Analyzer, Target Designer) наследуются всеми instance в маппингах; некоторые изменения могут сделать маппинг Invalid. См. [§3.1.8](chapter-03-01.md), [§3.2.9](chapter-03-02.md).

---

## 4.2.11. Типичные ошибки при создании

- **Забыть связать Source Qualifier с Target:** маппинг Invalid; данные не попадут в приёмник.
- **Не связать все обязательные порты target:** некоторые колонки останутся NULL или вызовут ошибку.
- **Связать несовместимые типы:** Designer не даст связать; при смене типа после связывания — перепроверить.
- **Добавить target без связей:** target должен получать данные от трансформации или Source Qualifier.
- **Не валидировать перед сохранением:** ошибки обнаруживаются при Save, но лучше проверять явно через Validate.

---

## Ключевое

- **Подготовка:** Source и Target definitions в папке; при необходимости — Mapplets, переиспользуемые трансформации.
- **Создание:** Mappings → Create или drag Source/Target в workspace.
- **Добавление Source:** drag из Navigator; Designer автоматически создаёт Source Qualifier и связывает с source.
- **Добавление Target:** drag из Navigator; связывать порты с выходом трансформаций или Source Qualifier.
- **Трансформации:** из палитры или Transformation Developer; настройка — двойной щелчок.
- **Связывание портов:** вручную (drag) или автоматически (by position, by name). Source — только с Source Qualifier.
- **Валидация и сохранение:** Validate перед Save; Invalid — нельзя запустить Session.
- **Pass-through:** Source → Source Qualifier → Target без трансформаций.

В [§4.3](chapter-04-03.md) мы разберём связывание источников, трансформаций и целей: линейные и разветвлённые потоки, несколько источников и несколько приёмников.
