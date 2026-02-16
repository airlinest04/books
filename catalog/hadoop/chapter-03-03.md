# §3.3 Программа MapReduce

В [§3.2](chapter-03-02.md) мы разобрали этапы выполнения job: сплиты, Map, shuffle and sort, Reduce, вывод в HDFS. Чтобы запустить такой job, нужно написать **программу**: классы **Mapper** и **Reducer** с методами map и reduce и **конфигурацию Job** (вход, выход, типы, число Reduce-задач). В этом разделе рассматриваем структуру программы MapReduce в Hadoop: объявление Mapper и Reducer, типы входов и выходов (Writable), настройка Job и **запуск через `hadoop jar`**. Полный пример с подсчётом слов — в [§3.4](chapter-03-04.md); здесь — каркас и ключевые элементы. См. [Глоссарий](glossary.md).

---

## 3.3.1. Класс Mapper и метод map

**Mapper** — класс, реализующий логику этапа Map. В Hadoop (пакет `org.apache.hadoop.mapreduce`) вы наследуетесь от класса **`Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>`** и переопределяете метод **`map(KEYIN key, VALUEIN value, Context context)`**. См. [Глоссарий](glossary.md).

Параметры типов:

- **KEYIN, VALUEIN** — типы ключа и значения на входе Map; они задаются **InputFormat**. Для типичного текстового входа (TextInputFormat) KEYIN — смещение в байтах (LongWritable), VALUEIN — строка (Text).
- **KEYOUT, VALUEOUT** — типы ключа и значения на выходе Map; их вы выдаёте в метод **`context.write(key, value)`**. Они должны совпадать с типами входа Reducer (KEYIN, VALUEIN для Reducer).

Сигнатура метода map:

```java
@Override
public void map(KEYIN key, VALUEIN value, Context context)
    throws IOException, InterruptedException {
  // Обработка одной входной записи; вызов context.write(keyOut, valueOut) для каждой выходной пары
}
```

Одна запись (key, value) — одна единица данных из RecordReader (например, одна строка файла). Внутри map вы можете вызвать **`context.write(keyOut, valueOut)`** ноль, один или несколько раз; каждая выдача — пара (ключ, значение) для этапа Reduce. Типы keyOut и valueOut должны быть **Writable** (или WritableComparable для ключа), например Text, IntWritable, LongWritable из пакета `org.apache.hadoop.io`. См. [Глоссарий](glossary.md).

---

## 3.3.2. Класс Reducer и метод reduce

**Reducer** — класс, реализующий логику этапа Reduce. Вы наследуетесь от **`Reducer<KEYIN, VALUEIN, KEYOUT, VALUEOUT>`** и переопределяете метод **`reduce(KEYIN key, Iterable<VALUEIN> values, Context context)`**. См. [Глоссарий](glossary.md).

Параметры типов:

- **KEYIN, VALUEIN** — должны совпадать с KEYOUT, VALUEOUT Mapper’а: это пары, пришедшие после shuffle and sort.
- **KEYOUT, VALUEOUT** — типы результата, записываемого в HDFS через **`context.write(keyOut, valueOut)`**.

Сигнатура метода reduce:

```java
@Override
public void reduce(KEYIN key, Iterable<VALUEIN> values, Context context)
    throws IOException, InterruptedException {
  // Агрегация по ключу: обход values, расчёт результата, context.write(key, result)
}
```

Метод вызывается **один раз на каждый уникальный ключ**; `values` — итератор по всем значениям этого ключа (порядок может быть задан сортировкой). Обычно вы обходите values, накапливаете результат (сумма, список, максимум) и один раз вызываете `context.write(key, result)`. Повторное использование одного и того же объекта value в итераторе (object reuse) в части API требует копирования значения, если нужно хранить его после итерации; детали см. в документации Hadoop.

---

## 3.3.3. Конфигурация Job

**Job** — объект, описывающий один MapReduce job: какие классы использовать, откуда читать вход, куда писать выход, число Reduce-задач и прочие параметры. См. [Глоссарий](glossary.md).

Типичная настройка в коде (драйвер, метод main или отдельный метод):

```java
Job job = Job.getInstance(conf, "MyMapReduceJob");
job.setJarByClass(MyDriverClass.class);

job.setMapperClass(MyMapper.class);
job.setReducerClass(MyReducer.class);

job.setMapOutputKeyClass(Text.class);      // KEYOUT маппера = KEYIN редьюсера
job.setMapOutputValueClass(IntWritable.class); // VALUEOUT маппера = VALUEIN редьюсера

job.setOutputKeyClass(Text.class);         // KEYOUT редьюсера (и итоговый вывод)
job.setOutputValueClass(IntWritable.class);    // VALUEOUT редьюсера

FileInputFormat.addInputPath(job, new Path(args[0]));
FileOutputFormat.setOutputPath(job, new Path(args[1]));

job.setNumReduceTasks(1);  // при необходимости изменить число Reduce-задач

job.waitForCompletion(true);
```

Кратко по пунктам:

- **setJarByClass** — класс, по которому будет найден JAR с job (для отправки на кластер).
- **setMapperClass / setReducerClass** — классы Mapper и Reducer.
- **setMapOutputKeyClass / setMapOutputValueClass** — типы вывода Map (и входа Reduce).
- **setOutputKeyClass / setOutputValueClass** — типы вывода Reduce (записи в HDFS).
- **FileInputFormat.addInputPath** — путь к входу в HDFS (файл или каталог); можно вызывать addInputPath несколько раз.
- **FileOutputFormat.setOutputPath** — каталог вывода в HDFS; он не должен существовать перед запуском (job создаёт его сам; при существовании — ошибка, если не включено перезаписывание).
- **setNumReduceTasks** — число Reduce-задач; по умолчанию 1. Ноль — только Map, без Reduce.
- **waitForCompletion(true)** — запуск job и ожидание завершения; аргумент true означает вывод прогресса в консоль.

Конфигурация **Configuration** (conf) передаётся в Job.getInstance; в ней задаются параметры HDFS, YARN и MapReduce (адрес NameNode, размер сплита и т.д.). Часть параметров можно задать из командной строки или файлов конфигурации кластера.

---

## 3.3.4. Драйвер и точка входа

**Драйвер** — код (обычно метод **main**), который создаёт Configuration, строит Job, задаёт вход/выход из аргументов командной строки и вызывает **waitForCompletion**. См. [Глоссарий](glossary.md).

Минимальный каркас:

```java
public static void main(String[] args) throws Exception {
  Configuration conf = new Configuration();
  Job job = Job.getInstance(conf, "ExampleJob");
  job.setJarByClass(ExampleDriver.class);
  job.setMapperClass(ExampleMapper.class);
  job.setReducerClass(ExampleReducer.class);
  // ... установка типов и путей ...
  FileInputFormat.addInputPath(job, new Path(args[0]));
  FileOutputFormat.setOutputPath(job, new Path(args[1]));
  System.exit(job.waitForCompletion(true) ? 0 : 1);
}
```

Аргументы **args[0]** и **args[1]** при запуске через `hadoop jar` — это, как правило, путь к входу в HDFS и путь к выходному каталогу в HDFS. Код выхода 0 при успехе, 1 при неудаче — удобно для скриптов.

---

## 3.3.5. Типы Writable

Ключи и значения в MapReduce должны сериализоваться для передачи между узлами и записи в HDFS. В Hadoop для этого используются типы, реализующие **Writable** (значения) или **WritableComparable** (ключи: Writable + сравнение для сортировки). См. [Глоссарий](glossary.md).

Часто используемые типы из **`org.apache.hadoop.io`**:

| Тип | Описание |
|-----|----------|
| Text | Строка (UTF-8). |
| IntWritable | Целое 32 бита. |
| LongWritable | Целое 64 бита. |
| FloatWritable, DoubleWritable | Числа с плавающей точкой. |
| NullWritable | Пустое значение (ключ или значение «без данных»). |

Для составных ключей или значений можно реализовать собственный Writable (методы write и readFields) или использовать составные типы (например, в старом API — TupleWritable; в приложениях часто делают свой класс). Типы вывода Map и входа/выхода Reduce должны быть согласованы с setMapOutputKeyClass, setOutputKeyClass и т.д.

---

## 3.3.6. Сборка и запуск: hadoop jar

Программу собирают в **JAR-файл** (с зависимостями Hadoop в classpath или упакованными в JAR). Запуск на кластере — команда **`hadoop jar`**. См. [Глоссарий](glossary.md).

Синтаксис:

```bash
hadoop jar <путь_к_jar> <главный_класс> [аргументы...]
```

Если в JAR указан Main-Class в манифесте, главный класс можно не указывать. Аргументы передаются в args[] метода main; обычно первый — входной путь в HDFS, второй — выходной каталог в HDFS.

Пример (после сборки проекта):

```bash
hadoop jar myjob.jar com.example.WordCount /user/hadoop/input /user/hadoop/output
```

Команда отправляет job в кластер (YARN); вход читается из `/user/hadoop/input`, результат пишется в `/user/hadoop/output`. Прогресс и логи выводятся в консоль; детальный статус и логи задач можно смотреть в веб-интерфейсе ResourceManager и History Server. Среда: Hadoop установлен, HADOOP_CONF_DIR или конфигурация указывают на кластер; для локального теста часто используют локальный режим (без YARN) или псевдо-распределённый кластер.

---

## 3.3.7. Типичные ошибки при написании программы

- **Не согласовать типы Map и Reduce:** setMapOutputKeyClass/setMapOutputValueClass должны совпадать с типами, которые Reducer ожидает на входе (KEYIN, VALUEIN). Несовпадение ведёт к ошибкам сериализации или ClassCastException.
- **Не задать выходной путь или указать существующий каталог:** выходной каталог не должен существовать; job создаёт его сам. Иначе job падает с ошибкой (если не включено перезаписывание).
- **Забыть setJarByClass:** без указания JAR классы могут не найтись на узлах при выполнении; задайте класс из вашего JAR (часто драйвер).
- **Использовать стандартные Java-типы (String, Integer) вместо Writable:** в контексте Hadoop ключи и значения должны быть Writable (Text, IntWritable и т.д.); иначе контекст не сможет их сериализовать.

---

## Ключевое

- **Mapper:** наследование от `Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>`, переопределение **map(key, value, context)**; вывод через **context.write(keyOut, valueOut)**; типы — Writable.
- **Reducer:** наследование от `Reducer<KEYIN, VALUEIN, KEYOUT, VALUEOUT>`, переопределение **reduce(key, values, context)**; один вызов на ключ; вывод через **context.write(keyOut, valueOut)**.
- **Job:** setMapperClass, setReducerClass; setMapOutputKeyClass/setMapOutputValueClass (выход Map); setOutputKeyClass/setOutputValueClass (выход Reduce); FileInputFormat.addInputPath, FileOutputFormat.setOutputPath; setNumReduceTasks; waitForCompletion.
- **Запуск:** сборка в JAR, команда **hadoop jar myjob.jar MainClass вход_путь выход_каталог**; аргументы передаются в main.
- Типы ключей и значений — **Writable** (Text, IntWritable, LongWritable и др.) из org.apache.hadoop.io.

В [§3.4](chapter-03-04.md) мы разберём полный пример: подсчёт слов (WordCount) — код Mapper и Reducer, конфигурация и запуск.
