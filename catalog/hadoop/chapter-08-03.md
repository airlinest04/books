# §8.3 Coordinator в Oozie: расписание и триггер по данным

В [§8.2](chapter-08-02.md) мы разобрали структуру workflow и типы действий. **Coordinator** задаёт **когда** запускать workflow и **с какими параметрами**. В этом разделе — описание coordinator в XML: **запуск по расписанию** (частота или cron), **запуск по появлению данных** в HDFS (data trigger) и **параметризация по дате** для ETL «за вчера». Запуск и мониторинг — в [§8.4](chapter-08-04.md). См. [Глоссарий](glossary.md).

---

## 8.3.1. Структура coordinator

Coordinator описывается XML-файлом с корневым элементом **coordinator-app**. Основные секции: **controls** (временной интервал работы coordinator), **frequency** или **cron** (расписание), при необходимости **datasets** и **input-events** (триггер по данным), **action** (какой workflow запускать и какие параметры передавать). Файл размещают в HDFS; при запуске coordinator job Oozie читает определение и по расписанию или по данным создаёт **запуски** (run) workflow. См. [Глоссарий](glossary.md).

Упрощённый пример (только по расписанию):

```xml
<coordinator-app name="daily_etl" frequency="${coord:days(1)}" start="2024-01-01T00:00Z" end="2025-12-31T00:00Z" timezone="UTC" xmlns="uri:oozie:coordinator:1.0">
  <action>
    <workflow>
      <app-path>${wf_application_path}</app-path>
      <configuration>
        <property><name>date</name><value>${coord:formatTime(coord:dateOffset(coord:nominalTime(), -1, 'DAY'), 'yyyy-MM-dd')}</value></property>
      </configuration>
    </workflow>
  </action>
</coordinator-app>
```

- **frequency="${coord:days(1)}"** — запуск раз в день; **start** и **end** — границы периода, в течение которого coordinator активен.
- **action/workflow** — путь к workflow в HDFS (app-path) и **configuration**: свойства, передаваемые в workflow; здесь **date** — дата в формате yyyy-MM-dd, сдвинутая на -1 день от nominal time (т.е. «вчера»).

---

## 8.3.2. Запуск по расписанию (time-based)

**Time-based** coordinator срабатывает по **времени**. Расписание задаётся одним из способов: См. [Глоссарий](glossary.md).

- **frequency** — интервал в минутах: `frequency="60"` (каждый час), или через функции: `${coord:minutes(30)}`, `${coord:hours(1)}`, `${coord:days(1)}`. Oozie вычисляет **nominal time** для каждого запуска (теоретическое время срабатывания) и передаёт его в workflow через встроенные переменные.
- **cron** — cron-подобное выражение (в зависимости от версии Oozie): например, «каждый день в 02:00». Элемент `<control><timeout>...</timeout></control>` задаёт максимальное время ожидания действия; при **concurrency** можно ограничить число одновременно выполняющихся запусков одного workflow.

Для каждого срабатывания coordinator создаёт один **workflow job** и передаёт ему параметры (в том числе вычисленную дату). Workflow использует эти параметры в путях и в конфигурации действий (см. [§8.2](chapter-08-02.md)).

---

## 8.3.3. Параметризация по дате

В ETL-пайплайнах типична задача «обработать данные за вчера»: пути в HDFS содержат дату (`/data/events/2024-01-15/`), запросы Hive фильтруют по дате. Coordinator передаёт в workflow **параметры**, вычисленные от **nominal time** (время текущего срабатывания). См. [Глоссарий](glossary.md).

Основные функции Oozie EL (expression language) для дат:

- **coord:nominalTime()** — nominal time текущего запуска (ISO-формат).
- **coord:dateOffset(coord:nominalTime(), -1, 'DAY')** — сдвиг на −1 день.
- **coord:formatTime(..., 'yyyy-MM-dd')** — форматирование даты в строку.

В конфигурации action coordinator задаётся, например:

```xml
<property><name>date</name><value>${coord:formatTime(coord:dateOffset(coord:nominalTime(), -1, 'DAY'), 'yyyy-MM-dd')}</value></property>
```

В job.properties workflow или в самом workflow используется `${date}`; при запуске из coordinator подставится значение «вчера». Аналогично можно задать **startDate**, **endDate** для диапазона или несколько параметров (год, месяц, день).

---

## 8.3.4. Data trigger: запуск по появлению данных

**Data trigger** — запуск workflow не по времени, а когда в заданном каталоге HDFS **появляются данные** (файлы или каталоги), удовлетворяющие условию. Это полезно, когда момент готовности данных неизвестен заранее (например, выгрузка из внешней системы). См. [Глоссарий](glossary.md).

В coordinator задаются:

- **dataset** — описание источника данных: URI (каталог в HDFS), шаблон имени файла/каталога, частота появления (frequency), начальная инстанция и т.д.
- **input-events** — привязка action к dataset: workflow запускается, когда для текущего nominal time **данные в dataset доступны** (файлы появились). Можно задать **timeout** и **concurrency**.

Oozie периодически проверяет наличие данных (по HDFS); при выполнении условий создаётся run workflow с параметрами (в том числе путями к появившимся данным, если они заданы в dataset). Детали синтаксиса dataset и input-events зависят от версии Oozie; в ряде конфигураций data trigger требует настройки Oozie server для опроса HDFS.

---

## 8.3.5. Связь coordinator и workflow

Coordinator не подменяет workflow — он **запускает** его. Один coordinator может порождать много запусков (по одному на каждое срабатывание расписания или каждое появление данных). Каждый запуск — отдельный workflow job со своим идентификатором и своими параметрами (дата, пути). В веб-интерфейсе Oozie видны и список coordinator job, и список запущенных ими workflow job; по ним можно смотреть логи и статус. При приостановке или остановке coordinator новые запуски не создаются; уже запущенные workflow продолжают выполняться до завершения (в зависимости от настроек).

---

## Ключевое

- **Coordinator** задаётся в XML (coordinator-app): расписание (frequency или cron), при необходимости data trigger (dataset, input-events), action с путём к workflow и конфигурацией.
- **Time-based:** frequency (интервал) или cron; start/end ограничивают период работы coordinator; для каждого срабатывания — nominal time и один run workflow.
- **Параметризация по дате:** coord:nominalTime(), coord:dateOffset(), coord:formatTime(); параметры передаются в workflow (например, date=вчера) для путей и запросов.
- **Data trigger:** запуск при появлении данных в HDFS (dataset + input-events); удобно при зависимости от внешних выгрузок.

В [§8.4](chapter-08-04.md) разберём запуск workflow и coordinator, веб-интерфейс Oozie и краткое сравнение с Airflow.
