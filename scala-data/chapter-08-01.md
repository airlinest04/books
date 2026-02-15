# §8.1 Spark на Scala

В [§7.4](chapter-07-04.md) мы коснулись сериализации для Spark. Эта глава посвящена **связи Scala и Spark**: как запускать jobs на Scala, создавать DataFrame и Dataset, и почему Scala остаётся родным языком Spark. **SparkSession** — точка входа в Spark SQL; через неё создают DataFrame и Dataset. В этом разделе: создание SparkSession, базовое создание DataFrame и типичный контекст использования. В [§8.2](chapter-08-02.md) рассмотрим RDD. См. [Глоссарий](glossary.md).

---

## 8.1.1. SparkSession — точка входа

**SparkSession** объединяет возможности SparkContext (RDD) и старого SQLContext (DataFrame). Создание через билдер:

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("MySparkJob")
  .master("local[*]")
  .getOrCreate()
```

- **appName** — имя приложения (отображается в UI);
- **master** — URL мастер-узла: **"local"** или **"local[*]"** для локального режима (число потоков = число ядер), **"spark://host:7077"** для кластера, **"yarn"** для YARN;
- **getOrCreate()** — переиспользует существующую сессию, если есть, иначе создаёт новую.

В production `master` обычно задают через `spark-submit`; в коде часто опускают, если запуск только через submit.

---

## 8.1.2. Создание DataFrame

DataFrame — распределённая таблица с именованными столбцами. Создание из коллекции:

```scala
import spark.implicits._

val data = Seq(
  ("Alice", 30),
  ("Bob", 25),
  ("Carol", 28)
)
val df = spark.createDataFrame(data).toDF("name", "age")
```

Через **toDF** задают имена столбцов. С case class и implicits:

```scala
case class Person(name: String, age: Int)
val people = Seq(Person("Alice", 30), Person("Bob", 25))
val df = people.toDF()
```

Чтение из файла:

```scala
val df = spark.read.csv("data.csv")
val dfWithHeader = spark.read.option("header", "true").csv("data.csv")
val dfJson = spark.read.json("data.json")
```

**spark.read** возвращает DataFrameReader; методы **csv**, **json**, **parquet** и др. загружают данные.

---

## 8.1.3. Базовые операции DataFrame

DataFrame поддерживает выбор столбцов, фильтрацию, агрегацию:

```scala
df.select("name", "age")
df.filter($"age" > 25)
df.groupBy("name").count()
df.show()
```

Для работы с колонками через `$"col"` нужен **import spark.implicits._**. Метод **show()** выводит строки в консоль (удобно при отладке).

---

## 8.1.4. Типичный контекст: Scala-скрипты и jobs

Spark на Scala используют в двух основных сценариях:

**1. Скрипты и интерактивный анализ** — REPL (spark-shell) или scala-cli для быстрых экспериментов. Локальный режим (`master("local[*]")`), чтение файлов, преобразования, вывод или запись результата.

**2. Production jobs** — JAR-приложения, запускаемые через **spark-submit** на кластере (YARN, Kubernetes, standalone). Конфигурация передаётся через аргументы submit; в коде создают SparkSession без жёстко заданного master.

```bash
spark-submit --class MyMain --master yarn --deploy-mode cluster myjob.jar
```

Типичный пайплайн: чтение (CSV, Parquet, Kafka) → преобразования (filter, map, join, agg) → запись (файл, таблица, Kafka). Подробнее о сборке и зависимостях — в [§8.4](chapter-08-04.md).

---

## 8.1.5. Остановка сессии

По окончании работы (особенно в скриптах) сессию останавливают:

```scala
spark.stop()
```

В долгоживущих приложениях и spark-shell это делают при завершении; в коротких job — после записи результата.

---

## 8.1.6. Связь с RDD и Dataset

- **RDD** — низкоуровневая распределённая коллекция; API в стиле map/filter/reduce. SparkSession даёт доступ через **spark.sparkContext**.
- **DataFrame** — таблица со схемой; оптимизируется Catalyst. По сути **Dataset[Row]** — строки без статического типа.
- **Dataset[T]** — типобезопасная обёртка над DataFrame; требуется Encoder для типа T. См. [§8.3](chapter-08-03.md).

---

## Ключевое

- **SparkSession** — точка входа; создание через builder: appName, master (local для локального режима), getOrCreate().
- **DataFrame** — spark.createDataFrame(...).toDF(...), spark.read.csv/json(...); для колонок нужен import spark.implicits._
- Типичный контекст: локальные скрипты (spark-shell) и production jobs через spark-submit.
- spark.stop() — остановка сессии по окончании работы.

В [§8.2](chapter-08-02.md) мы рассмотрим RDD: типизированные коллекции и методы map, filter, reduceByKey.
