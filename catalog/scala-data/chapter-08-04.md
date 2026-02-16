# §8.4 Зависимости и сборка

В [§8.3](chapter-08-03.md) мы разобрали Dataset API. Чтобы писать Spark-приложения на Scala, нужно добавить зависимости в проект и собрать JAR для запуска через **spark-submit**. В этом разделе: добавление spark-core и spark-sql в sbt, пример минимального job и команда spark-submit. В [§8.5](chapter-08-05.md) рассмотрим неявные преобразования (implicits). См. [Глоссарий](glossary.md).

---

## 8.4.1. Зависимости Spark в build.sbt

В **build.sbt** добавляют spark-core и spark-sql:

```scala
val sparkVersion = "3.5.0"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core" % sparkVersion % "provided",
  "org.apache.spark" %% "spark-sql"  % sparkVersion % "provided"
)
```

Конфигурация **"provided"** означает: зависимость используется при компиляции, но **не включается** в итоговый JAR. Spark уже есть на кластере при запуске через spark-submit; дублирование приведёт к конфликтам версий. Для локального запуска (sbt run) «provided» не подходит — Spark должен быть в classpath. В таком случае временно убирают `% "provided"` или добавляют отдельную конфигурацию для local.

Версия Scala в проекте должна совпадать с той, для которой собран Spark (обычно 2.12 или 2.13). Оператор **%%** подставляет scalaVersion автоматически.

---

## 8.4.2. Пример минимального job

Точка входа — объект с методом main:

```scala
// src/main/scala/SparkJob.scala
import org.apache.spark.sql.SparkSession

object SparkJob {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder()
      .appName("SimpleSparkJob")
      .getOrCreate()

    import spark.implicits._

    val inputPath = args.lift(0).getOrElse("input.csv")
    val outputPath = args.lift(1).getOrElse("output")

    val df = spark.read.option("header", "true").csv(inputPath)
    val result = df.filter($"age" >= 18)
    result.write.mode("overwrite").csv(outputPath)

    spark.stop()
  }
}
```

Пути входного и выходного файла передаются аргументами; при отсутствии — значения по умолчанию.

---

## 8.4.3. Сборка JAR

Для spark-submit нужен JAR с вашим кодом. Сборка «толстого» JAR (fat JAR, assembly) со всеми зависимостями обычно не требуется для Spark: кластер предоставляет Spark и Hadoop. Достаточно JAR только с вашим кодом.

**Пакетная сборка через sbt:**

```bash
sbt package
```

JAR появится в `target/scala-2.13/sparkjob_2.13-0.1.0-SNAPSHOT.jar` (имя и путь зависят от настроек в build.sbt). Spark-зависимости с «provided» в этот JAR не попадают.

Для fat JAR (если нужны сторонние библиотеки) используют плагин **sbt-assembly**; в таком JAR не включают spark-core и spark-sql (остаются provided).

---

## 8.4.4. Запуск через spark-submit

**spark-submit** — утилита для запуска Spark-приложений. Базовая команда:

```bash
spark-submit \
  --class SparkJob \
  --master local[*] \
  target/scala-2.13/sparkjob_2.13-0.1.0-SNAPSHOT.jar \
  input.csv output
```

- **--class** — полное имя класса с методом main (например, `my.package.SparkJob`);
- **--master** — local для локального режима, yarn, spark://host:7077 для кластера;
- далее — путь к JAR и аргументы приложения.

Для YARN:

```bash
spark-submit \
  --class SparkJob \
  --master yarn \
  --deploy-mode cluster \
  sparkjob.jar input.csv output
```

---

## 8.4.5. Локальный запуск через sbt

Для отладки без spark-submit:

```bash
sbt "run input.csv output"
```

Тогда зависимости Spark не должны быть «provided» — иначе runtime не найдёт классы. Можно завести два профиля: один для локального run (без provided), другой для package/spark-submit (с provided). Альтернатива — использовать spark-shell для интерактивных экспериментов.

---

## 8.4.6. Полный build.sbt

```scala
name := "sparkjob"
version := "0.1.0"
scalaVersion := "2.13.12"

val sparkVersion = "3.5.0"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core" % sparkVersion % "provided",
  "org.apache.spark" %% "spark-sql"  % sparkVersion % "provided"
)
```

---

## Ключевое

- **spark-core**, **spark-sql** — основные зависимости; версия должна соответствовать кластеру. Конфигурация **provided** — для сборки под spark-submit.
- Минимальный job: SparkSession, чтение, преобразование, запись; main принимает аргументы (пути и т.д.).
- **sbt package** — сборка JAR; **spark-submit --class ... --master ... jar args** — запуск.
- Для локального sbt run убрать «provided» или использовать отдельный профиль.

В [§8.5](chapter-08-05.md) мы рассмотрим неявные преобразования (implicits): spark.implicits._ и Encoders для case class.
