# §9.4 Тестируемость

В [§9.3](chapter-09-03.md) мы разобрали обработку ошибок через Try и Either. **Тестируемость** ETL-кода повышается, когда преобразования данных вынесены в **чистые функции**: их проверяют на малых примерах без файлов, БД и кластера. В этом разделе: выделение чистых функций преобразования, тесты на малых данных (ScalaTest или minitest) и моки для внешних вызовов. В [§9.5](chapter-09-05.md) рассмотрим логирование. См. [Глоссарий](glossary.md).

---

## 9.4.1. Выделение чистых функций преобразования

Чистая функция — детерминированная, без побочных эффектов ([§6.5](chapter-06-05.md)). Преобразования данных (фильтрация, парсинг, агрегация) естественно оформлять как чистые:

```scala
def filterInvalid(records: List[Record]): List[Record] =
  records.filter(r => r.id > 0 && r.value >= 0)

def parseAmount(s: String): Option[Double] =
  scala.util.Try(s.trim.replace(",", ".").toDouble).toOption
```

Чтение и запись (I/O) остаются на границах; внутренняя логика — чистые функции, которые легко тестировать.

---

## 9.4.2. Тесты на малых примерах данных

Для чистой функции достаточно передать входные данные и проверить результат. Зависимость в sbt:

```scala
libraryDependencies += "org.scalatest" %% "scalatest" % "3.2.19" % "test"
```

Пример с **ScalaTest** (AnyFunSuite):

```scala
import org.scalatest.funsuite.AnyFunSuite

class TransformSpec extends AnyFunSuite {
  test("filterInvalid removes records with negative id") {
    val input = List(Record(1, "a", 10.0), Record(-1, "b", 5.0))
    assert(filterInvalid(input) == List(Record(1, "a", 10.0)))
  }

  test("parseAmount returns Some for valid number") {
    assert(parseAmount("10.5") == Some(10.5))
    assert(parseAmount("1,5") == Some(1.5))
  }

  test("parseAmount returns None for invalid input") {
    assert(parseAmount("x") == None)
  }
}
```

Запуск: `sbt test`.

---

## 9.4.3. Minitest (кратко)

**Minitest** — лёгкий фреймворк без лишних зависимостей:

```scala
libraryDependencies += "io.monix" %% "minitest" % "2.9.6" % "test"
testFrameworks += new TestFramework("minitest.runner.Framework")
```

```scala
import minitest._

object TransformSuite extends SimpleTestSuite {
  test("filterInvalid") {
    val input = List(Record(1, "a", 10.0))
    assertEquals(filterInvalid(input), input)
  }
}
```

---

## 9.4.4. Моки для внешних вызовов

Когда шаг пайплайна зависит от внешнего сервиса (API, БД, файловая система), в тестах его заменяют **моком** — заглушкой с предсказуемым поведением.

Подход 1: **инъекция зависимости** — чтение/запись передаются как параметр:

```scala
def runPipeline(
  read: String => Try[List[Record]],
  write: (String, List[Record]) => Try[Unit],
  config: JobConfig
): Try[Unit] = for {
  raw <- read(config.inputPath)
  _   <- write(config.outputPath, transform(raw))
} yield ()
```

В тесте передают функции, возвращающие фиксированные данные; в production — реальные read/write.

Подход 2: **трайт с реализациями** — интерфейс для I/O, тестовая реализация возвращает стабы:

```scala
trait DataSource {
  def read(path: String): Try[List[Record]]
  def write(path: String, data: List[Record]): Try[Unit]
}

class MockDataSource(data: List[Record]) extends DataSource {
  def read(path: String) = Success(data)
  def write(path: String, d: List[Record]) = Success(())
}
```

---

## 9.4.5. Практические рекомендации

- Выделять преобразования в отдельные функции без I/O.
- Тестировать граничные случаи: пустой список, невалидные строки, предельные значения.
- Избегать тестов, зависящих от порядка выполнения или глобального состояния.
- Для сложных моков можно использовать библиотеки (например, mockito-scala), но часто достаточно простых заглушек.

---

## Ключевое

- **Чистые функции** преобразования — без I/O и мутации; легко тестируются на малых примерах.
- **ScalaTest** или **minitest** — фреймворки для unit-тестов; sbt test для запуска.
- **Моки** — заглушки для внешних вызовов; инъекция зависимостей или трайты с тестовыми реализациями.
- Тесты на граничных случаях; изоляция от файловой системы и сети.

В [§9.5](chapter-09-05.md) мы рассмотрим логирование: выбор библиотеки и логирование в ETL-пайплайне.
