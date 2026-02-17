# §2.5 Option и null

В [§2.4](chapter-02-04.md) мы разобрали строки и их методы. Завершаем главу о типах и операторах типом **Option** — способом представить **отсутствие значения** в Scala. Option позволяет безопасно работать с опциональными результатами (разбор строки, поиск по ключу, поля с пропусками) без исключений и без `null`. В [§3.1](chapter-03-01.md) перейдём к условиям if и управляющим конструкциям.

---

## 2.5.1. Что такое Option[T]

**Option[T]** — контейнер, который либо содержит значение типа `T`, либо не содержит ничего. Два подтипа:

- **Some(x)** — содержит значение `x`;
- **None** — не содержит значения.

Тип `Option[Int]` означает: «либо Int, либо ничего». Это явно отражает возможность отсутствия результата в сигнатуре, в отличие от возврата «просто Int», когда при отсутствии пришлось бы бросать исключение или возвращать null. См. [Глоссарий](glossary.md).

```scala
val someValue: Option[Int] = Some(42)
val noValue: Option[Int] = None
```

Option — **обобщённый** тип: `Option[String]`, `Option[Double]`, `Option[User]` и т.д. Он входит в стандартную библиотеку Scala и используется повсеместно — в коллекциях (Map.get возвращает Option), при разборе данных и в API.

---

## 2.5.2. Когда появляется Option

**Map.get** — при доступе по ключу возвращает `Option[V]`: `Some(value)` при наличии ключа, `None` при отсутствии:

```scala
val m = Map("a" -> 1, "b" -> 2)
m.get("a")   // Some(1)
m.get("c")   // None
```

**Безопасный разбор строки в число** — `scala.util.Try` или обёртка, возвращающая Option; часто пишут вспомогательную функцию:

```scala
def parseInt(s: String): Option[Int] =
  scala.util.Try(s.trim.toInt).toOption
parseInt("42")   // Some(42)
parseInt("x")    // None
```

**Поля с пропусками** — при чтении CSV или JSON поле может отсутствовать; вместо null удобно использовать `Option[String]` или `Option[Double]`.

**Java-библиотеки** — методы Java могут возвращать null; при обёртке в Scala результат часто конвертируют в Option через `Option(javaValue)`.

---

## 2.5.3. Безопасное извлечение: getOrElse

Метод **.getOrElse(default)** возвращает значение, если Option — `Some(x)`, иначе — `default`:

```scala
Some(42).getOrElse(0)   // 42
None.getOrElse(0)       // 0
```

Тип `default` должен соответствовать типу Option: для `Option[Int]` — `Int` (или совместимый тип).

```scala
val maybeName = Some("Alice")
maybeName.getOrElse("unknown")   // "Alice"

val empty: Option[String] = None
empty.getOrElse("unknown")       // "unknown"
```

Для разбора данных с запасным значением:

```scala
val raw = "42"
val num = parseInt(raw).getOrElse(0)   // 42

val bad = "x"
val num2 = parseInt(bad).getOrElse(0)  // 0
```

---

## 2.5.4. Проверка isEmpty, isDefined, nonEmpty

Методы для проверки наличия значения:

- **.isEmpty** — `true` для `None`, `false` для `Some`;
- **.isDefined** — `true` для `Some`, `false` для `None`;
- **.nonEmpty** — то же, что `isDefined`.

```scala
Some(1).isEmpty    // false
None.isEmpty       // true
Some(1).isDefined  // true
```

**Метод .get** — возвращает значение из `Some` и выбрасывает исключение для `None`. Использовать не рекомендуется: предпочтительны `getOrElse`, pattern matching или методы `.map`, `.flatMap` ([§5.4](chapter-05-04.md)).

---

## 2.5.5. Option.apply и обёртка null

Функция **Option(x)** превращает значение в Option: если `x` не null, возвращает `Some(x)`, иначе — `None`:

```scala
Option(42)      // Some(42)
Option("hi")    // Some("hi")
Option(null)    // None
```

Это удобно при работе с Java-кодом или API, возвращающими null:

```scala
val javaResult: String = someJavaMethod()   // может вернуть null
val opt: Option[String] = Option(javaResult)
opt.getOrElse("")
```

---

## 2.5.6. Сопоставление с образцом (pattern matching)

Option можно разбирать через **match**:

```scala
opt match {
  case Some(x) => println(s"Значение: $x")
  case None    => println("Значения нет")
}
```

Подробнее match рассмотрен в [§3.2](chapter-03-02.md); здесь важно, что Option естественно обрабатывается двумя ветками — `Some` и `None`.

---

## 2.5.7. null и его избегание

В Scala есть **null** — ссылка на отсутствие объекта, унаследованная от Java. Для ссылочных типов переменная может быть `null`, что приводит к `NullPointerException` при обращении к методам или полям.

**Рекомендация: не использовать null в Scala-коде.** Вместо него используйте Option:

- тип `Option[T]` явно говорит о возможности отсутствия значения;
- компилятор не защитит от null, но защитит от непроверенного извлечения из Option при использовании `getOrElse`, `map` и т.д.;
- Option хорошо комбинируется с функциональными цепочками (.map, .flatMap, for-comprehension).

При взаимодействии с Java вызывайте `Option(javaValue)`, чтобы превратить возможный null в Option.

---

## 2.5.8. Краткий обзор map и flatMap

Option поддерживает **.map** и **.flatMap** — преобразование значения «внутри» Option без явной проверки на None:

```scala
Some(21).map(_ * 2)      // Some(42)
None.map((x: Int) => x * 2)   // None
```

Если Option — `None`, результат `.map` тоже `None`; если `Some(x)`, к `x` применяется функция. Подробнее — в [§5.4](chapter-05-04.md); здесь достаточно понимать, что Option — не просто «значение или null», а контейнер с удобными методами для цепочек преобразований.

---

## 2.5.9. Типичные ошибки

- **Вызов .get без проверки** — при `None` будет исключение; используйте `getOrElse` или pattern matching.
- **Сравнение с null вместо Option** — если вы работаете с Option, проверяйте `isEmpty` или `isDefined`, а не `opt == null`.
- **Смешение None и null** — `None` и `null` разные: `None` — объект типа `Option[Nothing]`; `null` — ссылка. Для Option используйте `None`, не `null`.

---

## Ключевое

- **Option[T]** — либо `Some(x)` (есть значение), либо `None` (значения нет). Явно моделирует отсутствие результата.
- **.getOrElse(default)** — безопасное извлечение: значение из Some или default при None.
- **Option(x)** — оборачивает значение в Option; для `x == null` возвращает None. Удобно при работе с Java.
- **.isEmpty**, **.isDefined** — проверка наличия значения; **.get** — небезопасно, предпочтительны getOrElse, match, map.
- **null** — избегать; использовать Option. При Java-совместимости применять `Option(javaValue)`.

В [§3.1](chapter-03-01.md) мы начнём главу об управляющих конструкциях: условие if, возвращающее значение, и блоки в фигурных скобках.
