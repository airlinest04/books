# §9.5 Логирование

В [§9.4](chapter-09-04.md) мы разобрали тестируемость. **Логирование** в ETL-пайплайне помогает отслеживать прогресс, находить ошибки и анализировать производительность. В Scala обычно используют **SLF4J** как фасад и **Logback** как реализацию. В этом разделе: выбор библиотеки, настройка и логирование старта/завершения шагов и ошибок. Глава 9 завершается; далее — [Глоссарий](glossary.md) и приложения. См. [Глоссарий](glossary.md).

---

## 9.5.1. SLF4J и Logback

**SLF4J** (Simple Logging Facade for Java) — фасад для логирования; приложение зависит только от API SLF4J, а реализация (Logback, Log4j и др.) подключается на этапе сборки.

**Logback** — распространённая реализация SLF4J; используется по умолчанию в Spark и многих проектах на JVM.

Зависимости в sbt:

```scala
libraryDependencies ++= Seq(
  "org.slf4j" % "slf4j-api" % "2.0.9",
  "ch.qos.logback" % "logback-classic" % "1.4.11"
)
```

logback-classic подтягивает slf4j-api; он реализует интерфейс SLF4J.

---

## 9.5.2. Получение логгера

```scala
import org.slf4j.{Logger, LoggerFactory}

val logger: Logger = LoggerFactory.getLogger(getClass)
```

Или с именем класса явно:

```scala
val logger = LoggerFactory.getLogger("EtlPipeline")
```

Обычно логгер объявляют как поле объекта или класса. В Spark-приложениях Spark сам настраивает логирование; можно получить логгер и использовать его в своём коде.

---

## 9.5.3. Уровни и методы

Основные уровни (от менее к более критичным): **trace**, **debug**, **info**, **warn**, **error**:

```scala
logger.debug("Отладочное сообщение")
logger.info("Запуск пайплайна")
logger.warn("Подозрительное значение, пропуск")
logger.error("Ошибка при записи", exception)
```

Метод **error** может принимать исключение вторым аргументом — в стек вызовов попадает в лог. Для интерполяции строки в SLF4J используют `{}` как плейсхолдеры:

```scala
logger.info("Обработано {} записей за {} мс", count, duration)
```

---

## 9.5.4. Логирование в ETL-пайплайне

Типичные точки логирования:

- **Старт пайплайна** — входные параметры, конфигурация;
- **Старт и конец каждого шага** — имя шага, объём данных, длительность;
- **Ошибки** — при Failure, исключениях, невалидных данных.

```scala
def runPipeline(config: JobConfig): Try[Unit] = {
  logger.info("Starting pipeline: input={}, output={}", config.inputPath, config.outputPath)
  val start = System.currentTimeMillis()

  val result = for {
    raw <- read(config.inputPath)
    _   = logger.info("Read {} records", raw.size)
    transformed = transform(raw)
    _   <- write(config.outputPath, transformed)
  } yield {
    logger.info("Write completed")
    ()
  }

  result match {
    case Success(_) =>
      logger.info("Pipeline completed in {} ms", System.currentTimeMillis() - start)
    case Failure(e) =>
      logger.error("Pipeline failed", e)
  }
  result
}
```

---

## 9.5.5. Конфигурация Logback

Файл **logback.xml** в `src/main/resources` задаёт уровень логирования, формат и вывод (консоль, файл):

```xml
<configuration>
  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger - %msg%n</pattern>
    </encoder>
  </appender>
  <root level="INFO">
    <appender-ref ref="CONSOLE" />
  </root>
</configuration>
```

Уровень **INFO** по умолчанию; для отладки можно установить DEBUG для конкретного пакета. Spark настраивает свой логгер; при необходимости уровень для org.apache.spark задают отдельно.

---

## 9.5.6. Практические рекомендации

- Логировать старт и завершение шагов; при долгих операциях — прогресс или промежуточные счётчики.
- При ошибках передавать исключение в logger.error для записи стека.
- Не логировать чувствительные данные (пароли, токены); объёмные структуры — только сводно (число записей, размер).
- В production уровень DEBUG обычно отключают из-за объёма логов.

---

## Ключевое

- **SLF4J** — фасад; **Logback** — реализация. Зависимости: slf4j-api, logback-classic.
- **LoggerFactory.getLogger(...)** — получение логгера. Методы: debug, info, warn, error.
- В ETL логировать: старт/конец пайплайна, старт/конец шагов, ошибки с исключениями.
- **logback.xml** — настройка уровней и формата вывода.

Глава 9 завершена. Далее — [Глоссарий](glossary.md) и приложения книги.
