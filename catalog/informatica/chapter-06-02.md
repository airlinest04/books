# §6.2 Expression

В [§6.1](chapter-06-01.md) мы рассмотрели Source Qualifier. **Expression** — пассивная трансформация для вычислений на уровне одной строки: арифметика, конкатенация, преобразование типов, условная логика. Не меняет количество строк; для агрегаций по нескольким строкам используется Aggregator (см. [§8.1](chapter-08-01.md)). В этом разделе разберём порты Expression (Input, Output, Variable), Expression Editor, типичные функции и сценарии использования. IIF и DECODE подробно описаны в [§5.4](chapter-05-04.md). См. [Глоссарий](glossary.md).

---

## 6.2.1. Назначение и тип

**Expression** — пассивная, подключённая, нативная трансформация. Вычисляет значения для каждой строки независимо; не агрегирует данные. Источник: [Expression Transformation Overview](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/expression-transformation/expression-transformation-overview.html).

**Типичные сценарии:**

- Расчёт производных полей: цена × количество, скидка, итоговая сумма.
- Конкатенация: объединение имени и фамилии, формирование кода.
- Преобразование типов: строка в число, дата в строку.
- Условная логика: выбор значения по условию (IIF, DECODE).
- Проверка условий перед передачей в Filter или Router.

**Expression vs Aggregator:** Expression — по строке; Aggregator — по группе строк (SUM, AVG, COUNT и т.д.).

---

## 6.2.2. Порты Expression

Expression поддерживает Input, Output, Variable и Input/Output порты. Источник: [Configuring Ports](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/expression-transformation/configuring-ports.html), [Calculating Values](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/expression-transformation/configuring-ports/calculating-values.html).

| Тип порта | Назначение |
|-----------|------------|
| **Input** | Получает данные из вышестоящей трансформации или Source Qualifier; не содержит выражения. |
| **Input/Output** | Передаёт данные без изменения (pass-through); можно использовать в выражениях других портов. |
| **Output** | Результат выражения; выражение задаётся в Expression Editor. |
| **Variable** | Промежуточный расчёт; не передаётся наружу; используется в других Variable или Output. |

**Порядок вычисления:** Input → Variable → Output (см. [§5.4.2](chapter-05-04.md)). Variable может ссылаться на Input и другие Variable; Output — на Input и Variable.

**Пример:** расчёт общей суммы заказа. Input: `unit_price`, `quantity`. Output: `total_amount` с выражением `unit_price * quantity`. При необходимости промежуточного расчёта (например, скидка) — Variable `discount_amount`, затем Output `final_amount` = `(unit_price * quantity) - discount_amount`.

---

## 6.2.3. Expression Editor и функции

**Expression Editor** — диалог для ввода выражений в Output и Variable портах. Поддерживает Transformation Language: SQL-подобные функции, операторы, литералы. Источник: [Working with Expressions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/working-with-expressions.html).

**Часто используемые функции:**

| Категория | Функции | Пример |
|-----------|---------|--------|
| Условные | IIF, DECODE | `IIF( AMOUNT > 100, 'HIGH', 'LOW' )` |
| Строковые | SUBSTR, CONCAT, LTRIM, RTRIM, UPPER, LOWER | `CONCAT( FIRST_NAME, ' ', LAST_NAME )` |
| Преобразование типов | TO_CHAR, TO_DATE, TO_DECIMAL, TO_INTEGER | `TO_CHAR( ORDER_DATE, 'YYYY-MM-DD' )` |
| NULL-обработка | ISNULL, NVL, NULLIF | `NVL( PHONE, 'N/A' )` |
| Математические | ABS, ROUND, CEIL, FLOOR | `ROUND( AMOUNT, 2 )` |

Полный список — в [Transformation Language Reference](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-language-reference/functions.html).

**Mapping parameters:** `$$ParameterName` в выражениях; значение задаётся при выполнении Session.

---

## 6.2.4. Условная логика

Условная логика в Expression реализуется через **IIF** и **DECODE** (см. [§5.4.4](chapter-05-04.md), [§5.4.5](chapter-05-04.md)).

**IIF** — выбор из двух значений:

```
IIF( condition, value_if_true [, value_if_false] )
```

**DECODE** — поиск по значению или множественные условия с `DECODE( TRUE, ... )`:

```
DECODE( STATUS, 'A', 1, 'B', 2, 'C', 3, 0 )
DECODE( TRUE, AMOUNT < 50, 'LOW', AMOUNT < 200, 'MED', 'HIGH' )
```

Expression может вычислять флаги и коды для последующей передачи в Filter, Router или Update Strategy.

---

## 6.2.5. Pass-through и несколько Output

**Pass-through:** Output порт без выражения просто передаёт значение Input. Удобно, когда нужно сохранить исходное поле и добавить вычисляемое — Input/Output для pass-through, Output для расчёта.

**Несколько Output портов:** одна трансформация может содержать несколько Output с разными выражениями. Пример: из зарплаты сотрудника вычисляются федеральный налог, местный налог, взносы — каждый в отдельном Output; общие Input — зарплата и категория удержания.

---

## 6.2.6. Типичные ошибки

- **Variable ссылается на Output:** Variable может ссылаться только на Input и Variable; иначе ошибка валидации.
- **Неправильный порядок Variable:** Variable B использует Variable A — A должен быть выше B в списке портов.
- **Смешение типов в выражении:** результат выражения должен соответствовать типу порта; при несовпадении — явное преобразование (TO_DECIMAL, TO_CHAR и т.д.).
- **Деление на ноль:** `IIF( QTY = 0, 0, AMOUNT / QTY )` — проверка перед делением.
- **NULL в арифметике:** NULL + число = NULL; использовать NVL при необходимости.

---

## Ключевое

- **Expression** — пассивная трансформация для вычислений на уровне строки; не агрегирует.
- **Порты:** Input, Input/Output (pass-through), Output (с выражением), Variable (промежуточный расчёт).
- **Порядок:** Input → Variable → Output; Variable и Output не могут ссылаться на Output.
- **Функции:** IIF, DECODE, CONCAT, SUBSTR, TO_*, NVL, ROUND и др. — Transformation Language.
- **Несколько Output** — один Expression может вычислять несколько полей из общих Input.

В [§6.3](chapter-06-03.md) мы разберём Filter и Router: фильтрацию по условию и ветвление потока на несколько групп.
