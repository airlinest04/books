# §8.2 Workflow в Oozie: структура и действия

В [§8.1](chapter-08-01.md) мы разобрали назначение Oozie и понятия workflow, coordinator и bundle. **Workflow** в Oozie задаётся в **XML**: узлы (start, end, action, decision) и переходы между ними. В этом разделе — структура XML-документа workflow, **типы действий** (MapReduce, Hive, Pig, Shell, Java, Sqoop и др.) и **переходы по условиям** (decision). Coordinator — в [§8.3](chapter-08-03.md); запуск и мониторинг — в [§8.4](chapter-08-04.md). См. [Глоссарий](glossary.md).

---

## 8.2.1. Структура XML workflow

Workflow Oozie описывается XML-файлом с корневым элементом **workflow-app**. Обязательные элементы: **start** (точка входа), **end** (успешное завершение), один или несколько **action** (действия) и **decision** (при необходимости). Каждый узел имеет **name**; переходы задаются через **to** — имя следующего узла. См. [Глоссарий](glossary.md).

Минимальная структура:

```xml
<workflow-app name="example" xmlns="uri:oozie:workflow:1.0">
  <start to="first_action"/>
  <action name="first_action">
    <!-- тип действия: map-reduce, hive, pig, shell и т.д. -->
    <ok to="end"/>
    <error to="fail_node"/>
  </action>
  <end name="end"/>
  <kill name="fail_node">
    <message>Action failed</message>
  </kill>
</workflow-app>
```

- **start to="..."** — с какого узла начать выполнение.
- **action** — узел действия; внутри — конфигурация конкретного типа (map-reduce, hive и т.д.); дочерние элементы **ok** и **error** задают переход при успехе и при ошибке.
- **end** — конечный узел при нормальном завершении.
- **kill** — узел при ошибке; можно указать сообщение; после kill workflow завершается с неуспешным статусом.

Файл workflow (и при необходимости связанные скрипты, конфиги) размещают в HDFS; при запуске job Oozie указывают путь к этому файлу.

---

## 8.2.2. Действие MapReduce

Действие **map-reduce** запускает MapReduce job на кластере (через YARN). Внутри action задаются пути к jar, класс main, аргументы, пути входа/выхода, свойства job (configurations). См. [Глоссарий](glossary.md).

Пример фрагмента:

```xml
<action name="mr_job">
  <map-reduce>
    <job-tracker>${jobTracker}</job-tracker>
    <name-node>${nameNode}</name-node>
    <configuration>
      <property><name>mapreduce.job.jar</name><value>/path/to/job.jar</value></property>
      <property><name>mapreduce.map.class</name><value>com.example.MyMapper</value></property>
      <property><name>mapreduce.reduce.class</name><value>com.example.MyReducer</value></property>
      <property><name>mapreduce.input.fileinputformat.inputdir</name><value>${inputDir}</value></property>
      <property><name>mapreduce.output.fileoutputformat.outputdir</name><value>${outputDir}</value></property>
    </configuration>
  </map-reduce>
  <ok to="next_action"/>
  <error to="fail"/>
</action>
```

Переменные `${jobTracker}`, `${nameNode}`, `${inputDir}`, `${outputDir}` задаются в файле **job.properties** при запуске или передаются из coordinator (например, дата для путей).

---

## 8.2.3. Действия Hive, Pig, Sqoop

**Hive action** — выполнение Hive-скрипта (файл с HiveQL) или одного запроса. Указываются путь к скрипту в HDFS, параметры (например, дата), при необходимости путь к конфигу и аргументы. Oozie запускает Hive-клиент с этим скриптом; выполнение идёт на кластере (MapReduce/Tez/Spark в зависимости от настроек Hive). См. [Глоссарий](glossary.md).

**Pig action** — выполнение скрипта Pig Latin. Задаётся путь к скрипту в HDFS и параметры. Oozie запускает Pig с указанным скриптом; Pig компилирует его в job и выполняет на кластере.

**Sqoop action** — запуск Sqoop (import или export). В конфигурации указываются команда (import/export), JDBC-строка, таблица, пути HDFS, при необходимости параметры инкрементального импорта. Oozie выполняет Sqoop как отдельный процесс; Sqoop сам обращается к БД и к HDFS.

Для всех действий структура переходов одинакова: **ok to="..."** и **error to="..."**.

---

## 8.2.4. Действия Shell и Java

**Shell action** — выполнение shell-скрипта на узле, где запущена задача Oozie. Задаётся путь к скрипту в HDFS (или inline), аргументы, переменные окружения; при необходимости указывается путь к исполняемому файлу (например, bash). Используется для вспомогательных операций: копирование, вызов внешних утилит, подготовка окружения. См. [Глоссарий](glossary.md).

**Java action** — запуск Java-класса (main). Указываются класс, аргументы, пути к jar в HDFS; JVM запускается в контейнере. Подходит для небольшой кастомной логики без полноценного MapReduce job.

---

## 8.2.5. Decision: переход по условию

**Decision** (узел **decision**) задаёт ветвление по значению переменной или выражения. Вместо однозначного «ok → следующий узел» выполнение переходит в один из нескольких узлов в зависимости от условия. См. [Глоссарий](glossary.md).

Синтаксис (упрощённо):

```xml
<decision name="check_result">
  <switch>
    <case to="path_a">${wf:actionData('action_name')['status'] eq 'OK'}</case>
    <case to="path_b">${someVar eq 'value'}</case>
    <default to="path_default"/>
  </switch>
</decision>
```

Каждый **case** — условие (expression language); при первом истинном условии переход в указанный узел **to**; если ни одно не подошло — **default**. В условиях можно использовать свойства workflow (например, выход предыдущего действия), параметры job и встроенные функции Oozie (wf:actionData, coord:date() в coordinator и т.д.). Так реализуют сценарии вида «если шаг вернул код X — уведомить, иначе продолжить» или выбор ветки по параметру (например, по типу данных).

---

## 8.2.6. Параметры и свойства job

Workflow часто параметризуется: пути к данным, дата, имена таблиц. Параметры задаются в **job.properties** (файл при запуске `oozie job -run`) или передаются из coordinator. В XML они подставляются в виде **${paramName}**. Пример job.properties:

```
nameNode=hdfs://namenode:8020
jobTracker=resourcemanager:8032
inputDir=/data/events/${date}
outputDir=/data/out/${date}
```

В workflow XML используются `${inputDir}`, `${outputDir}`; при запуске из coordinator переменная `${date}` может задаваться выражением coordinator (например, дата за вчера). Так один и тот же workflow обслуживает разные даты и окружения.

---

## Ключевое

- **Workflow** задаётся в XML: start, action, end, kill; у каждого action — ok и error переходы.
- **Типы действий:** map-reduce (MapReduce job), hive (Hive-скрипт), pig (Pig-скрипт), sqoop (import/export), shell, java.
- **Decision** — узел с switch/case по условию; выбор следующего узла по выражению; default при отсутствии совпадения.
- Параметры workflow задаются в job.properties или из coordinator; в XML подстановка через ${paramName}.

В [§8.3](chapter-08-03.md) разберём coordinator: расписание, data trigger и параметризацию по дате.
