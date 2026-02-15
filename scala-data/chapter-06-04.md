# §6.4 Трейты (traits)

В [§6.3](chapter-06-03.md) мы разобрали case-классы. **Trait** (трейт) в Scala — механизм для описания интерфейса и **смешиваемого поведения**: класс может наследовать несколько трейтов через **extends** и **with**. Трейт похож на интерфейс в Java, но может содержать реализацию методов и полей. В этом разделе: объявление трейта, наследование через extends и with, пример трейта для «логируемого» поведения. В [§6.5](chapter-06-05.md) рассмотрим иммутабельность и чистые функции. См. [Глоссарий](glossary.md).

---

## 6.4.1. Объявление трейта

Синтаксис: **trait Имя** — с телом в фигурных скобках. Трейт может объявлять абстрактные методы (без тела) и методы с реализацией:

```scala
trait Comparable {
  def compare(other: Comparable): Int   // абстрактный метод
}

trait Loggable {
  def log(msg: String): Unit = println(s"[LOG] $msg")   // реализация по умолчанию
}
```

Трейт нельзя инстанцировать напрямую (нет `new Loggable`). Его смешивают в классы и объекты.

---

## 6.4.2. Наследование: extends

Класс наследует один трейт (или класс) через **extends**:

```scala
trait Loggable {
  def log(msg: String): Unit = println(s"[LOG] $msg")
}

class DataProcessor extends Loggable {
  def process(data: String): String = {
    log("Processing started")
    val result = data.toUpperCase
    log("Processing done")
    result
  }
}

val p = new DataProcessor
p.process("hello")   // выведет [LOG] ..., вернёт "HELLO"
p.log("test")        // метод из трейта
```

Класс получает все методы трейта. Если метод в трейте абстрактный, класс должен его реализовать (с **override** при переопределении уже реализованного).

---

## 6.4.3. Смешивание нескольких трейтов: with

Несколько трейтов смешивают через **with**; первый — через **extends**, остальные — через **with**:

```scala
trait Loggable {
  def log(msg: String): Unit = println(s"[LOG] $msg")
}

trait Timestamped {
  def timestamp: Long = System.currentTimeMillis
}

class DataService extends Loggable with Timestamped {
  def run(): Unit = log(s"Started at $timestamp")
}
```

Порядок трейтов влияет на разрешение методов при пересечении имён (линеаризация). На практике обычно избегают конфликтов и делают трейты узконаправленными.

---

## 6.4.4. Абстрактные члены трейта

Трейт может объявлять абстрактные методы и поля; класс, его наследующий, обязан их реализовать:

```scala
trait Writer {
  def write(line: String): Unit   // абстрактный
}

class FileWriter(path: String) extends Writer {
  override def write(line: String): Unit = {
    // запись в файл
  }
}
```

Абстрактное поле: `val name: String` — класс должен определить `name`. Трейт задаёт контракт; конкретный класс даёт реализацию.

---

## 6.4.5. Пример: трейт Loggable

Трейт **Loggable** добавляет поведение логирования классам данных или сервисам:

```scala
trait Loggable {
  def log(msg: String): Unit = println(s"[INFO] $msg")
  def logError(msg: String): Unit = println(s"[ERROR] $msg")
}

case class Order(id: Long, amount: Double)

class OrderService extends Loggable {
  def process(order: Order): Unit = {
    log(s"Processing order ${order.id}")
    if (order.amount < 0) logError("Invalid amount")
    // ...
  }
}
```

Любой класс, наследующий `Loggable`, получает методы `log` и `logError` без дублирования кода. Трейт можно смешать в несколько классов.

---

## 6.4.6. Трейт как интерфейс

Трейт без реализации — аналог интерфейса: описание контракта:

```scala
trait Closeable {
  def close(): Unit
}

class Resource extends Closeable {
  override def close(): Unit = { /* освобождение ресурса */ }
}
```

Тип `Closeable` можно использовать для полиморфизма — везде, где нужен «закрываемый» объект, подойдёт любой класс, реализующий трейт.

---

## 6.4.7. Типичные ошибки

- **Инстанцировать трейт** — `new Loggable` недопустимо; трейт смешивают в класс.
- **Забыть override** — при переопределении метода трейта нужен **override def**.
- **Циклическое наследование трейтов** — трейт A extends B, B extends A; компилятор может не разрешить или привести к неожиданной линеаризации.

---

## Ключевое

- **trait** — описание интерфейса и смешиваемого поведения; может содержать абстрактные и реализованные методы.
- **extends** — наследование одного трейта (или класса); **with** — смешивание дополнительных трейтов.
- Класс обязан реализовать абстрактные члены трейта; при переопределении — **override**.
- Трейт нельзя инстанцировать; его смешивают в классы. Пример: `Loggable` — общее поведение логирования для нескольких классов.

В [§6.5](chapter-06-05.md) мы рассмотрим иммутабельность и чистые функции: предпочтение неизменяемых структур и функций без побочных эффектов.
