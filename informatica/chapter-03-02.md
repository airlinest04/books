# §3.2 Определение приёмников (Targets)

В [§3.1](chapter-03-01.md) мы разобрали определение источников (Sources). **Target** (приёмник) — объект, описывающий структуру данных, в которые Integration Service записывает результат маппинга. В этом разделе мы рассмотрим Target Designer, импорт target из реляционных БД и плоских файлов, создание target из source, генерацию и выполнение SQL для создания таблицы в БД, а также обновление target. Подробнее режимы загрузки (Insert, Update, Bulk) — в [§12.3](chapter-12-03.md). См. [Глоссарий](glossary.md).

---

## 3.2.1. Target definition: что это и зачем

**Target definition** (определение приёмника) — объект репозитория, описывающий **структуру** целевой таблицы или файла: имена колонок (портов), типы данных, ключи (primary key для Update Strategy). Target не содержит данных — только метаданные, по которым Mapping Designer связывает выходные порты трансформаций с целевыми колонками, а Integration Service формирует INSERT/UPDATE/MERGE при выполнении сессии. См. [Глоссарий](glossary.md).

Target definition нужен, чтобы:

- **Mapping Designer** знал, в какие колонки записывать данные и как связывать порты;
- **Integration Service** при выполнении Session создавал корректные SQL-операции или записывал в файл по структуре target;
- при необходимости **создать таблицу в БД** — Designer генерирует DDL по target definition.

Target создаётся один раз и переиспользуется в маппингах. При изменении структуры целевой таблицы target нужно обновить. Источник: [Importing a Target Definition](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-targets/importing-a-target-definition.html).

---

## 3.2.2. Target Designer

**Target Designer** — панель (tool) в Designer для создания и редактирования target definitions. Открывается через меню **Tools → Target Designer** или соответствующую вкладку. В Target Designer отображаются приёмники выбранной папки; здесь выполняют импорт из БД и из файлов, создание target вручную, генерацию SQL. Источник: [Importing Relational Target Definitions](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-targets/importing-a-target-definition/importing-relational-target-definitions.html).

В Navigator под узлом **Targets** отображаются импортированные приёмники. Target definition отображается как прямоугольник с именем таблицы/файла и списком портов.

---

## 3.2.3. Импорт из реляционной БД

Импорт структуры таблицы из реляционной БД выполняется через **Targets → Import from Database**. Источник: [Importing Relational Target Definitions](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-targets/importing-a-target-definition/importing-relational-target-definitions.html).

Пошагово:

1. В Target Designer выбрать **Targets → Import from Database**.
2. Выбрать **ODBC data source** для целевой БД. При необходимости создать или изменить через Browse (ODBC Administrator).
3. Ввести **username** и **password** для подключения.
4. При необходимости указать **owner name**.
5. Нажать **Connect**.
6. Развернуть список объектов, найти нужные таблицы.
7. Выбрать одну или несколько таблиц (Shift — блок, Ctrl — отдельные; Select All / Select None).
8. Нажать **OK**.

Импортированные target definitions появляются в Target Designer и в Navigator под узлом Targets. Используется, когда целевая таблица **уже существует** в БД (например, слой Staging или витрина, созданная DBA). Структура target должна соответствовать структуре реальной таблицы, иначе при загрузке возможны ошибки (несовпадение типов, отсутствие колонок).

---

## 3.2.4. Импорт плоских файлов

Импорт структуры плоского файла выполняется через **Targets → Import from File**. Процесс аналогичен импорту Source из файла: выбор файла, code page, delimited или fixed-width, настройка разделителей. Target definition для файла описывает колонки и типы; при выполнении Session путь к файлу задаётся в настройках (или через parameter `$OutputFileName`).

Flat file target подходит для выгрузки данных в CSV, TXT и т.п. Подробнее форматы — в [§3.4](chapter-03-04.md).

---

## 3.2.5. Создание target из source

Часто целевая таблица должна иметь ту же структуру, что и источник (или подмножество колонок). В Designer можно **скопировать source в target**: создать target definition на основе существующего source. Меню: **Targets → Create from Source** (или аналог по версии). Выбирается source; создаётся target с теми же портами. При необходимости колонки можно удалить или переименовать.

Это удобно для сценария «копирование таблицы как есть» или для Staging-слоя с минимальными изменениями. См. [§4.4](chapter-04-04.md) (Copy wizard).

---

## 3.2.6. Создание target вручную

Target можно создать **вручную** без импорта: **Targets → Create**. Задаются имя, порты (колонки) с именами и типами. Для реляционного target можно указать primary key — это потребуется для Update Strategy при обновлении по ключу. Ручное создание полезно, когда:

- таблица ещё не создана в БД, и структура проектируется в Designer;
- целевая структура отличается от любой существующей таблицы;
- нужна только часть колонок источника.

---

## 3.2.7. Генерация и выполнение SQL (создание таблицы в БД)

После добавления **реляционного** target definition в репозиторий можно сгенерировать и выполнить DDL для создания таблицы в БД. Источник: [Creating a Target Table](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/working-with-targets/creating-a-target-table.html).

Пошагово:

1. В Target Designer выбрать target definition (или несколько).
2. Меню **Targets → Generate/Execute SQL**.
3. Нажать **Connect**, выбрать БД (ODBC data source), подключиться.
4. Указать **имя и путь** для SQL-файла (скрипт сохраняется на локальный диск).
5. Выбрать опции генерации (primary keys, индексы и т.д.).
6. **Generate SQL File** — только создать скрипт; **Generate and Execute** — создать и выполнить.

Designer генерирует CREATE TABLE (и при необходимости DROP TABLE) по структуре target. SQL записывается в текстовый файл; его можно отредактировать перед выполнением. Для flat file и XML target генерация SQL недоступна — таблица в БД не создаётся. Создание таблицы выполняется Designer при подключении к БД; для production часто предпочитают выполнять DDL вручную или через миграционные скрипты, а не из Designer.

---

## 3.2.8. Target и Connection: разделение ответственности

Как и для Source, **Target definition** и **Connection** — разные объекты:

| Объект | Где создаётся | Что хранит |
|--------|---------------|------------|
| **Target** | Designer (Target Designer) | Структура: колонки, типы, ключи. Метаданные для маппинга. |
| **Connection** | Workflow Manager | Параметры доступа к целевой БД. Используется Integration Service при записи. |

В Session для реляционного target указывается Connection; для flat file target — путь к файлу (Connection type: None). Target не содержит сведений о сервере или базе; Connection не знает о структуре таблицы. См. [§2.3](chapter-02-03.md), [§1.4](chapter-01-04.md).

---

## 3.2.9. Обновление target при изменении структуры

Если структура целевой таблицы изменилась (добавлены колонки, изменены типы), target definition нужно обновить. Варианты:

- **Re-import:** Targets → Import from Database для той же таблицы; Replace при конфликте.
- **Ручное редактирование:** добавить или изменить порты в Target Designer.

После обновления маппинги, использующие этот target, могут перейти в Impacted или Invalid. Нужно обновить связи портов в маппинге. При добавлении колонок в таблицу — добавить порты в target и в маппинг; при удалении — убрать связи. См. [§1.3.3](chapter-01-03.md).

---

## 3.2.10. Режимы загрузки: кратко

Target definition описывает **структуру**; режим загрузки (Insert, Update, Delete, Bulk) задаётся в **Session**, а не в target. В target при необходимости указывается **primary key** — он используется для Update Strategy (обновление по ключу). Подробнее настройки Session и режимы загрузки — в [§9.2](chapter-09-02.md), [§12.3](chapter-12-03.md).

---

## 3.2.11. Типичные ошибки при определении приёмников

- **Путать Target и Connection:** Target — структура; Connection — параметры доступа. Оба нужны для реляционного приёмника при выполнении.
- **Импорт из неверной таблицы или схемы:** проверить owner и имя таблицы; импорт из dev-таблицы при загрузке в prod приведёт к несоответствию, если структуры отличаются.
- **Несоответствие структуры target и реальной таблицы:** target должен отражать актуальную структуру; иначе ошибки при INSERT/UPDATE (лишние колонки, несовпадение типов).
- **Забыть primary key для Update:** при использовании Update Strategy в Session нужен primary key в target; иначе обновление по ключу не сработает.
- **Генерировать SQL в production без проверки:** скрипт DDL может содержать DROP; выполнять в production только после проверки и approval. Предпочтительно — генерировать файл, проверять, выполнять вручную или через миграционный процесс.

---

## Ключевое

- **Target definition** — метаданные структуры приёмника (колонки, типы, ключи); не содержит данных. Создаётся в Target Designer, хранится в папке репозитория.
- **Target Designer** — инструмент Designer для импорта и редактирования targets. Меню: Targets → Import from Database, Import from File, Create, Create from Source, Generate/Execute SQL.
- **Импорт из БД:** Targets → Import from Database; ODBC data source, выбор таблиц. Для существующих таблиц.
- **Импорт из файла:** Targets → Import from File; структура flat file.
- **Создание из source:** Targets → Create from Source — копирование структуры source в target.
- **Генерация SQL:** Targets → Generate/Execute SQL — создание таблицы в БД по target definition. Только для реляционных targets.
- **Target vs Connection:** Target — структура (Designer); Connection — параметры доступа (Workflow Manager). Режим загрузки (Insert/Update) — в Session.
- **Обновление:** при изменении таблицы — re-import или ручное редактирование target; проверить маппинги.

В [§3.3](chapter-03-03.md) мы разберём коннекторы и типы данных: встроенные коннекторы, маппинг типов между источниками и Informatica, преобразование типов.
