# §7.4 Сериализация

В [§7.3](chapter-07-03.md) мы разобрали парсинг CSV. **Сериализация** — преобразование объектов в формат для хранения или передачи (JSON, байты) и обратное восстановление. В этом разделе кратко: сериализация в JSON через библиотеки **circe** или **json4s** и сериализация для **Spark** (Kryo, Encoders). Детали работы со Spark — в [Главе 8](chapter-08-01.md). См. [Глоссарий](glossary.md).

---

## 7.4.1. Зачем нужна сериализация

Сериализация нужна для:

- **Обмена данными** — API, очереди сообщений (Kafka), хранение в файлах и базах;
- **Распределённых вычислений** — Spark сериализует данные для передачи между узлами кластера;
- **Кэширования** — сохранение промежуточных результатов на диск или в память.

Формат выбирают по задаче: JSON — человекочитаемый, удобен для API; бинарные форматы (Kryo, Avro, Parquet) — компактнее и быстрее при больших объёмах.

---

## 7.4.2. JSON: библиотека circe

**Circe** — популярная библиотека для работы с JSON в Scala. Зависимости (sbt):

```scala
libraryDependencies ++= Seq(
  "io.circe" %% "circe-core" % "0.14.6",
  "io.circe" %% "circe-generic" % "0.14.6",
  "io.circe" %% "circe-parser" % "0.14.6"
)
```

Сериализация и десериализация case class с автоматическим выводом кодека:

```scala
import io.circe._
import io.circe.generic.auto._
import io.circe.parser._
import io.circe.syntax._

case class User(id: Int, name: String, email: Option[String])

val user = User(1, "Alice", Some("alice@example.com"))
val json = user.asJson.noSpaces
// json: String = """{"id":1,"name":"Alice","email":"alice@example.com"}"""

val decoded = decode[User](json)
// decoded: Either[circe.Error, User] = Right(User(1, "Alice", Some("alice@example.com")))
```

- **asJson** — преобразование в JSON (требует `Encoder`; для case class — автоматически через `generic.auto._`);
- **decode[A]** — разбор JSON в тип A; возвращает `Either[Error, A]`.

---

## 7.4.3. JSON: библиотека json4s

**json4s** — ещё одна библиотека для JSON; часто используется в экосистеме Spark (Spark SQL использует json4s для парсинга JSON-строк). Зависимости:

```scala
libraryDependencies += "org.json4s" %% "json4s-native" % "4.0.7"
// или json4s-jackson для Jackson backend
```

Пример (нативно):

```scala
import org.json4s._
import org.json4s.native.JsonMethods._
import org.json4s.DefaultFormats

case class User(id: Int, name: String)

implicit val formats: DefaultFormats = DefaultFormats
val json = """{"id":1,"name":"Alice"}"""
val user = parse(json).extract[User]
```

**parse** разбирает строку в AST; **extract[T]** восстанавливает объект (использует рефлексию). Для записи — `compact(render(JObject(...)))` или Serialization.write.

---

## 7.4.4. Spark: Kryo

Spark сериализует данные при shuffle, кэшировании и передаче между водителем и исполнителями. По умолчанию используется **Java Serialization**; для производительности рекомендуют **Kryo**:

```scala
import org.apache.spark.SparkConf
import org.apache.spark.sql.SparkSession

val conf = new SparkConf()
  .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
  .registerKryoClasses(Array(classOf[MyCaseClass]))
val spark = SparkSession.builder().config(conf).getOrCreate()
```

Kryo быстрее и даёт более компактный вывод. Для case class и стандартных типов часто достаточно включить Kryo без явной регистрации; для кастомных классов **registerKryoClasses** повышает стабильность.

---

## 7.4.5. Spark: Encoders и Dataset

**Dataset** в Spark требует **Encoder** для сериализации/десериализации записей. Для case class и примитивов Encoder подставляется автоматически при импорте **spark.implicits._**:

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder().getOrCreate()
import spark.implicits._

case class Record(id: Int, value: Double)
val data = Seq(Record(1, 10.0), Record(2, 20.0))
val ds = spark.createDataset(data)
// ds: Dataset[Record]
```

Без `import spark.implicits._` компилятор выдаст ошибку «Unable to find encoder». Encoders используют внутренний механизм Spark (не Kryo напрямую) и оптимизированы для Catalyst. Подробнее — в [§8.3](chapter-08-03.md) и [§8.5](chapter-08-05.md).

---

## 7.4.6. Сводка

| Контекст | Инструмент | Назначение |
|----------|------------|------------|
| API, файлы, Kafka | circe, json4s | JSON ↔ case class |
| Spark RDD, shuffle | KryoSerializer | Бинарная сериализация объектов |
| Spark Dataset | Encoders (implicits) | Сериализация в колоночный формат |

---

## Ключевое

- **Сериализация** — преобразование объектов в формат для хранения/передачи; обратная операция — десериализация.
- **JSON:** circe (Encoder/Decoder, generic.auto) или json4s (parse, extract); для API и обмена данными.
- **Spark Kryo** — `spark.serializer = KryoSerializer`; для RDD и shuffle; при необходимости — registerKryoClasses.
- **Spark Encoders** — для Dataset; `import spark.implicits._` даёт Encoder для case class автоматически.

В [§8.1](chapter-08-01.md) мы начнём главу о связи Scala со Spark: SparkSession и типичный контекст использования.
