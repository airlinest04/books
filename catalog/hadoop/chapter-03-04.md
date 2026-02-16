# §3.4 Пример: подсчёт слов (WordCount)

В [§3.3](chapter-03-03.md) мы разобрали структуру программы MapReduce: классы Mapper и Reducer, конфигурацию Job и запуск через `hadoop jar`. В этом разделе — **полный пример**: классический **WordCount** (подсчёт слов в тексте). Map выдаёт для каждого слова пару (слово, 1), Reduce суммирует единицы по слову. Разберём код по шагам и команды запуска. Ограничения модели MapReduce и когда предпочесть другие подходы — в [§3.5](chapter-03-05.md). См. [Глоссарий](glossary.md).

---

## 3.4.1. Постановка задачи WordCount

**WordCount** — задача: по набору текстовых файлов подсчитать, **сколько раз встречается каждое слово**. Вход — файлы в HDFS (например, текстовые строки); выход — пары (слово, количество). См. [Глоссарий](glossary.md).

В терминах MapReduce:

- **Map:** для каждой строки разбить её на слова и для каждого слова выдать пару (слово, 1). Ключ — слово (Text), значение — единица (IntWritable).
- **Shuffle/Sort:** все пары с одинаковым словом собираются вместе.
- **Reduce:** для каждого слова приходит список [1, 1, 1, …]; нужно просуммировать и выдать (слово, сумма).

Это минимальный, но полный пример: есть группировка по ключу (слово) и агрегация (сумма). Ниже — реализация на Java с использованием API `org.apache.hadoop.mapreduce`.

---

## 3.4.2. Класс Mapper для WordCount

Mapper получает строки (ключ — смещение в файле, значение — строка). Разбиваем строку на слова по пробелам и для каждого слова пишем (слово, 1). См. [Глоссарий](glossary.md).

```java
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;
import java.util.StringTokenizer;

public class TokenizerMapper
    extends Mapper<LongWritable, Text, Text, IntWritable> {

  private final static IntWritable one = new IntWritable(1);
  private Text word = new Text();

  @Override
  public void map(LongWritable key, Text value, Context context)
      throws IOException, InterruptedException {
    StringTokenizer itr = new StringTokenizer(value.toString());
    while (itr.hasMoreTokens()) {
      word.set(itr.nextToken());
      context.write(word, one);
    }
  }
}
```

Пояснения:

- **LongWritable, Text** — типы входа (TextInputFormat: смещение строки, сама строка).
- **Text, IntWritable** — типы выхода: слово и единица.
- **value.toString()** — строка; **StringTokenizer** разбивает по пробелам (для реальных задач часто используют регулярные выражения или нормализацию: приведение к нижнему регистру, удаление знаков препинания).
- **context.write(word, one)** — одна пара на каждое слово в строке.

Один и тот же объект **one** переиспользуется для всех пар (IntWritable(1)); объект **word** перезаписывается перед каждым write — так принято для уменьшения числа создаваемых объектов.

---

## 3.4.3. Класс Reducer для WordCount

Reducer получает (слово, список единиц) и суммирует их. См. [Глоссарий](glossary.md).

```java
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

import java.io.IOException;

public class IntSumReducer
    extends Reducer<Text, IntWritable, Text, IntWritable> {

  private IntWritable result = new IntWritable();

  @Override
  public void reduce(Text key, Iterable<IntWritable> values, Context context)
      throws IOException, InterruptedException {
    int sum = 0;
    for (IntWritable val : values) {
      sum += val.get();
    }
    result.set(sum);
    context.write(key, result);
  }
}
```

Пояснения:

- Вход Reducer — (Text, IntWritable): слово и список единиц (типы совпадают с выходом Mapper).
- Выход — (Text, IntWritable): слово и итоговая сумма.
- Обход **values** и накопление **sum**; один вызов **context.write(key, result)** на слово.

---

## 3.4.4. Драйвер (main) и конфигурация Job

Драйвер создаёт Job, задаёт классы Mapper и Reducer, типы и пути входа/выхода. См. [Глоссарий](glossary.md).

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);  // опционально: предварительная сумма на узле Map
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

Пояснения:

- **setCombinerClass(IntSumReducer.class)** — опционально. Combiner выполняет ту же логику, что и Reducer, но на выводе одной Map-задачи локально; уменьшает объём данных, передаваемых в shuffle. Для WordCount Reducer и Combiner совпадают (сумма по ключу).
- Вход и выход задаются аргументами **args[0]** и **args[1]**.

Типы вывода Map (Text, IntWritable) совпадают с типами входа Reduce и с setOutputKeyClass/setOutputValueClass, поэтому **setMapOutputKeyClass** и **setMapOutputValueClass** в данном случае можно не вызывать (они выводятся из типов вывода Reducer). Явная установка не помешает.

---

## 3.4.5. Подготовка входа и запуск

Перед запуском job нужны входные данные в HDFS и собранный JAR. См. [Глоссарий](glossary.md).

**Загрузка тестовых данных в HDFS:**

```bash
hdfs dfs -mkdir -p /user/hadoop/input
hdfs dfs -put localfile.txt /user/hadoop/input/
# или несколько файлов:
hdfs dfs -put file1.txt file2.txt /user/hadoop/input/
```

**Сборка проекта** (пример для Maven; в проекте должна быть зависимость hadoop-mapreduce-client-core и при необходимости hadoop-common):

Сборка JAR (например, `wordcount.jar`) и упаковка классов — на усмотрение сборки (Maven, Gradle). Главный класс — WordCount.

**Запуск job:**

```bash
hadoop jar wordcount.jar WordCount /user/hadoop/input /user/hadoop/output
```

Выходной каталог **/user/hadoop/output** не должен существовать. Job создаст его и запишет файлы вида `part-r-00000`, `part-r-00001` (по одному на Reduce-задачу; по умолчанию одна Reduce-задача).

**Просмотр результата:**

```bash
hdfs dfs -cat /user/hadoop/output/part-r-00000
```

Пример строк вывода (слово и число через табуляцию):

```
hello   3
world   2
```

---

## 3.4.6. Соответствие кода этапам выполнения

| Этап из §3.2 | В WordCount |
|--------------|-------------|
| Сплиты | Текстовые файлы разбиваются на сплиты по блокам (TextInputFormat); каждая строка — одна запись (ключ — смещение, значение — строка). |
| Map | TokenizerMapper: для каждой строки разбиение на слова, для каждого слова — (слово, 1). |
| Shuffle/Sort | Пары группируются по слову (ключу); на узел Reduce приходят все единицы для каждого слова. |
| Reduce | IntSumReducer: для каждого слова суммирует единицы, пишет (слово, сумма). |
| Выход | Файлы part-r-* в выходном каталоге HDFS. |

Combiner (если включён) выполняется после Map на том же узле и уменьшает число пар, уходящих в shuffle: локально суммирует единицы по словам в выводе одной Map-задачи.

---

## 3.4.7. Варианты и расширения

- **Регистр и знаки препинания:** для сопоставления «Hello» и «hello» в Map приводят слово к нижнему регистру (**word.set(itr.nextToken().toLowerCase())**); знаки препинания можно отрезать или отфильтровывать по регулярному выражению.
- **Стоп-слова:** в Map не вызывать context.write для служебных слов (the, a, is, …) — отфильтровать по множеству стоп-слов.
- **Число Reduce-задач:** при большом словаре можно задать **job.setNumReduceTasks(N)** для параллельного подсчёта; ключи распределятся по задачам по хешу.
- **Вход из каталога:** FileInputFormat.addInputPath(job, new Path("/user/hadoop/input")) обрабатывает все файлы в каталоге (рекурсия при необходимости задаётся отдельно).

Эти доработки не меняют общую схему Map → (combiner) → shuffle → Reduce.

---

## 3.4.8. Типичные ошибки в WordCount

- **Не учитывать переиспользование объекта value в Iterable:** в reduce при обходе values объект IntWritable может переиспользоваться; если нужно сохранять значения в список, копируйте (new IntWritable(val.get())). В простом суммировании достаточно читать val.get().
- **Выходной каталог уже существует:** удалите его (**hdfs dfs -rm -r /user/hadoop/output**) или используйте API перезаписи вывода (если поддерживается версией).
- **Путь к JAR или классу неверен:** убедитесь, что hadoop jar находит главный класс (указан в команде или в манифесте JAR) и что зависимости Hadoop доступны на узлах кластера (обычно они уже установлены в кластере).

---

## Ключевое

- **WordCount:** Map для каждого слова выдаёт (слово, 1); Reduce суммирует единицы по слову и пишет (слово, сумма).
- **TokenizerMapper:** вход (LongWritable, Text) — строка; разбиение на слова, context.write(слово, 1).
- **IntSumReducer:** вход (Text, Iterable<IntWritable>); сумма values; context.write(слово, сумма).
- **Драйвер:** Job с setMapperClass(TokenizerMapper), setReducerClass(IntSumReducer); опционально setCombinerClass(IntSumReducer); пути из args; hadoop jar wordcount.jar WordCount вход выход.
- **Запуск:** данные в HDFS → hadoop jar … → результат в part-r-* в выходном каталоге; просмотр — hdfs dfs -cat.

В [§3.5](chapter-03-05.md) мы разберём ограничения модели MapReduce: итеративные алгоритмы, интерактивные запросы, латентность и накладные расходы и связь с появлением Spark.
