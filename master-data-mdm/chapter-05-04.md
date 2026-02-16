# §5.4 Инструменты и SQL-паттерны

В [§5.1](chapter-05-01.md)–[§5.3](chapter-05-03.md) мы разобрали обнаружение дубликатов, сравнение и слияние. Завершая главу 5, рассмотрим **инструменты и SQL-паттерны** для дедупликации: как реализовать её в SQL (GROUP BY, оконные функции), как интегрировать с ETL-инструментами серии (Informatica, Datagram) и какие ограничения при отсутствии облачных MDM-сервисов. Книга фокусируется на on-prem и SQL; MDM-платформы и облачные сервисы — вне охвата. В [§6.1](chapter-06-01.md) начнём главу 6 — иерархии и связи.

---

## 5.4.1. Дедупликация в SQL: обзор

SQL позволяет реализовать часть логики дедупликации непосредственно в базе:

- **Обнаружение групп** — GROUP BY по ключевому полю (ИНН и т.д.); HAVING COUNT(*) > 1.
- **Выбор «победителя» в группе** — оконные функции (ROW_NUMBER, FIRST_VALUE) по приоритету.
- **Формирование золотой записи** — агрегация (MAX, MIN по правилам) или выбор одной строки из группы.
- **Fuzzy matching** — в PostgreSQL с модулем fuzzystrmatch (levenshtein, jarowinkler); в других СУБД — через пользовательские функции или внешние скрипты.

Ограничения SQL: попарное сравнение с fuzzy matching для больших объёмов в чистом SQL неэффективно; блокировку и сложные комбинированные правила удобнее реализовывать в ETL (Python, Java) или в скриптах. SQL хорошо подходит для простых случаев (точное совпадение по ключу) и для финального «выбора одной записи из группы» после того, как группы определены. См. [§5.1](chapter-05-01.md).

---

## 5.4.2. GROUP BY для обнаружения дубликатов

Группировка по ключевому полю выявляет дубликаты:

```sql
-- Группы по ИНН (кандидаты на слияние)
SELECT inn, COUNT(*) AS cnt, 
       array_agg(source_system || ':' || source_id) AS sources
FROM staging_counterparties
WHERE inn IS NOT NULL AND TRIM(inn) != ''
GROUP BY inn
HAVING COUNT(*) > 1;

-- Список записей-дубликатов с деталями
SELECT c.*, 
       COUNT(*) OVER (PARTITION BY c.inn) AS group_size
FROM staging_counterparties c
WHERE inn IN (
    SELECT inn FROM staging_counterparties 
    WHERE inn IS NOT NULL 
    GROUP BY inn HAVING COUNT(*) > 1
);
```

Для блокировки по префиксу — GROUP BY по производному ключу:

```sql
-- Блокировка по первым 4 символам нормализованного названия
SELECT LEFT(LOWER(TRIM(name)), 4) AS block_key,
       COUNT(*) AS cnt
FROM staging_counterparties
GROUP BY LEFT(LOWER(TRIM(name)), 4)
HAVING COUNT(*) > 1;
```

---

## 5.4.3. Оконные функции для выбора «победителя»

Оконные функции позволяют в каждой группе пронумеровать строки по приоритету и выбрать одну:

```sql
-- Выбор одной записи из группы по приоритету источника
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (
               PARTITION BY inn 
               ORDER BY 
                   CASE source_system 
                       WHEN 'ERP' THEN 1 
                       WHEN 'CRM' THEN 2 
                       WHEN 'BILLING' THEN 3 
                       ELSE 4 
                   END,
                   updated_at DESC NULLS LAST
           ) AS rn
    FROM staging_counterparties
    WHERE inn IS NOT NULL
)
SELECT * FROM ranked WHERE rn = 1;
```

Результат — по одной записи на группу (по ИНН); приоритет — ERP, затем CRM, затем биллинг; при равенстве — самая свежая по updated_at. Аналогично для выбора по «самому полному» — ORDER BY LENGTH(name) DESC и т.д.

---

## 5.4.4. Агрегация для merge по полям

Для простого merge по полям (брать MAX/MIN по приоритету) можно использовать агрегацию:

```sql
-- Упрощённо: для каждой группы взять MAX по приоритету
-- (работает, если приоритет можно закодировать в сортировку)
SELECT inn,
       MAX(name) KEEP (DENSE_RANK FIRST ORDER BY source_priority, updated_at DESC) AS name,
       MAX(phone) KEEP (DENSE_RANK FIRST ORDER BY source_priority, updated_at DESC) AS phone
FROM (
    SELECT *, 
           CASE source_system WHEN 'ERP' THEN 1 WHEN 'CRM' THEN 2 ELSE 3 END AS source_priority
    FROM staging_counterparties
) t
GROUP BY inn;
```

Синтаксис KEEP (DENSE_RANK FIRST ORDER BY ...) поддерживается в Oracle; в PostgreSQL используют подзапрос с ROW_NUMBER и условную агрегацию или распределённую логику в CTE. Универсальный подход — оконные функции + фильтр rn=1, как в примере выше; для merge по полям с разными приоритетами по атрибутам нужна более сложная логика (отдельный ROW_NUMBER по полям или несколько проходов). См. [§4.2](chapter-04-02.md).

---

## 5.4.5. Fuzzy matching в PostgreSQL

Модуль `fuzzystrmatch`:

```sql
CREATE EXTENSION IF NOT EXISTS fuzzystrmatch;

-- Сравнение двух строк
SELECT levenshtein('ООО Ромашка', 'Ромашка ООО');
SELECT 1.0 - levenshtein('ООО Ромашка', 'Ромашка ООО')::float 
       / GREATEST(length('ООО Ромашка'), length('Ромашка ООО'));

-- Поиск похожих в таблице (медленно при больших объёмах!)
SELECT a.id, b.id, 
       1.0 - levenshtein(a.name, b.name)::float / 
       GREATEST(length(a.name), length(b.name)) AS similarity
FROM staging_counterparties a
JOIN staging_counterparties b ON a.id < b.id
WHERE 1.0 - levenshtein(a.name, b.name)::float / 
      GREATEST(length(a.name), length(b.name)) >= 0.85;
```

Для больших таблиц такой cross join неприемлем; нужна блокировка (сравнивать только внутри блоков) или вынос fuzzy matching в ETL/скрипт. В ETL (Python) — библиотеки `rapidfuzz`, `jellyfish`; пакетная обработка с блокировкой. См. [§5.2](chapter-05-02.md).

---

## 5.4.6. Связь с ETL-инструментами

ETL-платформы (Informatica, Talend, Neoflex Datagram, dbt и др.) могут выполнять дедупликацию:

- **Source Qualifier / преобразования:** дедупликация по ключу в Source Qualifier; агрегация в Aggregator.
- **Специализированные трансформации:** Match в Informatica MDM (если используется); в PowerCenter — Router, Filter, Expression для реализации правил.
- **Вызов внешних скриптов:** Stored Procedure, Command Task — вызов Python/Java для fuzzy matching и сложной логики.
- **SQL-объекты:** выполнение подготовленных SQL (обнаружение групп, вставка золотых записей) через SQL-трансформацию или подключение к БД.

Типичный пайплайн: загрузка в Staging → (опционально) скрипт/Python для fuzzy matching и формирования групп → SQL или трансформации для merge по правилам → загрузка в Core (золотые записи) и таблицу маппинга. См. [§11.3](chapter-11-03.md), книга «Informatica», книга «Neoflex Datagram».

---

## 5.4.7. Без облачных MDM-сервисов

Книга не охватывает облачные MDM-сервисы (Informatica MDM Cloud, SAP MDG, Microsoft Purview и т.д.). Фокус — on-prem решения:

- **SQL и хранимые процедуры** в целевой СУБД (PostgreSQL, Oracle, Greenplum).
- **ETL-инструменты серии** — Informatica PowerCenter, Neoflex Datagram, Talend open source.
- **Скрипты** (Python, Java) для fuzzy matching и сложной логики; вызов из ETL или по расписанию.

Подход «ручной»: проектирование правил, реализация в SQL/ETL, поддержка и доработка. MDM-платформы дают готовые функции сопоставления и слияния, но требуют лицензий и интеграции; в рамках книги достаточен уровень SQL + ETL. См. [TOC](TOC.md), введение книги.

---

## 5.4.8. Пример: полный цикл дедупликации в SQL

Упрощённый сценарий для контрагентов по ИНН:

```sql
-- 1. Группы дубликатов
WITH groups AS (
    SELECT inn FROM staging_counterparties
    WHERE inn IS NOT NULL AND TRIM(inn) != ''
    GROUP BY inn HAVING COUNT(*) > 1
),
-- 2. Выбор «победителя» в каждой группе
ranked AS (
    SELECT c.*, 
           ROW_NUMBER() OVER (
               PARTITION BY c.inn 
               ORDER BY CASE c.source_system 
                   WHEN 'ERP' THEN 1 WHEN 'CRM' THEN 2 ELSE 3 END,
                   c.updated_at DESC
           ) AS rn
    FROM staging_counterparties c
    JOIN groups g ON c.inn = g.inn
),
-- 3. Золотые записи (одна на группу)
golden AS (
    INSERT INTO core_counterparty (golden_id, inn, name, phone, address, ...)
    SELECT nextval('golden_seq'), inn, name, phone, address, ...
    FROM ranked WHERE rn = 1
    RETURNING golden_id, inn
)
-- 4. Маппинг (упрощённо: только для «победителей»; полный маппинг — отдельно)
SELECT * FROM golden;
```

Полный маппинг всех записей группы на golden_id требует дополнительного INSERT из всех строк ranked (не только rn=1) с соответствующим golden_id. Merge по полям из разных записей — более сложный запрос или несколько шагов. См. [§5.3](chapter-05-03.md).

---

## 5.4.9. Типичные ошибки

- **Делать cross join для fuzzy matching на больших таблицах:** O(N²); использовать блокировку или вынос в ETL.
- **Забыть про транзитивное замыкание:** A–B и B–C дубликаты → группа A–B–C; простой GROUP BY по одному полю это не покроет.
- **Считать, что ROW_NUMBER даёт merge по полям:** ROW_NUMBER выбирает целую строку; для merge по полям с разными приоритетами по атрибутам нужна отдельная логика по полям.
- **Не логировать результат:** при автоматической дедупликации полезно сохранять, сколько групп найдено, сколько записей слито, для мониторинга.

---

## Ключевое

- **SQL для дедупликации:** GROUP BY для обнаружения групп; ROW_NUMBER для выбора «победителя»; агрегация — для простых случаев.
- **Оконные функции** — ROW_NUMBER, DENSE_RANK; ORDER BY по приоритету источника и дате.
- **Fuzzy matching в PostgreSQL** — fuzzystrmatch (levenshtein, jarowinkler); для больших объёмов — блокировка или ETL.
- **ETL-инструменты** — Informatica, Datagram, Talend; трансформации, вызов скриптов для сложной логики.
- **Без облачных MDM** — on-prem SQL + ETL; проектирование правил и реализация вручную.
- **Полный цикл** — группы → ранжирование → выбор золотой → маппинг; merge по полям — более сложная реализация.

В [§6.1](chapter-06-01.md) мы начнём главу 6 — иерархии и связи: дерево, parent_id, обход вверх/вниз.
