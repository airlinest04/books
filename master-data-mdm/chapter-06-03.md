# §6.3 Разрешение иерархий в запросах

В [§6.1](chapter-06-01.md) и [§6.2](chapter-06-02.md) мы рассмотрели структуру иерархий и множественные разрезы. В этом разделе сосредоточимся на **разрешении иерархий в запросах**: как в SQL обойти дерево, собрать путь, выполнить агрегацию по уровням иерархии (продажи по категориям с учётом подкатегорий). В [§6.4](chapter-06-04.md) разберём связь с DWH и схемой «снежинка».

---

## 6.3.1. Рекурсивные CTE: напоминание

**Рекурсивный CTE** (Common Table Expression) — способ обхода дерева в SQL стандарта 1999. Структура: якорь (начальные строки) + рекурсивная часть (джойн с самим CTE). См. [§6.1](chapter-06-01.md).

Обход вниз (все потомки узла):

```sql
WITH RECURSIVE tree AS (
    SELECT category_id, parent_id, name, 1 AS level
    FROM dim_category WHERE category_id = 1
    UNION ALL
    SELECT c.category_id, c.parent_id, c.name, t.level + 1
    FROM dim_category c
    JOIN tree t ON c.parent_id = t.category_id
)
SELECT * FROM tree;
```

Обход вверх (все предки узла):

```sql
WITH RECURSIVE path AS (
    SELECT category_id, parent_id, name, 1 AS level
    FROM dim_category WHERE category_id = 4
    UNION ALL
    SELECT c.category_id, c.parent_id, c.name, p.level + 1
    FROM dim_category c
    JOIN path p ON c.category_id = p.parent_id
)
SELECT * FROM path;
```

Поддерживается в PostgreSQL, SQL Server, Oracle (с 11g), MySQL 8+, SQLite 3.8+. В §6.3.2–6.3.4 — применение к агрегации.

---

## 6.3.2. Агрегация по иерархии: задача

Типичная задача: **продажи по категориям** с учётом подкатегорий. Продукт привязан к листовой категории; факт продажи — к продукту. Чтобы получить «продажи по родительской категории», нужно:

1. Поднять продукт на уровень родительской категории (продукт → категория → родитель категории).
2. Суммировать факты по категории (включая все дочерние).

Два варианта:

- **Ролл-ап (roll-up):** для каждого узла иерархии посчитать сумму фактов по нему и всем его потомкам. Категория «Сыры» = сумма продаж по «Сыры», «Твёрдые», «Мягкие» и всем продуктам в них.
- **Денормализация:** в факт или витрину заранее загрузить имена/ID категорий верхнего уровня (корневая, уровень 1, 2); агрегация — простой GROUP BY без рекурсии. См. [§6.4](chapter-06-04.md).

---

## 6.3.3. Агрегация через рекурсивный CTE

Ролл-ап «продажи по категориям» (включая подкатегории):

```sql
-- Продукты привязаны к категориям; факт — fact_sales(product_id, amount)
WITH RECURSIVE descendants AS (
    -- Все потомки каждой категории (включая саму категорию)
    SELECT category_id AS ancestor_id, category_id AS descendant_id, 0 AS distance
    FROM dim_category
    UNION ALL
    SELECT d.ancestor_id, c.category_id, d.distance + 1
    FROM dim_category c
    JOIN descendants d ON c.parent_id = d.descendant_id
)
SELECT 
    d.ancestor_id,
    cat.name AS category_name,
    SUM(f.amount) AS total_sales
FROM descendants d
JOIN dim_category cat ON cat.category_id = d.ancestor_id
JOIN dim_product p ON p.category_id = d.descendant_id
JOIN fact_sales f ON f.product_id = p.product_id
GROUP BY d.ancestor_id, cat.name;
```

Идея: CTE `descendants` строит пары (ancestor_id, descendant_id) — для каждой категории перечислены она и все её потомки. Джойн с продуктами по `descendant_id`, с фактом — по `product_id`. Группировка по `ancestor_id` даёт сумму по категории и всем подкатегориям.

Замечание: этот CTE порождает пары для всех узлов; при больших деревьях объём данных растёт. Closure table ([§6.1](chapter-06-01.md)) хранит те же пары материализованно — запрос упрощается, обновление при изменении иерархии сложнее.

---

## 6.3.4. Oracle: CONNECT BY и агрегация

В Oracle иерархию задают через `CONNECT BY`. Агрегация с ролл-апом — через подзапрос или `CONNECT_BY_ROOT`:

```sql
-- Продажи по категориям с учётом потомков (Oracle)
SELECT 
    CONNECT_BY_ROOT cat.category_id AS root_category_id,
    CONNECT_BY_ROOT cat.name AS root_category_name,
    SUM(f.amount) AS total_sales
FROM dim_category cat
JOIN dim_product p ON p.category_id = cat.category_id
JOIN fact_sales f ON f.product_id = p.product_id
START WITH cat.parent_id IS NULL
CONNECT BY cat.parent_id = PRIOR cat.category_id
GROUP BY CONNECT_BY_ROOT cat.category_id, CONNECT_BY_ROOT cat.name;
```

`CONNECT_BY_ROOT` возвращает значение столбца из корневого узла текущей ветки. Здесь для каждой строки категории берётся корень её ветки; группировка по корню даёт сумму по всей ветке.

Для «продажи по заданной категории и подкатегориям» — `START WITH category_id = :id` вместо `parent_id IS NULL`.

---

## 6.3.5. Сбор пути в запросе (breadcrumb)

Часто нужно вывести **путь** от корня до узла, например «Молочные / Сыры / Твёрдые». В рекурсивном CTE путь накапливается при обходе вниз:

```sql
WITH RECURSIVE tree AS (
    SELECT category_id, parent_id, name,
           1 AS level,
           name::TEXT AS path
    FROM dim_category
    WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.parent_id, c.name,
           t.level + 1,
           t.path || ' / ' || c.name
    FROM dim_category c
    JOIN tree t ON c.parent_id = t.category_id
)
SELECT category_id, name, path FROM tree;
```

В PostgreSQL для агрегирования пути от листа к корню удобна `string_agg` с `ORDER BY` в подзапросе, возвращающем предков.

---

## 6.3.6. Фильтрация по уровню иерархии

Иногда нужны только узлы определённого уровня (например, уровень 2 — подкатегории):

```sql
WITH RECURSIVE tree AS (
    SELECT category_id, parent_id, name, 1 AS level
    FROM dim_category WHERE parent_id IS NULL
    UNION ALL
    SELECT c.category_id, c.parent_id, c.name, t.level + 1
    FROM dim_category c
    JOIN tree t ON c.parent_id = t.category_id
)
SELECT * FROM tree WHERE level = 2;
```

Аналогично в Oracle: `WHERE LEVEL = 2` после `CONNECT BY`.

---

## 6.3.7. Производительность и материализация

Рекурсивные CTE при больших деревьях и больших фактах могут быть медленными: рекурсия выполняется при каждом запросе. Варианты оптимизации:

- **Closure table** — материализовать пары (ancestor, descendant); агрегация — джойны без рекурсии. Обновление при изменении иерархии — отдельный процесс. См. [§6.1](chapter-06-01.md).
- **Денормализация в витрину** — при загрузке в DWH писать в строку факта или среза категорию верхнего уровня, уровень 1, 2 и т.д.; отчёты — простой GROUP BY. См. [§6.4](chapter-06-04.md).
- **Индексы** — `(parent_id)`, `(category_id, parent_id)` для ускорения рекурсии.
- **Партиционирование** — для очень больших иерархий рассмотреть разделение по веткам (если модель допускает).

---

## 6.3.8. Типичные ошибки

- **Дублирование при агрегации:** без учёта иерархии джойн «продукт → категория» даёт одну строку на продукт; при джойне с CTE потомков одна строка продукта может повторяться, если продукт ошибочно привязан к нескольким категориям. Проверить целостность: продукт — ровно одна листовая категория.
- **Забыть саму категорию:** ролл-ап должен включать узел и его потомков; в closure table distance = 0 — сам узел.
- **Циклы в данных:** рекурсия зациклится; нужна проверка на циклы при загрузке или ограничение глубины (например, `level <= 10`). См. [§6.1](chapter-06-01.md).
- **Путать направление:** обход вниз — от родителя к детям (parent_id = prior id); обход вверх — наоборот.

---

## Ключевое

- **Разрешение иерархии** — обход дерева в SQL (рекурсивный CTE или CONNECT BY) для получения пути, потомков, предков.
- **Рекурсивный CTE** — якорь + рекурсивная часть; обход вниз и вверх. PostgreSQL, SQL Server, Oracle 11g+, MySQL 8+.
- **Агрегация по иерархии (ролл-ап)** — сумма фактов по категории и всем её подкатегориям; через CTE потомков или closure table.
- **CONNECT BY (Oracle)** — альтернатива; `CONNECT_BY_ROOT` для агрегации по ветке.
- **Путь (breadcrumb)** — конкатенация имён при обходе.
- **Фильтрация по уровню** — условие `level = N` в CTE или `LEVEL = N` в Oracle.
- **Производительность** — closure table, денормализация в витрину, индексы.

В [§6.4](chapter-06-04.md) рассмотрим связь иерархий с DWH: измерения как иерархии, схема «снежинка», денормализация для отчётов.
