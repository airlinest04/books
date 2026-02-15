# §9.3 Обработка ошибок

В [§9.2](chapter-09-02.md) мы использовали Try в цепочке преобразований. **Try[T]** и **Either[E, A]** — основные способы функциональной обработки ошибок в Scala: вместо исключений возвращают результат, несущий либо успех, либо информацию об ошибке. В этом разделе: Try (Success, Failure), Either (Left, Right) и кратко логирование и повтор при сбоях. В [§9.4](chapter-09-04.md) рассмотрим тестируемость. См. [Глоссарий](glossary.md).

---

## 9.3.1. Try[T]: Success и Failure

**Try[T]** — контейнер для вычисления, которое может выбросить исключение. Два подтипа:

- **Success(x)** — результат вычисления;
- **Failure(e)** — исключение, перехваченное при выполнении.

```scala
import scala.util.{Try, Success, Failure}

val t1 = Try(1 / 0)   // Failure(ArithmeticException)
val t2 = Try(10 / 2)  // Success(5)

def parseInt(s: String): Try[Int] = Try(s.trim.toInt)
parseInt("42")   // Success(42)
parseInt("x")    // Failure(NumberFormatException)
```

Try применяют к блоку кода; при исключении возвращается Failure с этим исключением.

---

## 9.3.2. Обработка Try: match, getOrElse, recover

**Pattern matching:**

```scala
parseInt(input) match {
  case Success(n) => println(s"Число: $n")
  case Failure(e) => println(s"Ошибка: ${e.getMessage}")
}
```

**getOrElse** — вернуть значение при Success или default при Failure:

```scala
parseInt("x").getOrElse(0)   // 0
```

**recover** — преобразовать Failure в Success с запасным значением:

```scala
Try(riskyOperation()).recover { case e: IOException => defaultValue }
```

**toOption** — Success(x) → Some(x), Failure → None. **fold** — обработать оба случая одной функцией.

---

## 9.3.3. Either[E, A]: Left и Right

**Either[E, A]** — значение одного из двух типов: **Left(e)** (обычно ошибка) или **Right(a)** (успех). С Scala 2.12 Either «правый»: map и flatMap работают с Right.

```scala
def parseConfig(path: String): Either[String, Config] =
  if (path.nonEmpty) Right(loadConfig(path))
  else Left("Путь не задан")

parseConfig("app.conf")   // Right(Config(...))
parseConfig("")           // Left("Путь не задан")
```

Either удобен, когда нужно возвращать осмысленное сообщение об ошибке (String, case class), а не исключение.

---

## 9.3.4. Обработка Either: match, fold, map

**Pattern matching:**

```scala
parseConfig(path) match {
  case Right(config) => runJob(config)
  case Left(error)   => println(s"Ошибка: $error")
}
```

**fold** — обработать оба варианта:

```scala
result.fold(
  error => println(s"Ошибка: $error"),
  config => runJob(config)
)
```

**map**, **flatMap** — применяются к Right; при Left результат остаётся Left. **getOrElse** — значение Right или default при Left.

---

## 9.3.5. Try vs Either

| Try | Either |
|-----|--------|
| Оборачивает код, который может выбросить исключение | Явный возврат Left/Right из функции |
| Failure содержит Throwable | Left может быть любого типа (String, case class) |
| Удобен для обёртки существующего кода | Удобен для domain-ошибок с понятным типом |

**Try.toEither** — преобразовать Try в Either: Success(x) → Right(x), Failure(e) → Left(e).

---

## 9.3.6. Логирование при ошибках

При Failure или Left логируют сообщение и контекст:

```scala
runPipeline(config) match {
  case Success(_) => logger.info("Pipeline completed")
  case Failure(e) => logger.error(s"Pipeline failed: ${e.getMessage}", e)
}
```

Или с Either:

```scala
result.fold(
  err => { logger.error(s"Config error: $err"); System.exit(1) },
  cfg => runJob(cfg)
)
```

Детали логирования — в [§9.5](chapter-09-05.md).

---

## 9.3.7. Повтор при сбое

Для нестабильных операций (сеть, внешние API) иногда добавляют повтор:

```scala
def withRetry[T](n: Int)(f: => T): Try[T] = {
  def attempt(remaining: Int): Try[T] =
    Try(f) match {
      case s: Success[T] => s
      case Failure(_) if remaining > 1 => attempt(remaining - 1)
      case f => f
    }
  attempt(n)
}

withRetry(3)(readFromApi(url))
```

Упрощённый пример; в production часто используют библиотеки с экспоненциальной задержкой и ограничением по времени.

---

## Ключевое

- **Try[T]** — Success(x) или Failure(exception); оборачивает код, который может выбросить исключение.
- Обработка: match, getOrElse, recover, toOption.
- **Either[E, A]** — Left(error) или Right(result); для возврата доменной ошибки вместо исключения.
- Обработка: match, fold, map, getOrElse.
- Логирование при Failure/Left; повтор — через обёртку с ограничением попыток.

В [§9.4](chapter-09-04.md) мы рассмотрим тестируемость: выделение чистых функций и тесты на малых данных.
