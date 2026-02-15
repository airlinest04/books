# §7.3 Парсинг CSV и простого текстового формата

В [§7.2](chapter-07-02.md) мы разобрали запись файлов. **CSV** (Comma-Separated Values) — распространённый формат табличных данных: каждая строка — запись, поля разделены разделителем (запятая, точка с запятой). Scala не содержит встроенного модуля CSV; для простых файлов достаточно **split** и построчной обработки. В этом разделе: разбиение строки по разделителю, обработка заголовка и преобразование в case class или Map. В [§7.4](chapter-07-04.md) рассмотрим сериализацию в JSON. См. [Глоссарий](glossary.md).

---

## 7.3.1. Разбиение строки: split

Метод **split(regex)** разбивает строку по разделителю и возвращает массив подстрок:

```scala
"a,b,c".split(",")           // Array("a", "b", "c")
"a;b;c".split(";")           // Array("a", "b", "c")
"a  b  c".split("\\s+")      // Array("a", "b", "c") — по пробелам
```

Аргумент — регулярное выражение. Точка в regex означает «любой символ»; для буквальной точки используйте **"\\."** или **"[.]"**. Для простого разделителя (запятая, точка с запятой) **split(",")** и **split(";")** достаточны.

---

## 7.3.2. Чтение CSV и обработка заголовка

Читаем файл построчно и разбиваем каждую строку:

```scala
import scala.io.Source
import scala.io.Codec

val source = Source.fromFile("data.csv")(Codec.UTF8)
val lines = source.getLines().toList
source.close()

val rows = lines.map(_.split(",").map(_.trim).toList)
```

Первая строка часто содержит **заголовки**. Отделяем её и используем для доступа по имени столбца:

```scala
val header = rows.head   // List("id", "name", "age")
val dataRows = rows.tail

// Каждая строка — List[String] в том же порядке, что и header
dataRows.foreach { cols =>
  // cols(0) — id, cols(1) — name, cols(2) — age
}
```

---

## 7.3.3. Преобразование в Map по заголовку

Заголовок даёт имена столбцов; каждую строку можно превратить в **Map[String, String]**:

```scala
def rowToMap(header: List[String], values: List[String]): Map[String, String] =
  header.zip(values).toMap

val header = rows.head
val records = dataRows.map(cols => rowToMap(header, cols))

// records(0)("name"), records(0)("age")
```

Тип значений — String; при необходимости числа и даты преобразуют отдельно (toInt, toDouble и т.д.).

---

## 7.3.4. Преобразование в case class

Определяем case class и функцию разбора строки:

```scala
case class User(id: Int, name: String, age: Int)

def parseUser(header: List[String], values: List[String]): Option[User] = {
  val m = header.zip(values).toMap
  for {
    id   <- m.get("id").flatMap(s => scala.util.Try(s.toInt).toOption)
    name <- m.get("name")
    age  <- m.get("age").flatMap(s => scala.util.Try(s.toInt).toOption)
  } yield User(id, name, age)
}

val header = rows.head
val users = dataRows.flatMap(cols => parseUser(header, cols))
```

Если порядок столбцов фиксирован (id, name, age), можно парсить по индексу:

```scala
def parseUser(cols: List[String]): Option[User] = {
  if (cols.size < 3) None
  else
    for {
      id  <- scala.util.Try(cols(0).trim.toInt).toOption
      age <- scala.util.Try(cols(2).trim.toInt).toOption
    } yield User(id, cols(1).trim, age)
}

val users = dataRows.flatMap(parseUser)
```

---

## 7.3.5. Обработка пустых и неверных строк

Строки с неправильным числом полей или нечисловыми значениями можно пропускать или логировать:

```scala
def parseUserSafe(cols: List[String]): Option[User] =
  cols match {
    case id :: name :: age :: _ =>
      for {
        i <- scala.util.Try(id.trim.toInt).toOption
        a <- scala.util.Try(age.trim.toInt).toOption
      } yield User(i, name.trim, a)
    case _ => None
}
```

Проверка на пустые строки:

```scala
val lines = source.getLines().toList.filter(_.trim.nonEmpty)
```

---

## 7.3.6. Разделитель «точка с запятой»

Во многих локалях (например, европейские) в CSV используется **;** вместо запятой:

```scala
val delimiter = ";"
val rows = lines.map(_.split(delimiter).map(_.trim).toList)
```

Разделитель можно определять по первой строке (подсчёт запятых и точек с запятой) или задавать параметром.

---

## 7.3.7. Ограничения простого split

Метод **split** не обрабатывает:

- кавычки вокруг полей (`"a,b"` — одно поле с запятой внутри);
- экранирование кавычек (`""` внутри поля);
- переносы строк внутри кавычек.

Для таких CSV используют библиотеки (kantan.csv, opencsv и др.) или парсер по RFC 4180. Для простых выгрузок без кавычек и спецсимволов split достаточно.

---

## 7.3.8. Полный пример

```scala
import scala.io.{Source, Codec}
import scala.util.Using

case class Record(id: Int, name: String, value: Double)

def parseCSV(path: String): List[Record] = {
  Using.resource(Source.fromFile(path)(Codec.UTF8)) { source =>
    val lines = source.getLines().filter(_.trim.nonEmpty).toList
    if (lines.isEmpty) Nil
    else {
      val header = lines.head.split(",").map(_.trim).toList
      val dataRows = lines.tail.map(_.split(",").map(_.trim).toList)
      dataRows.flatMap { cols =>
        if (cols.size < 3) None
        else
          for {
            id    <- scala.util.Try(cols(0).toInt).toOption
            value <- scala.util.Try(cols(2).toDouble).toOption
          } yield Record(id, cols(1), value)
      }
    }
  }
}
```

---

## Ключевое

- **split(",")** или **split(";")** — разбиение строки по разделителю; результат — Array[String].
- Первая строка — заголовки; **rows.head** / **rows.tail** для отделения данных.
- **header.zip(values).toMap** — преобразование строки в Map[String, String] по именам столбцов.
- Преобразование в case class: разбор по индексу или по Map; **Try().toOption** для безопасного парсинга чисел.
- Простой split не обрабатывает кавычки и переносы внутри полей; для сложного CSV — библиотеки.

В [§7.4](chapter-07-04.md) мы рассмотрим сериализацию в JSON и сериализацию для Spark.
