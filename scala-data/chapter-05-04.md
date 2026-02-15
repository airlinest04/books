# §5.4 Option (повторение и углубление)

В [§5.3](chapter-05-03.md) мы разобрали Set. Возвращаемся к **Option** — в [§2.5](chapter-02-05.md) мы познакомились с Some/None и getOrElse. Здесь углубим работу с Option: цепочки преобразований через `.map` и `.flatMap`, а также **for-comprehension** над Option — удобный способ комбинировать несколько опциональных вычислений. В [§5.5](chapter-05-05.md) рассмотрим общие операции коллекций: map, flatMap, filter, foldLeft. См. [Глоссарий](glossary.md).

---

## 5.4.1. Краткое напоминание: Some и None

**Option[T]** — либо `Some(x)` (значение есть), либо `None` (значения нет). Тип явно показывает возможность отсутствия результата. Создание и базовые проверки:

```scala
val some: Option[Int] = Some(42)
val none: Option[Int] = None

some.isEmpty   // false
none.isEmpty   // true
some.getOrElse(0)   // 42
none.getOrElse(0)   // 0
```

Option часто появляется при `Map.get`, разборе строк, работе с полями данных с пропусками и при обёртке Java-кода. Подробнее — в [§2.5](chapter-02-05.md).

---

## 5.4.2. map — преобразование значения внутри Option

Метод **.map(f)** применяет функцию к значению внутри Option. Если Option — `Some(x)`, результат — `Some(f(x))`; если `None` — остаётся `None`:

```scala
Some(21).map(_ * 2)       // Some(42)
None.map((x: Int) => x * 2)   // None

Some("hello").map(_.length)   // Some(5)
None: Option[String]; None.map(_.length)   // None
```

map не требует явной проверки на None: преобразование выполняется только при наличии значения. Цепочка map-вызовов коротко замыкается на первом None:

```scala
Some(10).map(_ + 1).map(_ * 2).map(_.toString)   // Some("22")
None.map(_ + 1).map(_ * 2)   // None
```

---

## 5.4.3. flatMap — цепочка опциональных вычислений

Метод **.flatMap(f)** принимает функцию `A => Option[B]`. Если Option — `Some(x)`, к `x` применяется `f` и возвращается результат (Option[B]); если `None` — возвращается `None`. В отличие от map, функция возвращает Option, а flatMap «разворачивает» вложенность, не создавая `Option[Option[B]]`.

Типичный сценарий — последовательный доступ к Map по цепочке ключей:

```scala
val users = Map("alice" -> Map("city" -> "Moscow"), "bob" -> Map("city" -> "SPB"))
val city: Option[String] = users.get("alice").flatMap(_.get("city"))   // Some("Moscow")
users.get("charlie").flatMap(_.get("city"))   // None
```

Если бы использовали только map, получили бы вложенный Option:

```scala
users.get("alice").map(_.get("city"))   // Some(Some("Moscow")) — Option[Option[String]]
users.get("alice").flatMap(_.get("city"))   // Some("Moscow") — Option[String]
```

Другой пример — разбор и преобразование:

```scala
def parseInt(s: String): Option[Int] = scala.util.Try(s.trim.toInt).toOption

def doubleIfPositive(n: Int): Option[Int] = if (n > 0) Some(n * 2) else None

parseInt("21").flatMap(doubleIfPositive)   // Some(42)
parseInt("-1").flatMap(doubleIfPositive)   // None
parseInt("x").flatMap(doubleIfPositive)    // None
```

---

## 5.4.4. getOrElse и orElse

**getOrElse(default)** возвращает значение из Some или default при None (рассмотрено в §2.5):

```scala
Some(42).getOrElse(0)   // 42
None.getOrElse(0)       // 0
```

**orElse(alternative)** возвращает текущий Option, если он Some, иначе — альтернативный Option:

```scala
Some(1).orElse(Some(2))   // Some(1)
None.orElse(Some(2))      // Some(2)
None.orElse(None)         // None
```

Это удобно для каскада значений по умолчанию или альтернативных источников данных.

---

## 5.4.5. for-comprehension над Option

**for-comprehension** — синтаксический сахар для цепочек flatMap и map. Каждый генератор вида `x <- opt` извлекает значение из Option; если на любом шаге встретится None, весь результат — None.

Пример: получить город пользователя по имени и дополнительно проверить, что город непустой:

```scala
val users = Map("alice" -> Map("city" -> "Moscow"), "bob" -> Map("city" -> ""))

def nonEmpty(s: String): Option[String] = if (s.nonEmpty) Some(s) else None

val city = for {
  profile <- users.get("alice")
  rawCity <- profile.get("city")
  city <- nonEmpty(rawCity)
} yield city
// city: Option[String] = Some("Moscow")
```

Эквивалентная запись через flatMap:

```scala
val city = users.get("alice")
  .flatMap(_.get("city"))
  .flatMap(nonEmpty)
```

for-comprehension часто читается легче при нескольких шагах. Можно добавлять фильтры через `if`:

```scala
val result = for {
  a <- Some(10)
  b <- Some(5)
  if b != 0
} yield a / b
// result: Option[Int] = Some(2)

val fail = for {
  a <- Some(10)
  b <- None
  if b != 0
} yield a / b
// fail: Option[Int] = None
```

Если `b <- None`, следующий шаг не выполняется, результат — None. Фильтр `if b != 0` при false отфильтровывает значение и даёт None (через withFilter).

---

## 5.4.6. filter, foreach, toList

Option поддерживает методы, аналогичные коллекциям:

- **filter(p)** — оставить Some(x), если `p(x)` истинно; иначе None:

```scala
Some(5).filter(_ > 0)   // Some(5)
Some(-1).filter(_ > 0)  // None
None.filter((x: Int) => x > 0)   // None
```

- **foreach(f)** — применить `f` к значению, если Some; при None — ничего не делать:

```scala
Some(42).foreach(println)   // выведет 42
None.foreach(println)       // ничего
```

- **toList** — превратить в список длины 0 или 1:

```scala
Some(1).toList   // List(1)
None.toList      // List()
```

---

## 5.4.7. Практический пример: цепочка опциональных полей

При работе с JSON, CSV или вложенными структурами часто нужно «пройти» по цепочке опциональных полей. for-comprehension упрощает код:

```scala
case class Address(city: Option[String], street: Option[String])
case class User(name: String, address: Option[Address])

def cityOf(user: User): Option[String] =
  for {
    addr <- user.address
    city <- addr.city
  } yield city

cityOf(User("Alice", Some(Address(Some("Moscow"), None))))   // Some("Moscow")
cityOf(User("Bob", None))   // None
```

Без for-comprehension пришлось бы вызывать flatMap вручную; при большом числе уровней код становится нечитаемым.

---

## 5.4.8. Типичные ошибки

- **Использовать .get** — при None будет исключение; предпочитайте getOrElse, match, map, flatMap.
- **map вместо flatMap** при функции, возвращающей Option — получится Option[Option[T]]; нужен flatMap.
- **Забыть про None** — при цепочке Map.get любой отсутствующий ключ даёт None; проверяйте результат.

---

## Ключевое

- **Option[T]** — Some(x) или None. **map(f)** — преобразовать значение внутри; при None остаётся None.
- **flatMap(f)** — функция возвращает Option; результат «разворачивается» в один Option. Нужен для цепочек: Map.get(...).flatMap(...).
- **getOrElse(default)**, **orElse(alternative)** — безопасное извлечение и альтернативы.
- **for-comprehension** — удобная запись цепочек flatMap/map над Option: `for { x <- opt1; y <- opt2 } yield ...`; при любом None результат — None.
- **filter**, **foreach**, **toList** — дополнительные методы Option по аналогии с коллекциями.

В [§5.5](chapter-05-05.md) мы рассмотрим общие операции коллекций: size, isEmpty, contains, map, flatMap, filter, foreach, foldLeft, reduce и toList.
