# §9.2 Цепочка преобразований

В [§9.1](chapter-09-01.md) мы разобрали конфигурацию. ETL-пайплайн — последовательность шагов: **read** (чтение данных), **transform** (преобразования), **write** (запись результата). В этом разделе: организация такой цепочки, композиция функций или вызов по шагам и обработка ошибок через **Option** и **Try**. В [§9.3](chapter-09-03.md) подробнее рассмотрим Try, Either и логирование. См. [Глоссарий](glossary.md).

---

## 9.2.1. Последовательность шагов

Типичный ETL можно представить как цепочку:

```
read(input) → transform1 → transform2 → ... → write(output)
```

Каждый шаг — функция; результат одного шага передаётся в следующий. Ошибка на любом шаге должна прерывать выполнение или обрабатываться явно.

---

## 9.2.2. Вызов по шагам

Простейший способ — последовательные вызовы с промежуточными переменными:

```scala
def runPipeline(config: JobConfig): Unit = {
  val raw = readData(config.inputPath)
  val cleaned = filterInvalid(raw)
  val transformed = transform(cleaned)
  writeData(config.outputPath, transformed)
}
```

Читаемость высокая; легко добавить логирование или проверки между шагами. Недостаток — при ошибке в середине цепочки нужно решать, как её обработать.

---

## 9.2.3. Option и Try для обработки ошибок

**Option** — когда операция может не вернуть результат (None при отсутствии данных или ошибке разбора). **Try** — когда операция может выбросить исключение: Success(result) или Failure(exception).

```scala
def readData(path: String): Option[List[Record]] =
  scala.util.Try(parseCSV(path)).toOption

def runPipeline(config: JobConfig): Option[Unit] =
  for {
    raw <- readData(config.inputPath)
    cleaned = filterInvalid(raw)
    transformed = transform(cleaned)
  } yield writeData(config.outputPath, transformed)
```

Цепочка через for-comprehension коротко замыкается на первом None: при неудачном чтении последующие шаги не выполняются.

---

## 9.2.4. Try для операций с исключениями

**Try[T]** удобен, когда шаги могут выбросить исключение (I/O, парсинг):

```scala
import scala.util.{Try, Success, Failure}

def runPipeline(config: JobConfig): Try[Unit] =
  for {
    raw    <- Try(readData(config.inputPath))
    cleaned = filterInvalid(raw)
    _      <- Try(writeData(config.outputPath, transform(cleaned)))
  } yield ()

runPipeline(config) match {
  case Success(_) => println("Done")
  case Failure(e) => println(s"Error: ${e.getMessage}")
}
```

Try автоматически оборачивает исключения в Failure; for-comprehension останавливается на первом Failure.

---

## 9.2.5. Композиция функций

Если каждый шаг — чистая функция без I/O, их можно композировать:

```scala
val pipeline = filterInvalid _ andThen transform
val result = pipeline(readData(path))
```

Или вручную через вложенные вызовы:

```scala
val result = transform(filterInvalid(readData(path)))
```

В ETL чаще смешаны чистые преобразования и операции с эффектами (чтение, запись). Тогда цепочку строят по шагам, а чистые части выделяют в отдельные функции (см. [§9.4](chapter-09-04.md)).

---

## 9.2.6. Пример цепочки с Try

```scala
import scala.util.{Try, Using}
import scala.io.{Source, Codec}
import java.io.PrintWriter

case class Record(id: Int, name: String, value: Double)

def read(path: String): Try[List[Record]] = Try(parseCSV(path))

def transform(records: List[Record]): List[Record] =
  records.filter(_.value > 0).map(r => r.copy(name = r.name.trim))

def write(path: String, records: List[Record]): Try[Unit] =
  Using(new PrintWriter(path, "UTF-8")) { pw =>
    records.foreach(r => pw.println(s"${r.id},${r.name},${r.value}"))
  }.toTry

def runPipeline(inputPath: String, outputPath: String): Try[Unit] =
  for {
    raw  <- read(inputPath)
    transformed = transform(raw)
    _    <- write(outputPath, transformed)
  } yield ()
```

read и write возвращают Try; transform — чистая функция. Ошибка read или write приводит к Failure; transform не бросает исключений.

---

## Ключевое

- ETL-пайплайн — последовательность: read → transform → write.
- **Вызов по шагам** — понятная структура; между шагами легко добавлять логирование и проверки.
- **Option** и **Try** — для обработки ошибок; for-comprehension коротко замыкается на None/Failure.
- **Try** — для операций с исключениями (I/O); Success/Failure вместо throw/catch.
- Чистые преобразования выделяют в отдельные функции; композиция — andThen или вложенные вызовы.

В [§9.3](chapter-09-03.md) мы подробнее рассмотрим Try, Either и логирование при ошибках.
