# §5.1 Трансформация: определение и типы

В [§4.1](chapter-04-01.md)–[§4.4](chapter-04-04.md) мы рассмотрели Mapping и его создание. **Трансформация** (Transformation) — объект, реализующий этап Transform в ETL: генерирует, изменяет или передаёт данные. В этом разделе мы дадим определение трансформации, три оси классификации (Active/Passive, Connected/Unconnected, Native/Non-native), краткий обзор основных трансформаций и места их создания. Подробнее Active/Passive — в [§5.2](chapter-05-02.md), Connected/Unconnected — в [§5.3](chapter-05-03.md), порты и выражения — в [§5.4](chapter-05-04.md). См. [Глоссарий](glossary.md).

---

## 5.1.1. Определение трансформации

**Трансформация** (Transformation) — объект репозитория, который **генерирует, изменяет или передаёт** данные. Integration Service использует настроенную в трансформации логику для преобразования данных при выполнении Session. Источник: [Transformations Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview.html), [Working with Transformations in a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mappings/working-with-transformations-in-a-mapping.html).

Данные проходят через **порты** (ports) трансформации, связанные в маппинге с другими объектами. Трансформации в маппинге задают операции, которые Integration Service выполняет над данными: фильтрация, вычисления, объединение, агрегация и т.д.

---

## 5.1.2. Три оси классификации

Трансформации классифицируются по трём признакам. Источник: [Transformations Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview.html).

| Ось | Варианты | Кратко |
|-----|----------|--------|
| **Active / Passive** | Active, Passive | Active — меняет количество строк, границы транзакций или тип строк; Passive — не меняет |
| **Connected / Unconnected** | Connected, Unconnected | Connected — в потоке данных; Unconnected — вызывается по выражению, возвращает одно значение |
| **Native / Non-native** | Native, Non-native | Native — встроенные в Designer; Non-native — Custom, Java, SQL, Union и др. |

Одна трансформация может сочетать признаки: например, Lookup — Active или Passive, Connected или Unconnected, Native. Подробнее — в [§5.2](chapter-05-02.md), [§5.3](chapter-05-03.md).

---

## 5.1.3. Основные трансформации: краткий обзор

Сводная таблица (по документации PowerCenter 10.5). Источник: [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

| Трансформация | Active/Passive | Connected/Unconnected | Native/Non-native | Назначение |
|---------------|----------------|----------------------|-------------------|------------|
| **Aggregator** | Active | Connected | Native | Агрегация (SUM, AVG, COUNT, GROUP BY) |
| **Expression** | Passive | Connected | Native | Вычисления, формулы, IIF, DECODE |
| **Filter** | Active | Connected | Native | Фильтрация по условию |
| **Router** | Active | Connected | Native | Маршрутизация по группам условий |
| **Source Qualifier** | Active | Connected | Native | Чтение из relational/flat file source |
| **Lookup** | Active или Passive | Connected или Unconnected | Native | Поиск в таблице/файле по ключу |
| **Joiner** | Active | Connected | Native | Join двух источников |
| **Union** | Active | Connected | Non-native | Объединение потоков (UNION ALL) |
| **Sorter** | Active | Connected | Native | Сортировка |
| **Rank** | Active | Connected | Native | Ограничение по топ-N |
| **Sequence Generator** | Passive | Connected | Native | Генерация первичных ключей |
| **Update Strategy** | Active | Connected | Native | Insert/Update/Delete/Reject |
| **Stored Procedure** | Passive | Connected или Unconnected | Native | Вызов хранимой процедуры |
| **Normalizer** | Active | Connected | Native | Нормализация (COBOL, разворот) |
| **Custom** | Active или Passive | Connected | Non-native | Вызов процедуры в DLL/shared library |
| **Java** | Active или Passive | Connected | Non-native | Пользовательская логика на Java |
| **SQL** | Active или Passive | Connected | Non-native | Выполнение SQL-запросов |

Полный список — в [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

---

## 5.1.4. Native и Non-native

**Native transformations** — встроенные трансформации Designer (Aggregator, Expression, Filter, Lookup, Joiner, Source Qualifier и др.). Реализованы в движке PowerCenter; поддерживают pushdown при совместимости с СУБД. Источник: [Native and Non-native Transformations](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/native-and-non-native-transformations.html).

**Non-native transformations** — создаются через Custom transformation (вызов внешней процедуры) или предоставляются Designer как предустановленные типы: Java, SQL, Union. К ним применяются правила Custom transformation. Union, несмотря на встроенность, классифицируется как Non-native.

---

## 5.1.5. Создание трансформаций

Трансформации создают в трёх местах. Источник: [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

| Место | Назначение |
|-------|------------|
| **Mapping Designer** | Трансформация для использования в одном маппинге (single-use) |
| **Mapplet Designer** | Трансформации внутри Mapplet; Input и Output — только в Mapplet |
| **Transformation Developer** | Переиспользуемая трансформация для нескольких маппингов |

При добавлении переиспользуемой трансформации в маппинг создаётся **instance** (экземпляр). Изменения в исходной трансформации в Transformation Developer наследуются всеми instance; некоторые изменения могут сделать маппинги Invalid.

---

## 5.1.6. Связь с этапом Transform в ETL

Трансформации реализуют этап **Transform** в ETL: очистка, маппинг полей, обогащение, агрегация, валидация. См. [§1.4](chapter-01-04.md).

| Вид преобразования | Трансформации |
|--------------------|---------------|
| Фильтрация | Filter, Router |
| Вычисление, условная логика | Expression |
| Обогащение по справочнику | Lookup |
| Объединение потоков | Joiner, Union |
| Агрегация | Aggregator |
| Сортировка, дедупликация | Sorter, Rank, Distinct |
| Определение режима записи | Update Strategy |

---

## 5.1.7. Типичные ошибки

- **Путать Active и Passive при связывании:** несколько Active в один вход — нельзя; см. [§4.3](chapter-04-03.md).
- **Считать, что все трансформации — Connected:** Lookup и Stored Procedure могут быть Unconnected; вызываются по выражению.
- **Не различать Native и Non-native:** Non-native (Java, SQL, Custom) могут иметь ограничения по pushdown и производительности.
- **Редактировать instance вместо reusable:** изменения в instance не распространяются; для переиспользования — редактировать в Transformation Developer.

---

## Ключевое

- **Трансформация** — объект репозитория, генерирующий, изменяющий или передающий данные; Integration Service выполняет логику при Session.
- **Три оси:** Active/Passive (меняет ли количество строк), Connected/Unconnected (в потоке или по вызову), Native/Non-native (встроенная или Custom/Java/SQL).
- **Основные:** Aggregator, Expression, Filter, Router, Source Qualifier, Lookup, Joiner, Union, Sorter, Rank, Update Strategy и др.
- **Native** — встроенные; **Non-native** — Custom, Java, SQL, Union.
- **Создание:** Mapping Designer (single-use), Mapplet Designer, Transformation Developer (reusable).
- **Transform в ETL** — реализуется через трансформации в маппинге.

В [§5.2](chapter-05-02.md) мы разберём активные и пассивные трансформации: как они влияют на количество строк и на правила связывания в маппинге.
