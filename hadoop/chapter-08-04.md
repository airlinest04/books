# §8.4 Запуск и мониторинг Oozie

В [§8.3](chapter-08-03.md) мы разобрали coordinator: расписание, data trigger и параметризацию по дате. Чтобы workflow и coordinator реально выполнялись, их нужно **отправить** в Oozie и при необходимости **отслеживать** статус и логи. В этом разделе — **отправка** workflow и coordinator через CLI, **веб-интерфейс** Oozie для мониторинга и управление job (приостановка, остановка), а также **краткое сравнение** с Apache Airflow. Следующая глава — [Sqoop](chapter-09-01.md): перенос данных между БД и HDFS. См. [Глоссарий](glossary.md).

---

## 8.4.1. Подготовка и размещение в HDFS

Перед запуском нужно разместить в **HDFS** определения workflow и coordinator и все зависимые файлы (скрипты Hive, Pig, jar, конфиги). Типичная структура: один каталог на workflow (например, `/user/oozie/workflows/etl_daily/`), в нём файл `workflow.xml`, подкаталоги `scripts/` (hive, pig) и при необходимости `lib/` (jar). В job.properties указывается путь к этому каталогу; Oozie читает workflow.xml из него. Для coordinator аналогично: coordinator.xml в своём каталоге или в том же, в зависимости от принятой схемы. См. [Глоссарий](glossary.md).

Пример содержимого job.properties для workflow:

```
nameNode=hdfs://namenode:8020
jobTracker=resourcemanager:8032
oozie.wf.application.path=/user/oozie/workflows/etl_daily
date=2024-01-15
```

Свойство **oozie.wf.application.path** — путь к каталогу в HDFS, где лежит workflow.xml. Остальные свойства подставляются в workflow как ${nameNode}, ${date} и т.д.

---

## 8.4.2. Отправка workflow

Запуск **workflow** выполняется командой **oozie job** с опцией **-run** (или **-submit** для только постановки в очередь без немедленного старта). Конфигурация передаётся через файл **-config**. См. [Глоссарий](glossary.md).

Пример:

```bash
oozie job -config job.properties -run
```

Oozie прочитает job.properties, возьмёт путь из oozie.wf.application.path, загрузит workflow.xml из HDFS, подставит параметры и запустит выполнение. В ответ выводится **Job ID** (например, `0000001-240115-...`). По этому ID можно смотреть статус и логи.

Только поставить в очередь без запуска:

```bash
oozie job -config job.properties -submit
```

Запуск уже отправленного job:

```bash
oozie job -start <job_id>
```

---

## 8.4.3. Отправка coordinator

**Coordinator** запускается аналогично: в **job.properties** задаётся **oozie.coord.application.path** — путь к каталогу в HDFS с coordinator.xml (и при необходимости с workflow, на который ссылается coordinator). См. [Глоссарий](glossary.md).

Пример:

```
oozie.coord.application.path=/user/oozie/coordinators/daily_etl
nameNode=hdfs://namenode:8020
```

Запуск:

```bash
oozie job -config coord.properties -run -oozie http://oozie-server:11000/oozie
```

После запуска coordinator Oozie начинает по расписанию (или по data trigger) создавать run workflow; каждый run имеет свой ID. Через CLI можно приостановить coordinator (**-suspend**), возобновить (**-resume**), остановить (**-kill**).

---

## 8.4.4. Веб-интерфейс и мониторинг

Oozie предоставляет **веб-интерфейс** (обычно по адресу `http://oozie-server:11000/oozie`). В нём отображаются: См. [Глоссарий](glossary.md).

- **Список job** — workflow и coordinator; статус (RUNNING, SUCCEEDED, FAILED, KILLED, SUSPENDED и т.д.), время начала и окончания.
- **Детали job** — граф workflow с подсветкой текущего узла; для каждого действия — статус, ссылки на логи в YARN (для MapReduce, Hive, Pig) или вывод действия.
- **Coordinator** — список созданных run; для каждого run — статус и ссылка на соответствующий workflow job.

По ссылке на внешний ID (YARN application ID) можно перейти в веб-интерфейс ResourceManager и смотреть логи мапперов и редьюсеров. При ошибке удобно идти от failed action в Oozie к логам конкретного задания в YARN.

Управление из веб-интерфейса: кнопки Kill, Suspend, Resume для выбранного job.

---

## 8.4.5. Основные команды CLI

Краткая сводка команд **oozie job**:

| Команда | Назначение |
|---------|------------|
| **-run** | Запустить job (workflow или coordinator) по конфигу. |
| **-submit** | Отправить job без запуска; далее -start. |
| **-info &lt;id&gt;** | Показать статус и параметры job. |
| **-log &lt;id&gt;** | Вывести логи job. |
| **-kill &lt;id&gt;** | Остановить job. |
| **-suspend &lt;id&gt;** | Приостановить (coordinator перестанет создавать новые run). |
| **-resume &lt;id&gt;** | Возобновить приостановленный job. |

Адрес Oozie server задаётся опцией **-oozie** или переменной окружения **OOZIE_URL**.

---

## 8.4.6. Сравнение с Apache Airflow (кратко)

**Apache Airflow** — оркестратор задач, популярный в современных пайплайнах данных. Отличия от Oozie (кратко): См. [Глоссарий](glossary.md).

- **Описание пайплайна:** в Airflow пайплайны задаются **кодом на Python** (DAG — directed acyclic graph); в Oozie — **XML**. Для части пользователей Python удобнее для версионирования и сложной логики.
- **Интерфейс:** Airflow даёт веб-UI с графом задач, логами по каждой задаче, возможностью перезапуска с места сбоя; Oozie UI проще, но тоже показывает граф и ссылки на логи.
- **Интеграция:** Oozie изначально заточен под Hadoop (HDFS, YARN, Hive, Pig, Sqoop); действия и конфигурация тесно связаны с экосистемой. Airflow универсален: операторы для множества систем (в том числе Hive, Spark, облачные сервисы); для чисто Hadoop-кластера может потребоваться настройка окружения.
- **Расписание и параметризация:** оба поддерживают расписание и передачу параметров (дата и т.д.); в Airflow это делается через контекст DAG run и переменные.

Выбор между Oozie и Airflow зависит от стека (уже используемый Hadoop vs облако/мультиплатформа), предпочтений по формату описания (XML vs Python) и требований к UI и расширяемости. В классических Hadoop-кластерах Oozie остаётся стандартным решением.

---

## Ключевое

- **Подготовка:** workflow.xml, coordinator.xml и скрипты размещаются в HDFS; в job.properties задаётся oozie.wf.application.path (или oozie.coord.application.path) и параметры.
- **Запуск:** `oozie job -config job.properties -run`; для coordinator — путь к каталогу с coordinator.xml; возвращается Job ID.
- **Мониторинг:** веб-интерфейс Oozie — список job, статус, граф workflow, ссылки на логи в YARN; CLI: -info, -log, -kill, -suspend, -resume.
- **Airflow:** пайплайны на Python (DAG), универсальные операторы; Oozie — XML и глубокая интеграция с Hadoop.

В [гл. 9](chapter-09-01.md) переходим к **Sqoop**: импорт данных из реляционных СУБД в HDFS и экспорт из HDFS в БД.
