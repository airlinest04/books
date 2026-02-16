# §6.3 Filter и Router

В [§6.2](chapter-06-02.md) мы рассмотрели Expression. **Filter** и **Router** — активные трансформации, отбирающие строки по условию. Filter проверяет одно условие и пропускает только строки, где оно TRUE; остальные отбрасывает. Router проверяет несколько условий и направляет строки в разные выходные группы (каждая группа — своя ветка потока). В этом разделе разберём Filter Condition, группы Router, Default Group и когда выбирать Filter, а когда Router. См. [Глоссарий](glossary.md).

---

## 6.3.1. Filter: назначение и условие

**Filter** — активная трансформация; уменьшает количество строк. Пропускает строки, для которых условие возвращает TRUE; строки с FALSE отбрасывает и записывает сообщение в session log. Источник: [Filter Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/filter-transformation/filter-transformation-overview.html).

**Filter Condition** — выражение, возвращающее TRUE или FALSE. Задаётся в Properties → Filter Condition через Expression Editor. Источник: [Filter Condition](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/filter-transformation/filter-condition.html).

**Примеры:**
```
SALARY > 30000
SALARY > 30000 AND SALARY < 100000
STATUS = 'ACTIVE'
ORDER_DATE >= $$StartDate
```

**Правила:**
- Условие чувствительно к регистру.
- NULL в условии трактуется как FALSE — строка отбрасывается.
- Число 0 — эквивалент FALSE; любое ненулевое — TRUE.
- Можно использовать AND, OR; mapping parameters (`$$Param`).

**Ограничение:** входные порты Filter должны приходить из **одной** трансформации. Нельзя объединять порты из нескольких источников в один Filter — для этого сначала Joiner или Union.

---

## 6.3.2. Filter: размещение и производительность

Filter рекомендуется размещать **как можно ближе к источникам**, чтобы не передавать отбрасываемые строки через последующие трансформации. Альтернатива — фильтрация на уровне источника (Source Filter или SQL-override в Source Qualifier, см. [§6.1](chapter-06-01.md)), когда условие выражается в SQL.

---

## 6.3.3. Router: назначение и группы

**Router** — активная трансформация; проверяет данные по **нескольким** условиям и направляет строки в разные выходные группы. Источник: [Router Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/router-transformation/router-transformation-overview.html).

**Структура Router:**
- **Input group** — входные порты.
- **User-defined groups** — группы с условием (Group Filter Condition); каждая группа — отдельный выход.
- **Default group** — группа без условия; получает строки, не попавшие ни в одну user-defined группу.

**Group Filter Condition** — выражение TRUE/FALSE для каждой строки. Строка передаётся в группу, если условие TRUE. Источник: [Using Group Filter Conditions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/router-transformation/working-with-groups/using-group-filter-conditions.html).

**Важно:** одна строка может попасть в **несколько** групп, если удовлетворяет нескольким условиям. Пример: `employee_salary > 1000` (Group 1) и `employee_salary > 2000` (Group 2); при salary=3000 строка идёт в обе группы.

---

## 6.3.4. Router vs несколько Filter

При проверке одних и тех же данных по **нескольким** условиям Router эффективнее нескольких Filter: Integration Service обрабатывает входящие данные **один раз**. Три Filter подряд — три прохода; один Router с тремя группами — один проход. Источник: [Router Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/router-transformation/router-transformation-overview.html).

| Сценарий | Рекомендация |
|----------|---------------|
| Одно условие, одна ветка | Filter |
| Несколько условий, несколько веток (разные targets/трансформации) | Router |
| Одно условие, отбросить остальное | Filter |
| Несколько условий + «прочие» в default | Router |

---

## 6.3.5. Default group в Router

**Default group** — получает строки, для которых **все** user-defined условия вернули FALSE. Если default group не подключена к трансформации или target, эти строки отбрасываются. Источник: [Using Group Filter Conditions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/router-transformation/working-with-groups/using-group-filter-conditions.html).

Пример: группы France, Japan, USA с условиями по стране; default — остальные шесть стран. Строки из France идут только в группу France; строки из Germany — в default.

---

## 6.3.6. Типичные ошибки

- **Filter с портами из разных трансформаций:** вход Filter — только из одной вышестоящей трансформации.
- **Взаимоисключающие условия в Router:** если нужна жёсткая маршрутизация «строго в одну группу», условия должны быть взаимоисключающими (например, `STATUS='A'`, `STATUS='B'`, `STATUS='C'`); иначе одна строка может попасть в несколько групп.
- **NULL в условии Filter/Router:** NULL трактуется как FALSE; для явной обработки NULL использовать `ISNULL(port)` или `NVL(port, default)` в условии.
- **Забыть подключить default group:** строки, не попавшие в user-defined группы, идут в default; если default не подключена — они отбрасываются (может быть намеренно).

---

## Ключевое

- **Filter** — одно условие; пропускает TRUE, отбрасывает FALSE; активная трансформация.
- **Router** — несколько групп с условиями; Default group — строки, не попавшие в user-defined группы.
- **Одна строка может попасть в несколько групп Router**, если удовлетворяет нескольким условиям.
- **Router эффективнее нескольких Filter** при проверке одних данных по разным условиям — один проход вместо нескольких.
- **Filter:** вход только из одной трансформации; размещать ближе к источникам.

В [§6.4](chapter-06-04.md) мы разберём Sorter и Union: сортировку потока и объединение нескольких потоков в один.
