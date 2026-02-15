# §6.3 Типы данных и приведение

В [§6.2](chapter-06-02.md) мы разобрали выборку столбцов и фильтрацию. При работе с данными часто нужно изменить **тип** столбца: например, строка с числом — в целое или дробное, строка с датой — в timestamp. В PySpark типы задаются через **pyspark.sql.types**; приведение выполняется методом **cast()**. В этом разделе рассмотрим основные типы (IntegerType, StringType, TimestampType и др.), использование **cast()** в выражениях и при задании схемы при чтении. В [§6.4](chapter-06-04.md) перейдём к записи данных. См. [Глоссарий](glossary.md).

---

## 6.3.1. Основные типы: pyspark.sql.types

Модуль **pyspark.sql.types** содержит классы типов данных Spark SQL. Часто используемые:

| Тип (класс) | Краткое имя (строка для cast) | Описание |
|-------------|--------------------------------|----------|
| **IntegerType** | `"int"` или `"integer"` | 32-битное целое. |
| **LongType** | `"long"` или `"bigint"` | 64-битное целое. |
| **DoubleType** | `"double"` | 64-битное число с плавающей точкой. |
| **FloatType** | `"float"` | 32-битное число с плавающей точкой. |
| **StringType** | `"string"` | Строка. |
| **BooleanType** | `"boolean"` | Логический тип. |
| **TimestampType** | `"timestamp"` | Дата и время. |
| **DateType** | `"date"` | Только дата (без времени). |
| **DecimalType(precision, scale)** | — | Число фиксированной точности (для денег и точных расчётов). |

При задании схемы при чтении (см. [§6.1](chapter-06-01.md)) используют эти классы: **StructField("col_name", IntegerType(), True)** — столбец с именем col_name, тип целое, nullable=True. См. [§3.2](chapter-03-02.md), [Глоссарий](glossary.md).

---

## 6.3.2. Приведение типа: cast()

Метод **cast()** вызывается у **Column** (результат col("x")) и принимает тип — строку с кратким именем или объект DataType. Возвращает выражение столбца с новым типом. См. [Глоссарий](glossary.md).

```python
from pyspark.sql.functions import col

# Строка с именем типа
df = df.withColumn("age_int", col("age").cast("int"))
df = df.withColumn("amount_double", col("amount_str").cast("double"))
df = df.withColumn("flag", col("flag_str").cast("boolean"))

# Через класс типа
from pyspark.sql.types import LongType, DecimalType
df = df.withColumn("id_big", col("id").cast(LongType()))
df = df.withColumn("price", col("price_str").cast(DecimalType(10, 2)))
```

При несовместимых значениях (например, строка "abc" при приведении к int) Spark обычно подставляет **null** в результирующий столбец; ошибка не выбрасывается по умолчанию. Проверка и замена неверных значений — через условия (when/otherwise) или предварительную очистку. См. [§6.2](chapter-06-02.md), [§6.5](chapter-06-05.md).

---

## 6.3.3. Дата и время: to_date, to_timestamp

Строки с датой и временем приводят к типам **date** и **timestamp** функциями **to_date()** и **to_timestamp()** из **pyspark.sql.functions**. Второй аргумент — формат строки (pattern по правилам Java SimpleDateFormat / совместимый с Spark). См. [Глоссарий](glossary.md).

```python
from pyspark.sql.functions import to_date, to_timestamp

df = df.withColumn("dt", to_date(col("date_str"), "yyyy-MM-dd"))
df = df.withColumn("ts", to_timestamp(col("datetime_str"), "yyyy-MM-dd HH:mm:ss"))
```

Если формат не указан, Spark пытается разобрать строку в стандартном формате; при нестандартном формате результат может быть null. После приведения к date/timestamp можно использовать функции **year()**, **month()**, **dayofmonth()**, **date_add()**, **datediff()** и т.д. См. документацию Spark SQL по функциям даты и времени.

---

## 6.3.4. Схема при чтении: StructType и StructField

При чтении с **явной схемой** (см. [§6.1](chapter-06-01.md)) схема задаётся объектом **StructType**, содержащим список **StructField**. Каждый StructField — имя столбца, тип (класс из types) и флаг nullable. См. [§3.2](chapter-03-02.md), [Глоссарий](glossary.md).

```python
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, LongType, TimestampType

schema = StructType([
    StructField("user_id", LongType(), True),
    StructField("event", StringType(), True),
    StructField("count", IntegerType(), True),
    StructField("created_at", TimestampType(), True),
])
df = spark.read.schema(schema).csv("/path/to/file.csv", header=True)
```

Так типы фиксируются уже при чтении; при необходимости после чтения можно дополнительно приводить столбцы через cast (например, если в файле часть значений в другом формате). См. [Глоссарий](glossary.md).

---

## 6.3.5. Проверка схемы и типов

Текущую схему DataFrame можно посмотреть через **df.printSchema()** или **df.schema** (возвращает StructType). Поле **df.dtypes** возвращает список кортежей (имя_столбца, строка_типа) — удобно для программной проверки. После приведения типа имеет смысл вызвать printSchema() или show() и убедиться, что столбец имеет ожидаемый тип и значения не потеряны (нет лишних null из-за неудачного cast). См. [§5.3](chapter-05-03.md).

---

## Ключевое

- **Типы** задаются в **pyspark.sql.types**: IntegerType, LongType, DoubleType, StringType, BooleanType, TimestampType, DateType, DecimalType и др.; при cast можно использовать строки ("int", "string", "double", "timestamp" и т.д.).
- **cast()** — приведение столбца к другому типу: col("x").cast("int") или .cast(LongType()); несовместимые значения обычно дают null.
- **Дата и время:** to_date(col, format), to_timestamp(col, format) для разбора строк; после приведения доступны функции year, month, date_add и др.
- **Схема при чтении:** StructType([StructField(name, type, nullable), ...]) задаётся в .schema(schema) при read; типы фиксируются с самого начала.
- Проверка схемы — printSchema(), df.schema, df.dtypes; после cast проверять тип и наличие неожиданных null.

В [§6.4](chapter-06-04.md) мы разберём запись данных: форматы, режимы (overwrite, append) и партиционирование partitionBy().
