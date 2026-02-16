# §8.3 Normalizer и другие

В [§8.2](chapter-08-02.md) мы рассмотрели Rank и Distinct. **Normalizer** — активная трансформация для разворота данных (unpivot): одна строка с несколькими повторяющимися колонками превращается в несколько строк. В этом разделе — Normalizer (VSAM и Pipeline), Sequence Generator, Stored Procedure, Update Strategy и краткий обзор Custom, Java, SQL. Подробнее Update Strategy — в [§12.3](chapter-12-03.md). См. [Глоссарий](glossary.md).

---

## 8.3.1. Normalizer: назначение

**Normalizer** — активная трансформация; получает строку с несколькими повторяющимися колонками (multiple-occurring columns) и возвращает по одной строке на каждое вхождение. Источник: [Normalizer Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/normalizer-transformation/normalizer-transformation-overview.html).

**Пример:** таблица с продажами по кварталам (Store, Q1, Q2, Q3, Q4):

| Store  | Q1  | Q2  | Q3  | Q4  |
|--------|-----|-----|-----|-----|
| Store1 | 100 | 300 | 500 | 700 |
| Store2 | 250 | 450 | 650 | 850 |

Normalizer разворачивает в:

| Store  | Sales | Quarter |
|--------|-------|---------|
| Store1 | 100   | 1       |
| Store1 | 300   | 2       |
| Store1 | 500   | 3       |
| Store1 | 700   | 4       |
| Store2 | 250   | 1       |
| …      | …     | …       |

Integration Service генерирует ключ для каждой исходной строки; при нескольких вхождениях все выходные строки получают один и тот же ключ. Источник: [Pipeline Normalizer Transformation](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/normalizer-transformation/pipeline-normalizer-transformation.html).

---

## 8.3.2. VSAM и Pipeline Normalizer

| Тип | Назначение | Создание |
|-----|------------|----------|
| **VSAM Normalizer** | Source Qualifier для COBOL-источника; колонки из COBOL, read-only | Автоматически при добавлении COBOL source в маппинг |
| **Pipeline Normalizer** | Разворот данных из relational или flat file | Вручную в Transformation Developer или Mapping Designer |

VSAM Normalizer обрабатывает multiple-occurring колонки и REDEFINES из COBOL. Pipeline Normalizer — для relational и flat file; колонки задаются вручную. Источник: [Normalizer Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/normalizer-transformation/normalizer-transformation-overview.html).

**Pipeline Normalizer:** для каждой повторяющейся колонки — отдельный input port; выход — по одной строке на каждое вхождение; GCID (Generated Column ID) — индекс вхождения (1, 2, 3, 4 для кварталов). Источник: [Pipeline Normalizer Transformation](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/normalizer-transformation/pipeline-normalizer-transformation.html).

---

## 8.3.3. Sequence Generator

**Sequence Generator** — пассивная трансформация; генерирует числовые значения. Используется для первичных ключей, замены отсутствующих ключей, циклического перебора диапазона. Источник: [Sequence Generator Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/sequence-generator-transformation/sequence-generator-transformation-overview.html).

**Порты:** NEXTVAL и CURRVAL (output only). Источник: [Sequence Generator Ports](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/sequence-generator-transformation/sequence-generator-ports.html).

| Порт | Назначение |
|------|------------|
| **NEXTVAL** | Следующее значение последовательности; при подключении Integration Service генерирует блок чисел |
| **CURRVAL** | Текущее значение (NEXTVAL + Increment By); при подключении — одно значение на блок |

**Reusable:** переиспользуемый Sequence Generator сохраняет целостность последовательности при параллельных сессиях в один target; разные instance — риск дубликатов ключей. Источник: [Sequence Generator Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/sequence-generator-transformation/sequence-generator-transformation-overview.html).

---

## 8.3.4. Stored Procedure

**Stored Procedure** — пассивная трансформация; вызывает хранимую процедуру в БД (Transact-SQL, PL/SQL и др.). Источник: [Stored Procedure Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/stored-procedure-transformation/stored-procedure-transformation-overview.html).

**Сценарии:**
- Проверка состояния target перед загрузкой.
- Проверка свободного места в БД.
- Специализированные расчёты.
- Управление индексами (drop/recreate).

Процедура должна существовать в БД до создания трансформации; может быть в source, target или любой БД с connection. Connected — в потоке; Unconnected — вызов из выражения (:SP). Источник: [Stored Procedure Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/stored-procedure-transformation/stored-procedure-transformation-overview.html).

---

## 8.3.5. Update Strategy

**Update Strategy** — активная трансформация; помечает строки для Insert, Update, Delete или Reject. Источник: [Update Strategy Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/update-strategy-transformation/update-strategy-transformation-overview.html).

**Константы и значения:** Источник: [Flagging Rows Within a Mapping](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/update-strategy-transformation/flagging-rows-within-a-mapping.html).

| Операция | Константа | Число |
|----------|-----------|-------|
| Insert   | DD_INSERT | 0     |
| Update   | DD_UPDATE | 1    |
| Delete   | DD_DELETE | 2     |
| Reject   | DD_REJECT | 3     |

Выражение в Update Strategy возвращает одну из констант; любое другое значение трактуется как Insert. Стратегия задаётся на уровне Session и на уровне маппинга. Подробнее — в [§12.3](chapter-12-03.md).

---

## 8.3.6. Custom, Java, SQL: краткий обзор

| Трансформация | Тип | Назначение |
|---------------|-----|------------|
| **Custom** | Active/Passive, Connected, Non-native | Вызов процедуры в DLL/shared library |
| **Java** | Active/Passive, Connected, Non-native | Пользовательская логика на Java; bytecode в репозитории |
| **SQL** | Active/Passive, Connected, Non-native | Выполнение SQL-запросов к БД |

Custom и Java — для сложной логики, не реализуемой стандартными трансформациями. SQL — для произвольных запросов в рамках маппинга. Источник: [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

---

## 8.3.7. Типичные ошибки

- **Normalizer без учёта generated key:** ключ связывает строки одной исходной записи; при последующей агрегации или join учитывать его.
- **Sequence Generator: дубликаты при параллельных сессиях:** использовать reusable Sequence Generator для всех сессий в один target.
- **Update Strategy без настройки Session:** стратегия в маппинге применяется только при включённой опции в Session (Treat source rows as).
- **Stored Procedure в неподдерживаемой БД:** не все СУБД поддерживают stored procedures; синтаксис различается.

---

## Ключевое

- **Normalizer** — разворот multiple-occurring колонок в строки; VSAM (COBOL) и Pipeline (relational/flat file); generated key, GCID.
- **Sequence Generator** — NEXTVAL, CURRVAL; reusable для уникальных ключей при параллельных сессиях.
- **Stored Procedure** — вызов процедуры в БД; Connected или Unconnected.
- **Update Strategy** — DD_INSERT (0), DD_UPDATE (1), DD_DELETE (2), DD_REJECT (3).
- **Custom, Java, SQL** — Non-native; расширяют возможности маппинга.

В [§8.4](chapter-08-04.md) мы разберём Mapplets и повторное использование трансформаций.
