# §6.2 Объекты (singleton)

В [§6.1](chapter-06-01.md) мы разобрали классы. **Объект** (object) в Scala — **singleton**: существует ровно один экземпляр; создаётся лениво при первом обращении. Объект заменяет статические члены из Java и часто используется для точки входа (main), утилит и **объекта-компаньона** — объекта с тем же именем, что и класс, в одном файле. В [§6.3](chapter-06-03.md) рассмотрим case-классы. См. [Глоссарий](glossary.md).

---

## 6.2.1. Объявление объекта

Синтаксис: **object Имя** — как у класса, но ключевое слово `object`. Создаётся один экземпляр; к методам и полям обращаются как **ObjectName.method()**, **ObjectName.field**:

```scala
object Logger {
  def info(message: String): Unit = println(s"INFO: $message")
  def error(message: String): Unit = println(s"ERROR: $message")
}

Logger.info("Started")
Logger.error("Something went wrong")
```

Объект может содержать поля и методы. Поля вычисляются при первом обращении (ленивая инициализация). Объект — «стабильный путь» для импортов: `import Logger.info` импортирует метод.

---

## 6.2.2. Поля и методы объекта

Поля объекта — общие для всех обращений (аналог `static` в Java):

```scala
object Config {
  val appName: String = "DataProcessor"
  var debug: Boolean = false

  def isDebugEnabled: Boolean = debug
}

Config.appName    // "DataProcessor"
Config.debug = true
Config.isDebugEnabled   // true
```

Методы объекта вызываются без создания экземпляра. Объект удобен для констант, настроек, фабричных методов и утилит.

---

## 6.2.3. Объект-компаньон

**Объект-компаньон** (companion object) — объект с тем же именем, что и класс, объявленный в **том же файле**. Класс при этом называют классом-компаньоном. Компаньоны могут обращаться к приватным членам друг друга.

```scala
// Файл Person.scala
class Person(val name: String, val age: Int) {
  override def toString: String = s"Person($name, $age)"
}

object Person {
  def apply(name: String, age: Int): Person = new Person(name, age)

  def fromBirthYear(name: String, birthYear: Int): Person =
    new Person(name, 2025 - birthYear)
}
```

Вызов **Person.apply(...)** позволяет писать **Person(name, age)** вместо `new Person(name, age)` — метод `apply` вызывается при обращении к объекту как к функции. Case-классы ([§6.3](chapter-06-03.md)) автоматически получают такой `apply` в компаньоне.

---

## 6.2.4. apply — фабричный метод

Метод **apply** в объекте-компаньоне позволяет создавать экземпляры без ключевого слова `new`:

```scala
object Person {
  def apply(name: String, age: Int): Person = new Person(name, age)
}

val p = Person("Alice", 30)   // эквивалентно Person.apply("Alice", 30)
```

Синтаксис `Person("Alice", 30)` — вызов `Person.apply("Alice", 30)`. Это типичный приём для фабрик и билдеров.

---

## 6.2.5. Другие фабричные методы в компаньоне

Помимо `apply`, в компаньоне размещают альтернативные способы создания экземпляра:

```scala
object User {
  def apply(id: Int, name: String): User = new User(id, name)

  def fromString(s: String): Option[User] = {
    val parts = s.split(",")
    if (parts.length == 2)
      Some(new User(parts(0).trim.toInt, parts(1).trim))
    else
      None
  }
}
```

Такие методы возвращают `Option`, если создание может не удаться (например, при разборе строки).

---

## 6.2.6. Объект как точка входа программы

Объект с методом **main(args: Array[String]): Unit** — классическая точка входа для JVM (рассмотрено в [§1.4](chapter-01-04.md)):

```scala
object Main {
  def main(args: Array[String]): Unit = {
    println("Hello, World!")
  }
}
```

При запуске JVM ищет класс с методом `main`; компилятор Scala генерирует его для объекта `Main`. В Scala 3 также можно использовать `@main` для более простого объявления точки входа.

---

## 6.2.7. Требование: один файл

Класс и его объект-компаньон должны быть объявлены в **одном и том же файле**. Иначе они не считаются компаньонами и не имеют доступа к приватным членам друг друга. В REPL их нужно вводить вместе (например, в режиме `:paste`).

---

## 6.2.8. Типичные ошибки

- **Объект и класс в разных файлах** — они не будут компаньонами; приватный доступ не сработает.
- **Создание экземпляра объекта** — объект — singleton, к нему не применяют `new`; обращаются по имени: `Logger.info(...)`.
- **Путать класс и объект** — `Person` как класс создаёт экземпляры через `new Person(...)`; `Person` как объект — singleton с методами вроде `Person.apply(...)`.

---

## Ключевое

- **object Имя** — singleton: один экземпляр, ленивая инициализация. Методы и поля вызывают как **ObjectName.method()**, **ObjectName.field**.
- **Объект-компаньон** — объект с тем же именем, что и класс, в одном файле; класс и объект могут обращаться к приватным членам друг друга.
- **apply** в компаньоне — фабричный метод: `Person("Alice", 30)` эквивалентно `Person.apply("Alice", 30)`.
- В компаньоне размещают фабрики (`fromString`, `apply`), утилиты и альтернативные способы создания экземпляров.
- Точка входа — объект с `def main(args: Array[String]): Unit`.
- Класс и компаньон должны быть в одном файле.

В [§6.3](chapter-06-03.md) мы рассмотрим case-классы: автоматические equals, hashCode, toString и создание без new.
