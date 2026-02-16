# §7.3 Joiner: соединение источников

В [§7.2](chapter-07-02.md) мы рассмотрели кеш Lookup. **Joiner** — активная трансформация для соединения данных из двух источников по условию; аналог SQL JOIN. Поддерживает гетерогенные источники (разные БД, flat file). В этом разделе разберём master/detail, типы join (Normal, Master Outer, Detail Outer, Full Outer), Join Condition, sorted input и объединение более двух источников. Сравнение Lookup и Joiner — в [§7.4](chapter-07-04.md). См. [Глоссарий](glossary.md).

---

## 7.3.1. Назначение и структура

**Joiner** — активная трансформация; объединяет данные из двух связанных источников по условию. Поддерживает источники из разных БД и flat file. Источник: [Joiner Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/joiner-transformation-overview.html).

**Master pipeline** — один вход; заканчивается в Joiner.

**Detail pipeline** — второй вход; продолжается к target (данные идут через Joiner).

По умолчанию первый подключённый источник — detail, второй — master; можно поменять в Properties (колонка M на вкладке Ports).

**Ограничения:** нельзя использовать Joiner, если во входном pipeline есть Update Strategy; нельзя подключать Sequence Generator напрямую перед Joiner.

---

## 7.3.2. Типы join

Joiner поддерживает четыре типа. Источник: [Defining the Join Type](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/defining-the-join-type.html).

| Тип в Joiner | Аналог SQL | Результат |
|--------------|------------|-----------|
| **Normal** | INNER JOIN | Только строки с совпадением в обоих источниках |
| **Master Outer** | RIGHT OUTER JOIN | Все строки detail + совпадающие master; отсутствующие master — NULL |
| **Detail Outer** | LEFT OUTER JOIN | Все строки master + совпадающие detail; отсутствующие detail — NULL |
| **Full Outer** | FULL OUTER JOIN | Все строки обоих источников; отсутствующие — NULL |

Normal и Master Outer обычно быстрее Detail Outer и Full Outer.

---

## 7.3.3. Join Condition

**Join Condition** — условие совпадения строк; задаётся парами портов master = detail. Источник: [Defining a Join Condition](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/defining-a-join-condition.html).

**Пример:** `PART_ID1 = PART_ID2` — строки соединяются при равенстве ID.

**Правила:**
- Типы данных в паре должны совпадать; при несовпадении — Expression для приведения.
- Условие только на равенство (`=`); несколько пар объединяются через AND.
- Порядок портов в условии влияет на производительность.
- **NULL не считается совпадением:** NULL = NULL не соединяет строки; при необходимости — NVL в Expression перед Joiner.
- Char и Varchar: Char дополняется пробелами; `Char(40) = 'abcd'` и `Varchar(40) = 'abcd'` не совпадут из‑за trailing spaces.

**Рекомендации по производительности:**
- **Unsorted:** master — источник с меньшим числом строк.
- **Sorted:** master — источник с меньшим числом дубликатов по ключу join.

---

## 7.3.4. Sorted input и merge join

При **sorted input** Integration Service использует merge join: данные уже отсортированы по ключу join, сравнение идёт последовательно без полной загрузки в память. Источник: [Using Sorted Input](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/using-sorted-input.html).

**Настройка:**
1. Сортировать оба входа по ключу join (Sorter или Number of Sorted Ports в Source Qualifier).
2. В Joiner включить Use Sorted Input и указать sort origin ports (порты, по которым выполнена сортировка).
3. Join Condition должен использовать те же порты, что и sort order.

Порядок сортировки в Session должен совпадать с порядком в источниках. При партиционировании партиции должны сохранять порядок сортировки.

---

## 7.3.5. Несколько источников

Для соединения **более двух** источников выход первого Joiner подключается ко второму Joiner; второй источник — к другому входу. Цепочка: Source1 + Source2 → Joiner1 → Joiner2 ← Source3. Источник: [Joiner Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/joiner-transformation-overview.html).

**Один источник:** можно соединять две ветки одного pipeline (например, исходные данные и агрегированные) или два экземпляра одного source. Источник: [Joining Data from a Single Source](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/joiner-transformation/joining-data-from-a-single-source.html).

---

## 7.3.6. Blocking и транзакции

Joiner — **блокирующая** трансформация: для unsorted join должен накопить данные перед соединением. Master pipeline блокируется до завершения чтения detail (или наоборот, в зависимости от типа join). При больших объёмах — использование диска; sorted input снижает потребность в памяти.

---

## 7.3.7. Типичные ошибки

- **Несовпадение типов в Join Condition:** типы портов в паре должны совпадать.
- **NULL в ключе join:** NULL не совпадает с NULL; использовать NVL/default перед Joiner.
- **Char vs Varchar:** trailing spaces в Char; привести к одному типу.
- **Забыть Sorter при sorted input:** при включённом Use Sorted Input данные должны быть отсортированы; иначе неверный результат.
- **Неверный выбор master/detail:** для unsorted — master с меньшим числом строк; иначе избыточная память и медленная работа.

---

## Ключевое

- **Joiner** — соединение двух источников по условию; master и detail pipelines.
- **Normal** = INNER; **Master Outer** = RIGHT; **Detail Outer** = LEFT; **Full Outer** = FULL.
- **Join Condition** — пары master = detail; типы совпадают; NULL не совпадает.
- **Sorted input** — merge join; сортировка по ключу join в обоих входах.
- **Несколько источников** — цепочка Joiner; **один источник** — две ветки или два экземпляра.

В [§7.4](chapter-07-04.md) мы сравним Lookup и Joiner: когда использовать каждый, производительность и ограничения.
