# §8.2 RDD и типы

В [§8.1](chapter-08-01.md) мы разобрали SparkSession и DataFrame. **RDD** (Resilient Distributed Dataset) — базовая абстракция Spark: распределённая неизменяемая коллекция с отложенным вычислением. В Scala RDD **типизирован**: `RDD[String]`, `RDD[(String, Int)]` — компилятор знает тип элементов. В этом разделе: создание RDD, типизированные RDD и методы map, filter, reduceByKey. В [§8.3](chapter-08-03.md) рассмотрим Dataset API. См. [Глоссарий](glossary.md).

---

## 8.2.1. RDD как распределённая коллекция

**RDD** — распределённая коллекция элементов, разбитая на **партиции**; каждая партиция обрабатывается на отдельном узле. RDD неизменяем: преобразования (map, filter) возвращают новый RDD. Вычисления **ленивые** — выполняются только при вызове действия (action): collect, count, save и т.д.

Доступ к SparkContext для RDD — через **spark.sparkContext** (или переменная `sc` в spark-shell).

---

## 8.2.2. Создание RDD

**Из локальной коллекции** — **parallelize**:

```scala
val sc = spark.sparkContext
val rdd = sc.parallelize(Seq(1, 2, 3, 4, 5))
val rddPairs = sc.parallelize(Seq(("a", 1), ("b", 2), ("a", 3)))
```

**Из файла** — **textFile** (каждая строка — элемент):

```scala
val lines = sc.textFile("data.txt")
// lines: RDD[String]
```

---

## 8.2.3. Типизированные RDD

В Scala RDD имеет параметр типа: **RDD[A]**. Компилятор проверяет совместимость преобразований:

```scala
val rdd: RDD[String] = sc.textFile("data.txt")
val words: RDD[String] = rdd.flatMap(_.split(" "))
val pairs: RDD[(String, Int)] = words.map(w => (w, 1))
val counts: RDD[(String, Int)] = pairs.reduceByKey(_ + _)
```

Тип `RDD[(String, Int)]` — RDD пар (ключ, значение); такие RDD поддерживают **reduceByKey**, **groupByKey** и другие операции над парами.

---

## 8.2.4. map и flatMap

**map(f)** — преобразует каждый элемент:

```scala
val nums: RDD[Int] = sc.parallelize(Seq(1, 2, 3, 4, 5))
val doubled = nums.map(_ * 2)
```

**flatMap(f)** — применяет функцию, возвращающую коллекцию, и «разворачивает» результат в плоский RDD:

```scala
val lines: RDD[String] = sc.textFile("data.txt")
val words = lines.flatMap(line => line.split(" "))
```

---

## 8.2.5. filter

**filter(p)** — оставляет только элементы, для которых предикат истинен:

```scala
val even = nums.filter(_ % 2 == 0)
val nonEmpty = lines.filter(_.trim.nonEmpty)
```

---

## 8.2.6. reduceByKey

Для RDD пар **reduceByKey(f)** объединяет значения с одинаковым ключом. Функция `f` должна быть ассоциативной и коммутативной (для корректности при распределённом выполнении):

```scala
val pairs: RDD[(String, Int)] = words.map(w => (w, 1))
val counts = pairs.reduceByKey(_ + _)
```

В отличие от **groupByKey**, reduceByKey выполняет свёртку на каждой партиции до shuffle, что эффективнее по памяти и сети.

---

## 8.2.7. Трансформации и действия

**Трансформации** (map, filter, flatMap, reduceByKey) — ленивые; возвращают новый RDD. **Действия** (actions) запускают вычисление:

- **collect()** — собрать все элементы на драйвер (для больших RDD опасно по памяти);
- **count()** — число элементов;
- **take(n)** — первые n элементов;
- **saveAsTextFile(path)** — записать в файлы.

```scala
val result = counts.collect()
counts.saveAsTextFile("output")
```

---

## 8.2.8. Пример: подсчёт слов

```scala
val lines = sc.textFile("input.txt")
val words = lines.flatMap(_.split("\\s+")).filter(_.nonEmpty)
val pairs = words.map(w => (w, 1))
val counts = pairs.reduceByKey(_ + _)
counts.sortBy(_._2, ascending = false).take(10)
```

---

## Ключевое

- **RDD** — распределённая неизменяемая коллекция; вычисления ленивые.
- **Типизированные RDD:** RDD[String], RDD[(String, Int)] — компилятор проверяет типы.
- **Создание:** sc.parallelize(seq), sc.textFile(path).
- **map**, **flatMap**, **filter** — трансформации; **reduceByKey** — для RDD пар, свёртка по ключу.
- **collect**, **count**, **take**, **saveAsTextFile** — действия, запускающие вычисление.

В [§8.3](chapter-08-03.md) мы рассмотрим Dataset API: типобезопасность и Encoders для case class.
