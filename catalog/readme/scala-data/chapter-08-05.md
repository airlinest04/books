# §8.5 Неявные преобразования (implicit)

В [§8.4](chapter-08-04.md) мы разобрали зависимости и сборку. Завершаем главу о Spark темой **неявных преобразований** (implicits). Импорт **spark.implicits._** подставляет неявные значения и методы, необходимые для работы с Dataset и DataFrame в Scala: **Encoders** для case class, синтаксис **$"column"** для колонок и другие преобразования. Без этого импорта многие операции не скомпилируются. В [§9.1](chapter-09-01.md) начнём главу о типовых паттернах ETL: конфигурация и параметры. См. [Глоссарий](glossary.md).

---

## 8.5.1. Что такое implicits в Scala

В Scala **implicit** — механизм, позволяющий компилятору автоматически подставлять аргументы или преобразования. Если метод требует неявный параметр типа `Encoder[T]`, а в области видимости есть `implicit val enc: Encoder[T]`, компилятор подставит его сам — явно передавать не нужно.

Spark широко использует implicits: методы `createDataset`, `toDF`, `as[T]` и др. принимают неявные параметры (Encoder, SparkSession и т.д.). Импорт **spark.implicits._** вносит эти значения в область видимости.

---

## 8.5.2. import spark.implicits._

Одна строка импорта даёт доступ к неявным определениям из **SQLImplicits**:

```scala
val spark = SparkSession.builder().getOrCreate()
import spark.implicits._
```

После этого становятся доступны:

- **Encoders** для примитивов (Int, Long, String и т.д.), кортежей и **case class**;
- преобразование локальной коллекции в **DatasetHolder** (для вызова `.toDS()`, `.toDF()`);
- синтаксис **$"columnName"** для ссылки на колонку DataFrame.

---

## 8.5.3. Encoders для case class

При создании `Dataset[T]` Spark нужен **Encoder[T]** для сериализации и десериализации. Для case class Encoder выводится автоматически через макросы, но он должен быть в области видимости как неявный параметр:

```scala
import spark.implicits._

case class User(id: Int, name: String)
val users = Seq(User(1, "Alice"), User(2, "Bob"))
val ds = spark.createDataset(users)
```

Без `import spark.implicits._` компилятор выдаст: «Unable to find encoder for type stored in a Dataset». Implicits предоставляют `Encoder[User]` неявно.

---

## 8.5.4. Синтаксис $"column" для колонок

В DataFrame ссылка на колонку — объект типа **Column**. Синтаксис **$"name"** — вызов неявного метода, преобразующего строку в Column:

```scala
import spark.implicits._

df.filter($"age" > 25)
df.select($"name", $"age" + 1)
```

Без импорта `$"age"` не скомпилируется. Альтернатива — **col("age")** из `org.apache.spark.sql.functions`:

```scala
import org.apache.spark.sql.functions.col
df.filter(col("age") > 25)
```

---

## 8.5.5. toDF и toDS для локальных коллекций

С implicits локальная коллекция (Seq, List) получает методы **toDF()** и **toDS()**:

```scala
import spark.implicits._

case class Person(name: String, age: Int)
val people = Seq(Person("Alice", 30), Person("Bob", 25))
val ds = people.toDS()
val df = people.toDF()
```

Методы добавляются через неявное преобразование: `Seq[Person]` преобразуется в `DatasetHolder`, у которого есть toDS и toDF.

---

## 8.5.6. Явная передача Encoder

Если по какой-то причине implicits недоступны (например, в отдельном модуле без SparkSession), Encoder можно передать явно:

```scala
import org.apache.spark.sql.Encoders

val ds = spark.createDataset(users)(Encoders.product[User])
```

**Encoders.product[T]** создаёт Encoder для типа, расширяющего Product (кортежи, case class). Для примитивов — Encoders.scalaInt, Encoders.STRING и т.д.

---

## 8.5.7. Типичная ошибка

Забыть **import spark.implicits._** — самая частая причина ошибок «Unable to find encoder» или «value $ is not a member of StringContext». Решение: добавить импорт сразу после создания SparkSession.

---

## Ключевое

- **import spark.implicits._** — вносит неявные Encoders, преобразования и синтаксис $"column" для работы с Dataset и DataFrame.
- **Encoder[T]** для case class подставляется автоматически; без импорта — ошибка компиляции «Unable to find encoder».
- **$"columnName"** — ссылка на колонку DataFrame; требует implicits. Альтернатива — col("columnName").
- **toDS()**, **toDF()** для локальных коллекций — доступны через implicits. Явная передача — Encoders.product[T].

В [§9.1](chapter-09-01.md) мы начнём главу о типовых паттернах ETL: конфигурация, передача параметров и case class для настроек.
