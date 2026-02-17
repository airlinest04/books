# §9.1 Конфигурация и параметры

В [§8.5](chapter-08-05.md) мы завершили главу о Spark. Переходим к **типовым паттернам ETL и пайплайнов**: как организовывать конфигурацию, цепочки преобразований и обработку ошибок. Конфигурация ETL-job обычно включает пути к файлам, параметры подключения и опции. В этом разделе: загрузка конфига из файла и переменных окружения, **case class** для конфигурации и передача путей и опций в job. В [§9.2](chapter-09-02.md) рассмотрим цепочку преобразований. См. [Глоссарий](glossary.md).

---

## 9.1.1. Аргументы командной строки

Самый простой способ передать параметры — аргументы метода **main**:

```scala
def main(args: Array[String]): Unit = {
  val inputPath  = args.lift(0).getOrElse("input.csv")
  val outputPath = args.lift(1).getOrElse("output")
  val limit      = args.lift(2).flatMap(s => scala.util.Try(s.toInt).toOption).getOrElse(1000)
}
```

**args.lift(i)** возвращает `Option[String]` — при отсутствии элемента даёт `None` вместо исключения. **getOrElse** задаёт значение по умолчанию.

---

## 9.1.2. Переменные окружения

В Scala переменные окружения доступны через **sys.env** (Map[String, String]):

```scala
val inputPath  = sys.env.getOrElse("INPUT_PATH", "input.csv")
val dbUrl      = sys.env.get("DB_URL")   // Option[String]
```

Переменные окружения удобны для секретов и параметров, которые не хочется передавать в командной строке (пароли, ключи API). На кластере их задают в конфигурации YARN/Kubernetes или в скрипте запуска.

---

## 9.1.3. Case class для конфигурации

Вместо разбросанных переменных конфигурацию оформляют в **case class**:

```scala
case class JobConfig(
  inputPath:  String,
  outputPath: String,
  limit:      Int = 1000
)

def loadConfig(args: Array[String]): JobConfig = JobConfig(
  inputPath  = args.lift(0).getOrElse("input.csv"),
  outputPath = args.lift(1).getOrElse("output"),
  limit      = args.lift(2).flatMap(s => scala.util.Try(s.toInt).toOption).getOrElse(1000)
)

def main(args: Array[String]): Unit = {
  val config = loadConfig(args)
  runJob(config)
}
```

Так конфигурация типобезопасна, а значения по умолчанию заданы в одном месте.

---

## 9.1.4. Конфиг из файла: Typesafe Config

Библиотека **Typesafe Config** (Lightbend Config) — стандартный способ работы с конфигом в JVM. Зависимость в sbt:

```scala
libraryDependencies += "com.typesafe" % "config" % "1.4.3"
```

Загрузка:

```scala
import com.typesafe.config.ConfigFactory

val config = ConfigFactory.load()
val inputPath  = config.getString("job.inputPath")
val outputPath = config.getString("job.outputPath")
val limit      = config.getInt("job.limit")
```

Файл **application.conf** (в classpath или по пути из `-Dconfig.file=...`):

```hocon
job {
  inputPath = "input.csv"
  outputPath = "output"
  limit = 1000
}
```

Подстановка переменных окружения: `inputPath = ${?INPUT_PATH}` — значение берётся из `INPUT_PATH`, если задано.

---

## 9.1.5. Объединение источников

Конфигурацию часто собирают из нескольких источников: аргументы и переменные окружения имеют приоритет над файлом:

```scala
def loadConfig(args: Array[String]): JobConfig = {
  val config = ConfigFactory.load()
  JobConfig(
    inputPath  = args.lift(0).orElse(sys.env.get("INPUT_PATH")).getOrElse(config.getString("job.inputPath")),
    outputPath = args.lift(1).orElse(sys.env.get("OUTPUT_PATH")).getOrElse(config.getString("job.outputPath")),
    limit      = args.lift(2).flatMap(s => scala.util.Try(s.toInt).toOption)
      .getOrElse(config.getInt("job.limit"))
  )
}
```

Порядок приоритета: args → env → config file → defaults.

---

## 9.1.6. Типичные ошибки

- **args(i) без проверки** — при пустом args вызов args(0) вызовет исключение; использовать **args.lift(i)**.
- **Секреты в коде и в аргументах** — пароли и ключи передавать через переменные окружения или секрет-менеджеры.
- **Жёстко заданные пути** — выносить в конфиг или аргументы для переносимости между средами.

---

## Ключевое

- **Аргументы:** args.lift(i).getOrElse(default) — безопасное извлечение с значением по умолчанию.
- **Переменные окружения:** sys.env.get("KEY"), sys.env.getOrElse("KEY", default).
- **Case class** для конфигурации — типобезопасность и единая точка описания.
- **Typesafe Config** — загрузка из application.conf; поддержка HOCON, переменных окружения и подстановок.
- Приоритет: аргументы → env → файл → defaults.

В [§9.2](chapter-09-02.md) мы рассмотрим цепочку преобразований: read → transform → write и обработку ошибок.
