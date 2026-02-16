# §6.3 Case-классы

В [§6.2](chapter-06-02.md) мы разобрали объекты и объект-компаньон. **Case class** (кейс-класс) — особый вид класса для неизменяемых данных: компилятор автоматически генерирует **equals**, **hashCode**, **toString**, метод **copy** и объект-компаньон с **apply**. Case class — стандартный выбор для моделей данных, записей и сущностей; он удобен в **pattern matching** ([§3.2](chapter-03-02.md)). В [§6.4](chapter-06-04.md) рассмотрим трейты. См. [Глоссарий](glossary.md).

---

## 6.3.1. Объявление и создание

Синтаксис: **case class Имя(параметры)**. Создание — **без new**; компилятор добавляет метод `apply` в объект-компаньон:

```scala
case class User(id: Int, name: String, email: String)

val u = User(1, "Alice", "alice@example.com")
```

Параметры case class по умолчанию — **val**: они становятся неизменяемыми полями. Доступ — через имя поля:

```scala
u.id      // 1
u.name    // "Alice"
u.email = "new@mail.com"   // ошибка: val нельзя переназначить
```

---

## 6.3.2. equals и hashCode

Компилятор генерирует **equals** и **hashCode** по **структуре** (значениям полей), а не по ссылке. Два экземпляра с одинаковыми полями считаются равными:

```scala
val u1 = User(1, "Alice", "alice@example.com")
val u2 = User(1, "Alice", "alice@example.com")

u1 == u2   // true
u1.equals(u2)   // true
```

Это важно для использования case class в Set, Map (как ключ) и при сравнении записей данных. Обычный класс по умолчанию сравнивает по ссылке — для него пришлось бы переопределять equals и hashCode вручную.

---

## 6.3.3. toString

Компилятор генерирует **toString**, который выводит имя класса и значения полей:

```scala
User(1, "Alice", "alice@example.com").toString
// "User(1,Alice,alice@example.com)"
```

Удобно для отладки и логирования. В обычном классе пришлось бы писать `override def toString` самостоятельно.

---

## 6.3.4. copy — копия с изменениями

Case class получает метод **copy** для создания копии с изменением отдельных полей:

```scala
val u = User(1, "Alice", "alice@example.com")
val u2 = u.copy(email = "new@example.com")
// u2: User(1, "Alice", "new@example.com")

val u3 = u.copy(name = "Bob", id = 2)
```

Исходный экземпляр не меняется (неизменяемость). Поля, не указанные в `copy`, берутся из исходного экземпляра. Это типичный способ «обновления» неизменяемых структур данных.

---

## 6.3.5. Pattern matching

Case class удобно сопоставлять в **match**: образец разбирает значения полей и связывает их с переменными:

```scala
def describe(u: User): String = u match {
  case User(id, name, _) if id < 0 => s"Invalid user: $name"
  case User(_, "Alice", _) => "Found Alice"
  case User(id, name, email) => s"User $name (id=$id)"
}
```

В `case User(id, name, email)` переменные `id`, `name`, `email` получают значения полей. Можно использовать подстановку `_` для полей, которые не нужны, и guard (`if`) для дополнительных условий.

---

## 6.3.6. Case class как модель данных

Case class — основной способ моделировать сущности в Scala: строка таблицы, запись CSV, объект JSON:

```scala
case class Order(id: Long, customerId: Int, amount: Double, status: String)

val orders = List(
  Order(1, 101, 99.99, "completed"),
  Order(2, 102, 149.50, "pending")
)

orders.filter(_.status == "completed").map(_.amount).sum
```

Использование в коллекциях, с map, filter и pattern matching делает код выразительным. Case class также хорошо сочетается с Spark Dataset и сериализацией JSON ([§7.4](chapter-07-04.md)).

---

## 6.3.7. var в case class (не рекомендуется)

Параметры case class по умолчанию — **val**. Можно объявить **var**, но это противоречит идее неизменяемости и ухудшает предсказуемость; в данных обычно избегают:

```scala
case class MutableUser(id: Int, var name: String)   // не рекомендуется
```

---

## 6.3.8. Типичные ошибки

- **Попытка изменить поле** — параметры case class по умолчанию val; для «обновления» используйте `copy`.
- **Сравнение через == с обычным классом** — для обычного класса == сравнивает по ссылке; для case class — по структуре.
- **Использовать обычный class вместо case class** для моделей данных — case class даёт equals, hashCode, toString, copy и удобный pattern matching без лишнего кода.

---

## Ключевое

- **case class** — неизменяемая модель данных; компилятор генерирует equals, hashCode, toString, copy и apply (создание без new).
- Параметры — **val** по умолчанию; создание: `User(1, "Alice", "a@b.com`)` — без new.
- **equals** и **hashCode** — по структуре (значения полей).
- **copy** — копия с изменением отдельных полей; исходный экземпляр не меняется.
- **Pattern matching**: `case User(id, name, email) => ...` — разбор полей в match.
- Case class — стандарт для записей, сущностей и работы с данными в Scala.

В [§6.4](chapter-06-04.md) мы рассмотрим трейты: интерфейсы и смешиваемое поведение.
