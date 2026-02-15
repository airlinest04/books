# §7.1 Чтение файлов

В [§6.5](chapter-06-05.md) мы завершили главу об иммутабельности и чистых функциях. Переходим к **работе с данными**: файлы, потоки, сериализация. Чтение текстового файла в Scala выполняется через **scala.io.Source**. В этом разделе: Source.fromFile, построчное чтение через getLines, задание кодировки и **обязательное закрытие** ресурса (try-finally или Using). В [§7.2](chapter-07-02.md) рассмотрим запись файлов. См. [Глоссарий](glossary.md).

---

## 7.1.1. Source.fromFile

**Source.fromFile(path)** открывает файл и возвращает **BufferedSource** — источник для чтения символов или строк. Путь — строка (относительный или абсолютный):

```scala
import scala.io.Source

val source = Source.fromFile("data.txt")
```

По умолчанию используется кодировка платформы. Для явной кодировки передают второй аргумент или используют неявный **Codec**:

```scala
import scala.io.Codec

val source = Source.fromFile("data.txt")(Codec.UTF8)
// или
val source = Source.fromFile("data.txt", "UTF-8")
```

**Важно:** Source **не закрывает** файл автоматически. После чтения нужно вызвать **source.close()**, иначе дескриптор останется открытым.

---

## 7.1.2. Построчное чтение: getLines

Метод **getLines()** возвращает **Iterator[String]** — итератор по строкам файла (без символа перевода строки в конце):

```scala
val source = Source.fromFile("data.txt")(Codec.UTF8)
val lines = source.getLines()
lines.foreach(println)
source.close()
```

Итератор одноразовый: после полного обхода он пуст. Если нужно использовать строки несколько раз, преобразуйте в коллекцию: **lines.toList** (но тогда весь файл загрузится в память).

---

## 7.1.3. Чтение всего файла

**source.mkString** — читает весь файл в одну строку. **source.getLines().mkString("\n")** — то же с явным разделителем строк:

```scala
val source = Source.fromFile("data.txt")(Codec.UTF8)
val content = source.mkString
source.close()
```

Для больших файлов предпочтительно построчное чтение через getLines, чтобы не загружать всё в память.

---

## 7.1.4. Кодировка

Текстовые файлы (CSV, JSON, логи) часто в **UTF-8**. Если не указать кодировку, используется системная по умолчанию; при переносе между ОС возможны ошибки или искажённые символы.

```scala
import scala.io.Codec

Source.fromFile("data.txt")(Codec.UTF8)
Source.fromFile("data.txt", "UTF-8")
Source.fromFile("data.txt")(Codec("Windows-1251"))   // для старых файлов в CP1251
```

При неверной кодировке возможны **MalformedInputException** или «кракозябры» в строках. Для обмена данными рекомендуется UTF-8.

---

## 7.1.5. Закрытие ресурса: try-finally

Рекомендуемый способ — гарантировать закрытие через **try-finally**:

```scala
val source = Source.fromFile("data.txt")(Codec.UTF8)
try {
  for (line <- source.getLines()) {
    println(line)
  }
} finally {
  source.close()
}
```

Блок `finally` выполняется в любом случае — и при нормальном завершении, и при исключении. Так дескриптор файла не «зависнет» при ошибке.

---

## 7.1.6. scala.util.Using (Scala 2.13+)

В Scala 2.13 и 3 доступен **scala.util.Using** для автоматического управления ресурсами:

```scala
import scala.io.Source
import scala.io.Codec
import scala.util.Using

val result = Using(Source.fromFile("data.txt")(Codec.UTF8)) { source =>
  source.getLines().toList
}
// result: Try[List[String]] — Either успех или исключение

// Или Using.resource — бросает исключение при ошибке
Using.resource(Source.fromFile("data.txt")(Codec.UTF8)) { source =>
  source.getLines().foreach(println)
}
```

**Using.resource** закрывает источник после выхода из блока (в том числе при исключении). Использовать Using, когда доступна соответствующая версия Scala.

---

## 7.1.7. Обработка отсутствующего файла

При несуществующем пути **Source.fromFile** выбрасывает **FileNotFoundException** (или **NoSuchFileException** в зависимости от реализации). Обработка через try-catch:

```scala
import scala.io.Source
import scala.io.Codec
import scala.util.{Try, Success, Failure, Using}

val path = "data.txt"
Try(Source.fromFile(path)(Codec.UTF8)) match {
  case Success(source) =>
    try {
      source.getLines().foreach(println)
    } finally source.close()
  case Failure(e) =>
    println(s"Ошибка: ${e.getMessage}")
}
```

Можно использовать **Using** с `Try` или **Option** для более лаконичной обработки.

---

## 7.1.8. Типичные ошибки

- **Забыть close()** — Source не закрывается сам; при длительной работе приложения возможна утечка дескрипторов. Всегда использовать try-finally или Using.
- **Повторное использование Iterator** — getLines() возвращает одноразовый итератор; после обхода он пуст. Для повторного доступа сохранить результат в список: `getLines().toList`.
- **Не указать кодировку** — для файлов с не-ASCII символами явно задавать UTF-8 или нужную кодировку.

---

## Ключевое

- **Source.fromFile(path)** — открытие файла; второй аргумент или **Codec.UTF8** — кодировка.
- **getLines()** — Iterator[String] по строкам; итератор одноразовый.
- **source.mkString** — чтение всего файла в строку. Для больших файлов — getLines.
- **Обязательно закрывать:** try-finally с source.close() или **Using.resource** (Scala 2.13+).
- Для данных рекомендуется кодировка UTF-8.

В [§7.2](chapter-07-02.md) мы рассмотрим запись файлов: PrintWriter и построчная запись.
