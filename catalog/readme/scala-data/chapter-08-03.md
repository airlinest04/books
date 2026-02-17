# §8.3 Dataset API

В [§8.2](chapter-08-02.md) мы разобрали RDD. **Dataset[T]** — типизированный API Spark: распределённая коллекция с известным типом элементов. Dataset объединяет типобезопасность RDD с оптимизациями Catalyst (как у DataFrame). В Scala для создания Dataset из case class нужны **Encoders**; они подставляются автоматически при `import spark.implicits._`. В этом разделе: Dataset[T], Encoders для case class и связь с DataFrame. В [§8.4](chapter-08-04.md) рассмотрим зависимости и сборку. См. [Глоссарий](glossary.md).

---

## 8.3.1. Dataset[T] — типизированная коллекция

**Dataset[T]** — распределённая коллекция элементов типа T. Операции map, filter, flatMap принимают функции с типизированными аргументами; компилятор проверяет корректность до выполнения:

```scala
import spark.implicits._

case class User(id: Int, name: String, age: Int)
val users: Dataset[User] = spark.createDataset(Seq(
  User(1, "Alice", 30),
  User(2, "Bob", 25)
))

val adults = users.filter(_.age >= 18)
val names = users.map(_.name)
```

В `filter(_.age >= 18)` компилятор знает, что элемент — `User`, и обращение к полю `age` проверяется статически. Ошибка в имени поля (например, `_.ag`) будет найдена на этапе компиляции.

---

## 8.3.2. Encoders для case class

Spark должен уметь сериализовать и десериализовать элементы Dataset для передачи между узлами. Для этого используется **Encoder[T]**. Для case class и примитивов Encoder выводится автоматически при **import spark.implicits._**:

```scala
import spark.implicits._

case class Record(id: Long, value: Double)
val ds = spark.createDataset(Seq(Record(1, 10.0)))
// Encoder[Record] подставлен неявно
```

Без импорта implicits компилятор выдаст: «Unable to find encoder for type stored in a Dataset». Implicits предоставляют Encoders для стандартных типов и для case class (через макросы).

---

## 8.3.3. DataFrame как Dataset[Row]

**DataFrame** — это **Dataset[Row]**, где Row — неструктурированная строка с доступом по индексу или имени столбца. Dataset[User] типобезопасен; Dataset[Row] (DataFrame) — нет: обращение к полям идёт по строковым именам, ошибки обнаруживаются в runtime.

```scala
val df: DataFrame = spark.read.csv("data.csv")
val ds: Dataset[User] = spark.createDataset(users)

df.filter($"age" > 25)     // колонка по имени, проверка в runtime
ds.filter(_.age > 25)      // поле по типу, проверка при компиляции
```

---

## 8.3.4. Преобразование между Dataset и DataFrame

**Dataset.toDF()** — преобразование в DataFrame (теряется типизация, остаётся Dataset[Row]):

```scala
val ds: Dataset[User] = ...
val df: DataFrame = ds.toDF()
```

**DataFrame.as[T]** — преобразование в Dataset[T] при наличии Encoder и совпадении схемы:

```scala
val df = spark.read.option("header", "true").csv("users.csv")
val ds = df.as[User]
```

Для `as[User]` нужен Encoder[User]; при `import spark.implicits._` он будет подставлен. Схема CSV должна соответствовать полям case class (имена и типы).

---

## 8.3.5. Операции Dataset

Dataset поддерживает те же операции, что и DataFrame (select, filter, groupBy, join), плюс типобезопасные map, flatMap, filter с лямбдами над элементом:

```scala
val users: Dataset[User] = ...
users.filter(_.age >= 18).map(u => (u.name, u.age))
users.groupByKey(_.name).count()
```

**groupByKey** — аналог groupBy для Dataset; принимает функцию извлечения ключа. Результат — KeyValueGroupedDataset.

---

## 8.3.6. Чтение с типизацией

При чтении JSON с известной схемой можно сразу получить Dataset[T]:

```scala
val ds = spark.read.json("users.json").as[User]
```

Для CSV и других форматов обычно читают в DataFrame, приводят схему к нужному виду и вызывают as[T].

---

## 8.3.7. Когда использовать Dataset

- **Dataset[T]** — когда важна типобезопасность: case class как модель данных, проверка полей при компиляции, рефакторинг.
- **DataFrame** — когда схема динамическая, при работе через SQL или при чтении данных без заранее известной модели.
- **RDD** — когда нужны низкоуровневые операции или типы, для которых нет Encoder (например, произвольные Java-объекты с Kryo).

---

## Ключевое

- **Dataset[T]** — типизированная распределённая коллекция; map, filter с проверкой типов на этапе компиляции.
- **Encoder[T]** — нужен Spark для сериализации; для case class подставляется автоматически при **import spark.implicits._**.
- **DataFrame = Dataset[Row]**; toDF() и as[T] для преобразований.
- Dataset объединяет типобезопасность RDD с оптимизациями Catalyst.

В [§8.4](chapter-08-04.md) мы рассмотрим зависимости, сборку sbt и запуск job через spark-submit.
