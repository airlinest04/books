# §5.4 Порты и выражения

В [§5.1](chapter-05-01.md)–[§5.3](chapter-05-03.md) мы рассмотрели типы трансформаций. **Порты** (ports) — входы и выходы трансформации, через которые данные передаются и связываются в маппинге. **Выражения** (expressions) — формулы и функции, задающие логику вычислений. В этом разделе мы разберём типы портов (Input, Output, Variable), порядок их вычисления, выражения в Expression и других трансформациях, функции IIF и DECODE, а также использование mapping parameters. Подробнее Expression — в [§6.2](chapter-06-02.md). См. [Глоссарий](glossary.md).

---

## 5.4.1. Типы портов

Трансформации используют три типа портов. Источник: [Port Order](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/local-variables/guidelines-for-configuring-variable-ports/port-order.html).

| Тип | Описание |
|-----|----------|
| **Input** | Входной порт; получает данные из связанной вышестоящей трансформации или Source. Не содержит выражения; значение приходит из потока. |
| **Output** | Выходной порт; передаёт данные в нижестоящую трансформацию или Target. Может содержать выражение (Expression, Filter, Router и др.) или быть pass-through (просто передаёт значение Input). |
| **Variable** | Промежуточный порт; хранит результат вычисления для использования в других Variable или Output портах. Не передаётся наружу трансформации; только внутри. |

Не все трансформации имеют Variable порты. Expression, Aggregator и некоторые другие поддерживают Variable для промежуточных расчётов.

---

## 5.4.2. Порядок вычисления портов

Integration Service вычисляет порты в фиксированном порядке. Источник: [Port Order](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/local-variables/guidelines-for-configuring-variable-ports/port-order.html).

1. **Input ports** — первыми; не зависят от других портов. Порядок создания не важен.
2. **Variable ports** — вторыми. Могут ссылаться на Input и другие Variable, но **не на Output**. Порядок Variable важен: если Variable A используется в Variable B, A должен быть выше B в списке.
3. **Output ports** — последними. Могут ссылаться на Input и Variable. Output не может ссылаться на другой Output.

Пример: расчёт первоначальной стоимости здания (Variable) и затем корректировка на амортизацию (другая Variable) — первая Variable должна быть выше второй.

---

## 5.4.3. Выражения в трансформациях

Выражения задают логику: вычисление, условие, агрегацию. Источник: [Working with Expressions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/working-with-expressions.html).

| Трансформация | Назначение выражения | Возвращаемое значение |
|---------------|----------------------|------------------------|
| **Expression** | Расчёт на уровне строки (цена × количество, конкатенация и т.д.) | Результат вычисления для порта |
| **Filter** | Условие фильтрации | TRUE/FALSE; проходят только TRUE |
| **Router** | Условие для каждой группы | TRUE/FALSE по группе |
| **Aggregator** | Агрегатная функция (SUM, AVG, COUNT) или фильтр записей | Результат агрегации |
| **Update Strategy** | Флаг insert/update/delete/reject | Числовой код (DD_INSERT, DD_UPDATE и др.) |
| **Rank** | Условие для ранжирования | Результат условия или расчёта |
| **Transaction Control** | Условие commit/rollback | TC_CONTINUE_TRANSACTION, TC_COMMIT_BEFORE и др. |

Expression Editor доступен в этих трансформациях; поддерживает функции Transformation Language, пользовательские и custom-функции.

---

## 5.4.4. IIF: условное выражение

**IIF** (If-Then-Else) — возвращает одно из двух значений в зависимости от условия. Источник: [IIF](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-language-reference/functions/iif.html).

**Синтаксис:**
```
IIF( condition, value1 [, value2] )
```

- **condition** — выражение, дающее TRUE или FALSE.
- **value1** — значение при TRUE.
- **value2** — опционально; значение при FALSE. Если опущено: 0 для Numeric, пустая строка для String, NULL для Date/Time.

**Пример:** `IIF( SALES > 100, EMP_NAME, NULL )` — при продажах > 100 возвращает имя, иначе NULL.

**Вложенный IIF** для нескольких условий:
```
IIF( SALES > 0, IIF( SALES < 50, SALARY1, IIF( SALES < 100, SALARY2, BONUS )), 0 )
```

При множественных условиях DECODE может быть читаемее. См. [§5.4.5](chapter-05-04.md).

---

## 5.4.5. DECODE: поиск по значению

**DECODE** — ищет значение в порте и возвращает соответствующий результат. Источник: [DECODE](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-language-reference/functions/decode.html).

**Синтаксис:**
```
DECODE( value, first_search, first_result [, second_search, second_result]... [, default] )
```

Условия проверяются по порядку; возвращается результат первого совпадения. Если совпадений нет — default или NULL.

**Пример (поиск по ID):**
```
DECODE( ITEM_ID, 10, 'Flashlight', 14, 'Regulator', 20, 'Knife', 40, 'Tank', 'NONE' )
```

**DECODE( TRUE, ...)** для сложных условий (аналог множественного IIF):
```
DECODE( TRUE,
   SALES > 0 AND SALES < 50, SALARY1,
   SALES >= 50 AND SALES < 100, SALARY2,
   SALES >= 100 AND SALES < 200, SALARY3,
   SALES >= 200, BONUS,
   0 )
```

DECODE с TRUE проверяет условия сверху вниз и возвращает результат первого TRUE.

---

## 5.4.6. Другие часто используемые функции

- **ISNULL( value )** — TRUE, если value NULL.
- **NULLIF( value1, value2 )** — возвращает NULL, если value1 = value2; иначе value1.
- **NVL( value, default )** — возвращает default, если value NULL; иначе value.
- **SUBSTR, CONCAT, LTRIM, RTRIM** — строковые функции.
- **TO_DATE, TO_CHAR, TO_DECIMAL, TO_INTEGER** — преобразование типов.
- **SUM, AVG, COUNT, MIN, MAX** — агрегатные (в Aggregator).

Полный список — в [Transformation Language Reference](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-language-reference/functions.html).

---

## 5.4.7. Mapping parameters в выражениях

**Mapping parameter** — параметр, определённый в маппинге; значение задаётся при выполнении Session (в parameter file или в настройках). В выражениях используется синтаксис `$$ParameterName`. Источник: [§2.3](chapter-02-03.md).

Пример: `IIF( LOAD_DATE >= $$StartDate, 1, 0 )` — сравнение с параметром `StartDate`. Позволяет менять логику между запусками без изменения маппинга.

---

## 5.4.8. Input/Output порты

Некоторые порты имеют тип **Input/Output** — могут и принимать, и отдавать данные. В зависимости от трансформации и настройки порт ведёт себя как Input или Output. В связывании: Output/Input-Output подключается к Input/Input-Output.

---

## 5.4.9. Типичные ошибки

- **Variable ссылается на Output:** Variable может ссылаться только на Input и другие Variable; иначе ошибка валидации.
- **Неправильный порядок Variable:** Variable B использует Variable A — A должен быть выше B в списке портов.
- **Смешение типов в DECODE:** нельзя смешивать строковые и числовые результаты в одном DECODE; тип определяется по результату с наибольшей точностью.
- **IIF без value2 при необходимости NULL:** при опущенном value2 для String возвращается пустая строка, не NULL; явно указывать NULL при необходимости.
- **Забыть default в DECODE:** при отсутствии совпадения возвращается NULL; для явного значения — указать default.

---

## Ключевое

- **Input** — получает данные из потока; **Output** — передаёт наружу, может содержать выражение; **Variable** — промежуточный расчёт, только внутри трансформации.
- **Порядок вычисления:** Input → Variable → Output. Variable может ссылаться на Input и Variable; Output — на Input и Variable.
- **Выражения** — в Expression, Filter, Router, Aggregator, Update Strategy, Rank, Transaction Control.
- **IIF( condition, value1 [, value2] )** — условное значение; при опущенном value2 — 0, '', NULL в зависимости от типа.
- **DECODE( value, search1, result1, ... [, default] )** — поиск по значению; DECODE( TRUE, cond1, res1, ... ) — для сложных условий.
- **Mapping parameters** — `$$ParameterName` в выражениях.

В [§6.1](chapter-06-01.md) мы разберём Source Qualifier: SQL-override, фильтрация на уровне источника, сортировка и join нескольких таблиц.
