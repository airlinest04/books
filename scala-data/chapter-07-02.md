# §7.2 Запись файлов

В [§7.1](chapter-07-01.md) мы разобрали чтение файлов через Source. Запись текста в Scala выполняется через классы **java.io**: **PrintWriter** для простой построчной записи и **FileWriter** / **BufferedWriter** при необходимости более тонкого контроля. Scala не предоставляет собственного API записи — используют Java I/O. В этом разделе: PrintWriter, кодировка, построчная запись и закрытие ресурса. В [§7.3](chapter-07-03.md) рассмотрим парсинг CSV. См. [Глоссарий](glossary.md).

---

## 7.2.1. PrintWriter

**PrintWriter** — удобный класс для записи текста: методы **print**, **println**, **write**. Конструктор принимает путь к файлу и, при необходимости, кодировку:

```scala
import java.io.PrintWriter

val pw = new PrintWriter("output.txt")
pw.println("Первая строка")
pw.println("Вторая строка")
pw.close()
```

**Важно:** после записи нужно вызвать **close()**, иначе буфер может не сброситься на диск и данные будут потеряны. Рекомендуется использовать try-finally или Using.

---

## 7.2.2. Кодировка при записи

Без указания кодировки PrintWriter использует кодировку по умолчанию ОС. Для UTF-8 передают имя кодировки вторым аргументом:

```scala
import java.io.PrintWriter

val pw = new PrintWriter("output.txt", "UTF-8")
pw.println("Текст с кириллицей")
pw.close()
```

Для данных (CSV, JSON, логи) обычно задают **UTF-8** явно, чтобы избежать проблем при переносе между системами.

---

## 7.2.3. Построчная запись

Метод **println(s)** записывает строку и добавляет перевод строки. Для записи без перевода — **print(s)** или **write(s)**:

```scala
val pw = new PrintWriter("output.txt", "UTF-8")
try {
  val lines = List("a", "b", "c")
  lines.foreach(pw.println)
  // или вручную:
  pw.println("заголовок")
  pw.println("строка 1")
} finally {
  pw.close()
}
```

При построчной записи коллекции строк каждая строка попадает в файл на отдельной строке (println добавляет \n).

---

## 7.2.4. Закрытие: try-finally и Using

Закрытие через **try-finally** гарантирует сброс буфера даже при исключении:

```scala
import java.io.PrintWriter

val pw = new PrintWriter("output.txt", "UTF-8")
try {
  pw.println("данные")
} finally {
  pw.close()
}
```

В Scala 2.13+ можно использовать **scala.util.Using**:

```scala
import java.io.PrintWriter
import scala.util.Using

Using.resource(new PrintWriter("output.txt", "UTF-8")) { pw =>
  pw.println("данные")
}
```

---

## 7.2.5. FileWriter и BufferedWriter

Для более низкоуровневой записи используют **FileWriter** (с указанием кодировки через **OutputStreamWriter**) и **BufferedWriter**:

```scala
import java.io._
import java.nio.charset.StandardCharsets

val file = new File("output.txt")
val writer = new BufferedWriter(
  new OutputStreamWriter(new FileOutputStream(file), StandardCharsets.UTF_8)
)
try {
  writer.write("строка 1\n")
  writer.write("строка 2\n")
  writer.flush()   // принудительно сбросить буфер
} finally {
  writer.close()
}
```

**FileWriter** в конструкторе `FileWriter(File, Charset)` (Java 11+) или `FileWriter(File, boolean append, Charset)` позволяет задать кодировку. Для совместимости со старыми версиями Java часто используют связку FileOutputStream + OutputStreamWriter.

---

## 7.2.6. Режим дополнения (append)

Чтобы дописывать в существующий файл, а не перезаписывать его, используют **FileWriter** с флагом append:

```scala
import java.io._

val fw = new FileWriter("output.txt", true)   // true = append
val pw = new PrintWriter(fw)
try {
  pw.println("новая строка в конец")
} finally {
  pw.close()
}
```

PrintWriter(path) и PrintWriter(path, charset) создают файл заново; для append оборачивают FileWriter с append=true.

---

## 7.2.7. Типичные ошибки

- **Забыть close()** — буфер может не сброситься; при исключении данные потеряются. Всегда использовать try-finally или Using.
- **Не указать кодировку** — для не-ASCII символов явно задавать UTF-8.
- **Режим по умолчанию перезаписывает** — новый PrintWriter(path) перезаписывает файл; для дополнения использовать FileWriter с append=true.

---

## Ключевое

- **PrintWriter(path, charset)** — запись текста; **println(s)** — строка с переводом строки; **print(s)**, **write(s)** — без перевода.
- Кодировка: второй аргумент `"UTF-8"`; для данных указывать явно.
- Обязательно **close()** после записи; try-finally или **Using.resource**.
- **FileWriter(path, true)** — режим дополнения (append).
- Для построчной записи коллекции: `lines.foreach(pw.println)`.

В [§7.3](chapter-07-03.md) мы рассмотрим парсинг CSV: разбиение строк, обработка заголовка и преобразование в case class.
