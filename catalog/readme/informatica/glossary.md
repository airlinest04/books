# Глоссарий

Термины Informatica, используемые в книге.

---

## A

**Active transformation** (активная трансформация) — трансформация, меняющая количество строк, границы транзакций или тип строки; примеры: Filter, Aggregator, Router. См. [§5.2](chapter-05-02.md).

**Advanced Pushdown Optimization (APDO)** — расширенная pushdown-оптимизация Informatica в IICS для облачных DWH (Snowflake, Databricks, BigQuery, Redshift, Azure Synapse); SQL и нативные команды; снижение data egress и затрат. См. [§13.3](chapter-13-03.md).

**Aggregator** — активная трансформация; агрегация по группам (GROUP BY); функции SUM, AVG, COUNT, MIN, MAX, FIRST, LAST и др. См. [§8.1](chapter-08-01.md).

---

## B

**Bottleneck** (узкое место) — компонент (target, source, mapping, session, system), ограничивающий производительность сессии; выявляется по performance counters; устраняется по порядку. См. [§11.4](chapter-11-04.md).

**Buffer** (буфер) — блок памяти DTM для передачи данных между reader, transformation и writer threads; настраиваются DTM Buffer Size и Default Buffer Block Size. См. [§11.2](chapter-11-02.md).

---

## C

**CDC** (Change Data Capture) — выявление и получение только изменений в источнике (вставки, обновления, удаления) для инкрементальной загрузки или репликации; реализуется через лог транзакций, триггеры или сравнение снимков. См. [§1.1](chapter-01-01.md).

**CLAIRE** — AI-движок Informatica на базе метаданных; обеспечивает рекомендации по маппингам, интеллектуальную оптимизацию, обнаружение аномалий. См. [§1.1](chapter-01-01.md), [§13.2](chapter-13-02.md).

**Code page** (кодовая страница) — набор символов для интерпретации байтов в строках; настраивается для flat file и БД; влияет на размер String/Nstring и корректность кириллицы, Unicode. См. [§3.3](chapter-03-03.md).

**Connected transformation** (подключённая трансформация) — трансформация, получающая данные из потока и передающая в следующие объекты; вход и выход связаны с другими трансформациями или Source/Target. См. [§5.3](chapter-05-03.md).

**Connection** (подключение) — глобальный объект репозитория с параметрами доступа к БД, FTP, Loader, Queue или Application; создаётся в Workflow Manager, используется Integration Service при выполнении сессий. См. [§2.3](chapter-02-03.md).

**Connector** (коннектор) — механизм подключения PowerCenter к источнику или приёмнику; ODBC, нативные драйверы, FTP, Loader, Queue, Application. См. [§3.3](chapter-03-03.md).

---

## D

**Deployment group** (группа развёртывания) — группа объектов репозитория (Mappings, Workflows, Sessions) для развёртывания единым блоком между окружениями. См. [§2.4](chapter-02-04.md).

**Designer** — клиентское приложение PowerCenter для создания и редактирования маппингов (Sources, Targets, Mappings, трансформации). См. [§1.3](chapter-01-03.md), [§4.1](chapter-04-01.md).

**Domain** (домен) — базовая административная единица PowerCenter; объединяет узлы (Nodes), сервисы (Repository Service, Integration Service), Gateway Node и Service Manager. См. [§1.3](chapter-01-03.md).

**DTM** (Data Transformation Manager) — процесс Integration Service, выполняющий сессию; выделяет buffer memory, управляет reader/transformation/writer threads. См. [§11.2](chapter-11-02.md).

**DWH** (Data Warehouse, хранилище данных) — предметно-ориентированная, интегрированная и привязанная ко времени совокупность данных для аналитики, отчётности и принятия решений; наполняется данными из источников через ETL. См. [§1.1](chapter-01-01.md).

---

## E

**ELT** (Extract, Load, Transform) — альтернатива ETL: данные загружаются в целевое хранилище, затем преобразования выполняются в самом DWH или data lake; предпочтителен для облачных DWH с pushdown. См. [§13.3](chapter-13-03.md).

**ETL** (Extract, Transform, Load) — процесс интеграции данных: извлечение из источников, преобразование по правилам очистки и бизнес-логики, загрузка в целевое хранилище. Informatica реализует ETL через Sources, Mappings, Targets, Sessions и Workflows. См. [§1.1](chapter-01-01.md), [§1.4](chapter-01-04.md).

**Expression** — пассивная трансформация для вычислений на уровне строки; поддерживает Input, Output, Variable порты; Expression Editor с функциями Transformation Language. См. [§5.4](chapter-05-04.md), [§6.2](chapter-06-02.md).

**Expression Editor** — диалог в Designer для ввода выражений в портах трансформаций (Expression, Filter, Router и др.); поддерживает Transformation Language. См. [§5.4](chapter-05-04.md), [§6.2](chapter-06-02.md).

---

## F

**Filter** — активная трансформация; пропускает строки, для которых условие TRUE; отбрасывает FALSE. Одно условие, одна выходная ветка. См. [§6.3](chapter-06-03.md).

**Flat file** (плоский файл) — текстовый файл с данными в виде строк и колонок; форматы: delimited (CSV, TSV) и fixed-width. Импорт через Sources/Targets → Import from File. См. [§3.4](chapter-03-04.md).

**Folder** (папка) — контейнер для организации объектов в репозитории; все Sources, Targets, Mappings, Workflows хранятся в папках. Создаётся в Repository Manager. См. [§2.2](chapter-02-02.md).

---

## I

**IDMC** (Informatica Intelligent Data Management Cloud) — единая облачная платформа Informatica: интеграция данных, качество, governance, каталог, MDM. См. [§1.1](chapter-01-01.md), [§13.1](chapter-13-01.md).

**Idempotency** (идемпотентность) — свойство пайплайна: повторный запуск с теми же входными данными даёт тот же результат без дубликатов; важно при retry и recovery. См. [§12.1](chapter-12-01.md).

**IICS** (Informatica Intelligent Cloud Services) — облачная платформа интеграции Informatica; включает Cloud Data Integration (ETL/ELT). См. [§1.1](chapter-01-01.md), [§13.1](chapter-13-01.md).

**Informatica** — корпоративная платформа управления данными; центральное место занимают ETL и интеграция данных. Включает PowerCenter (on-premise), Informatica Cloud (IICS), IDMC и др. См. [§1.1](chapter-01-01.md).

**Integration Service** — исполняющий движок PowerCenter; выполняет workflow и сессии (извлечение, преобразование, загрузка); читает метаданные из репозитория, подключается к источникам и приёмникам. См. [§1.3](chapter-01-03.md), [§9.2](chapter-09-02.md).

**Интеграция данных** (Data Integration) — объединение данных из множества источников в единое целевое хранилище или приложение; охватывает ETL, репликацию, синхронизацию, виртуализацию. См. [§1.1](chapter-01-01.md).

---

## J

**Joiner** — активная трансформация; соединяет два источника по Join Condition; типы: Normal (INNER), Master Outer (RIGHT), Detail Outer (LEFT), Full Outer. См. [§7.3](chapter-07-03.md), [§7.4](chapter-07-04.md).

---

## L

**Link** (связь) — соединение между задачами в workflow или worklet; определяет порядок выполнения; может иметь условие (link condition); циклы запрещены. См. [§9.4](chapter-09-04.md).

**Link condition** (условие связи) — выражение на связи между задачами; при True следующая задача выполняется; при False — пропускается; используется для условного запуска. См. [§9.4](chapter-09-04.md), [§10.3](chapter-10-03.md).

**Lookup** — трансформация для поиска данных в справочнике (таблица, файл) по Lookup Condition; поддерживает Connected/Unconnected, Cached/Uncached. См. [§7.1](chapter-07-01.md), [§7.2](chapter-07-02.md).

---

## M

**Mapping** (маппинг) — объект в Designer, описывающий поток данных от источников через трансформации к приёмникам; визуальное представление ETL-логики. См. [§1.4](chapter-01-04.md), [§4.1](chapter-04-01.md).

**Mapping Designer** — панель Designer для создания и редактирования маппингов; добавление Source, Target, трансформаций; связывание портов. Режимы: Iconized, Normal, Edit. См. [§4.1](chapter-04-01.md), [§4.2](chapter-04-02.md).

**Mapping parameter** (параметр маппинга) — параметр, определённый в маппинге; значение задаётся при выполнении сессии; используется в выражениях трансформаций. См. [§2.3](chapter-02-03.md), [§5.4](chapter-05-04.md).

**Mapplet** — переиспользуемая группа трансформаций в Mapplet Designer; добавляется в маппинги как единый блок; Input и Output — вход и выход. См. [§2.2](chapter-02-02.md), [§8.4](chapter-08-04.md).

**Mapplet Designer** — панель Designer для создания и редактирования Mapplets; Input и Output трансформации доступны только здесь. См. [§5.1](chapter-05-01.md), [§8.4](chapter-08-04.md).

**MDM** (Master Data Management, управление мастер-данными) — практика обеспечения единого согласованного представления сущностей (клиенты, продукты, контракты) по организации; Informatica MDM — отдельный продукт в экосистеме. См. [§1.1](chapter-01-01.md).

---

## N

**Native datatypes** (нативные типы) — типы данных, специфичные для источника или приёмника (Oracle NUMBER, SQL Server int, flat file string); отображаются в Source Analyzer и Target Designer. См. [§3.3](chapter-03-03.md).

**Node** (узел) — логическое представление вычислительной платформы (сервера) в домене PowerCenter; на узле выполняются процессы Repository Service и Integration Service. См. [§1.3](chapter-01-03.md).

**Normalizer** — активная трансформация; разворачивает multiple-occurring колонки в строки (unpivot); VSAM (COBOL) и Pipeline (relational/flat file). См. [§8.3](chapter-08-03.md).

---

## O

**On-premise** (в собственных ЦОД) — развёртывание программного обеспечения на серверах организации (в собственном ЦОД); под полным контролем заказчика; данные не покидают периметр. См. [§1.2](chapter-01-02.md).

---

## P

**Parameter file** — текстовый файл со списком параметров и переменных (name=value) для workflow и session; позволяет менять connections, пути, даты между запусками и окружениями. См. [§2.3](chapter-02-03.md), [§10.3](chapter-10-03.md).

**Partitioning** (партиционирование) — механизм PowerCenter для параллельной обработки данных; разделение потока на партиции; типы: Pass-through, Round-robin, Hash, Key Range, Database; требует Partitioning option. См. [§11.1](chapter-11-01.md).

**Passive transformation** (пассивная трансформация) — трансформация, не меняющая количество строк, границы транзакций и тип строки; примеры: Expression, Sequence Generator. См. [§5.2](chapter-05-02.md).

**Persistent cache** (постоянный кеш) — кеш Lookup, сохраняемый между сессиями; переиспользуется при следующих запусках без чтения источника. См. [§7.2](chapter-07-02.md).

**pmrep** — командная утилита PowerCenter для операций с метаданными: экспорт, импорт, сравнение объектов; автоматизация развёртывания и CI/CD. См. [§2.4](chapter-02-04.md).

**Port** (порт) — вход или выход трансформации; типы: Input (вход из потока), Output (выход, может содержать выражение), Variable (промежуточный расчёт). См. [§5.4](chapter-05-04.md).

**PowerCenter** — корпоративная ETL-платформа Informatica для on-premise; включает Designer, Repository, Integration Service, Workflow Manager, Workflow Monitor. См. [§1.1](chapter-01-01.md), [§1.2](chapter-01-02.md), [§1.3](chapter-01-03.md).

**Pushdown** (pushdown optimization) — перенос логики трансформаций в source или target БД; Integration Service генерирует SQL; БД выполняет обработку; типы: source-side, target-side, full. См. [§11.3](chapter-11-03.md).

---

## R

**Rank** — активная трансформация; отбирает топ-N или bottom-N строк по Rank port; поддерживает Group By и Rank Index. См. [§8.2](chapter-08-02.md).

**Rank Index** — выходной порт Rank с номером ранга (1, 2, 3, …); при совпадении значений несколько строк получают один ранг. См. [§8.2](chapter-08-02.md).

**Rank port** — порт Rank, по которому выполняется ранжирование (мера для сравнения). См. [§8.2](chapter-08-02.md).

**Recovery** (восстановление) — возобновление workflow после Stop, Abort или Terminate; стратегии: Resume (с checkpoint), Restart (повторный запуск), Fail (продолжить без восстановления сессии). См. [§10.4](chapter-10-04.md).

**Repository** (репозиторий) — реляционная БД, хранящая метаданные PowerCenter (Sources, Targets, Mappings, Workflows, Connections); управляется Repository Service. См. [§1.3](chapter-01-03.md), [§2.1](chapter-02-01.md).

**Repository Domain** (домен репозиториев) — логическая группа репозиториев в PowerCenter Client для обмена метаданными через global repository; не путать с PowerCenter Domain. См. [§2.1](chapter-02-01.md).

**Repository Server** — совокупность Repository (БД метаданных) и Repository Service (процесс доступа); термин часто используют как синоним Repository Service. См. [§2.1](chapter-02-01.md).

**Repository Service** — серверный процесс PowerCenter; управляет доступом к репозиторию, блокировками объектов, версионированием; принимает запросы от клиентов и Integration Service. См. [§1.3](chapter-01-03.md), [§2.1](chapter-02-01.md).

**Router** — активная трансформация; несколько групп с условиями; направляет строки в разные выходные ветки; Default group — строки, не попавшие в user-defined группы. См. [§6.3](chapter-06-03.md).

---

## S

**Schedule** (расписание) — настройки Workflow Scheduler: когда Integration Service запускает workflow; Run On Demand, Run Every, Customized Repeat; Start/End Options. См. [§10.1](chapter-10-01.md).

**SCD** (Slowly Changing Dimension, медленно меняющееся измерение) — измерение DWH, атрибуты которого меняются со временем; Type 1 (перезапись), Type 2 (версионность), Type 3 (предыдущее значение); реализуется через Slowly Changing Dimensions Wizard. См. [§4.4](chapter-04-04.md), [§12.1](chapter-12-01.md).

**Secure Agent** — агент Informatica Cloud, устанавливаемый в периметре заказчика; обеспечивает доступ IICS к on-premise источникам (БД, файлы) без выноса данных в облако Informatica. См. [§1.2](chapter-01-02.md).

**Select Distinct** — опция Source Qualifier; добавляет SELECT DISTINCT в default query; дедупликация на стороне источника; при SQL-override не применяется. См. [§6.1](chapter-06-01.md), [§8.2](chapter-08-02.md).

**Sequence Generator** — пассивная трансформация; генерирует числовые значения (NEXTVAL, CURRVAL) для первичных ключей и последовательностей. См. [§8.3](chapter-08-03.md).

**Session** (сессия) — задача в Workflow, содержащая инструкции для Integration Service по выполнению маппинга; ссылается на Mapping; настраивает Connections, пути, параметры. См. [§1.4](chapter-01-04.md), [§9.2](chapter-09-02.md).

**Session parameter** (параметр сессии) — значение, меняющееся между запусками сессии (connection, путь к файлу и т.д.); задаётся в parameter file или в свойствах Session. См. [§2.3](chapter-02-03.md), [§10.3](chapter-10-03.md).

**Shortcut** (ярлык) — ссылка на объект из shared folder; local shortcut — в том же репозитории, global shortcut — из global repository в Repository Domain. См. [§2.2](chapter-02-02.md).

**Source** (источник) — определение структуры данных-источника в Designer (таблица БД, файл); метаданные хранятся в репозитории; используется в маппинге. См. [§1.4](chapter-01-04.md), [§3.1](chapter-03-01.md).

**Source Analyzer** — панель Designer для создания и импорта source definitions; Sources → Import from Database, Import from File. См. [§3.1](chapter-03-01.md).

**Source Filter** (фильтр источника) — условие WHERE в Source Qualifier; добавляется в default query; уменьшает объём данных на этапе извлечения. См. [§6.1](chapter-06-01.md).

**Source Qualifier** — трансформация в маппинге, связывающая Source с потоком данных; по умолчанию генерирует SELECT; поддерживает SQL-override для фильтрации и join на стороне источника. См. [§1.4](chapter-01-04.md), [§6.1](chapter-06-01.md).

**Sorter** — активная трансформация; сортирует данные по Sort Key (один или несколько портов); Ascending/Descending; опция Distinct. См. [§6.4](chapter-06-04.md).

**SQL-override** (переопределение SQL) — замена default query Source Qualifier на произвольный SQL; позволяет фильтрацию, сортировку, join на стороне источника. См. [§6.1](chapter-06-01.md), [§11.3](chapter-11-03.md).

**Staging** (стейджинг) — промежуточный слой DWH для «сырых» данных с минимальной трансформацией; загрузка из источника в Staging — типовой первый шаг пайплайна. См. [§12.1](chapter-12-01.md).

**Stored Procedure** — пассивная трансформация; вызывает хранимую процедуру в БД; Connected или Unconnected. См. [§8.3](chapter-08-03.md).

---

## T

**Target** (приёмник) — определение структуры целевой таблицы или файла в Designer; метаданные хранятся в репозитории; используется в маппинге. См. [§1.4](chapter-01-04.md), [§3.2](chapter-03-02.md).

**Target definition** (определение приёмника) — объект репозитория, описывающий структуру целевой таблицы или файла (колонки, типы, ключи); используется Mapping Designer и Integration Service. См. [§3.2](chapter-03-02.md).

**Target Designer** — панель Designer для создания и редактирования target definitions; импорт из БД и файлов, Create from Source, Generate/Execute SQL. См. [§3.2](chapter-03-02.md).

**Transformation** (трансформация) — объект репозитория, генерирующий, изменяющий или передающий данные; реализует этап Transform в ETL. Классификация: Active/Passive, Connected/Unconnected, Native/Non-native. См. [§5.1](chapter-05-01.md).

**Transformation datatypes** (типы трансформаций) — универсальные типы PowerCenter (Integer, Decimal, String, Date/Time и др.), основанные на ANSI SQL-92; используются во всех трансформациях; позволяют переносить данные между разными платформами. См. [§3.3](chapter-03-03.md).

---

## U

**Unconnected transformation** (неподключённая трансформация) — трансформация, не связанная с потоком; вызывается из выражения (:LKP, :SP) и возвращает одно значение. Поддерживают Lookup, Stored Procedure, External Procedure. См. [§5.3](chapter-05-03.md).

**Union** — активная трансформация; несколько input groups, один output; объединяет потоки с совместимой структурой (аналог SQL UNION ALL); дубликаты не удаляет. См. [§6.4](chapter-06-04.md).

**Update Strategy** — активная трансформация; помечает строки для Insert (DD_INSERT), Update (DD_UPDATE), Delete (DD_DELETE) или Reject (DD_REJECT). См. [§8.3](chapter-08-03.md), [§12.3](chapter-12-03.md).

---

## V

**Version control** (версионирование) — при team-based development: хранение нескольких версий объектов, labels, сравнение и развёртывание выбранной версии. См. [§2.1](chapter-02-01.md), [§2.4](chapter-02-04.md).

---

## W

**Workflow** — набор инструкций (tasks), которые Integration Service выполняет при запуске; оркестрация Session, Command, Email, Decision и др. Создаётся в Workflow Manager. См. [§9.1](chapter-09-01.md).

**Workflow Manager** — клиентское приложение PowerCenter для создания workflow, задач (Session, Command, Email и др.), расписаний и соединений. См. [§1.3](chapter-01-03.md), [§9.1](chapter-09-01.md).

**Workflow Monitor** — клиентское приложение PowerCenter для мониторинга выполнения workflow; Gantt Chart и Task view; просмотр статуса, Session Log, Workflow Log; Stop и Abort. См. [§1.3](chapter-01-03.md), [§10.2](chapter-10-02.md).

**Worklet** — объект, представляющий набор задач workflow для переиспользования; создаётся в Worklet Designer; вкладывается в workflow и другие worklets; reusable или non-reusable. См. [§2.2](chapter-02-02.md), [§9.3](chapter-09-03.md).

**Worklet Designer** — панель Workflow Manager для создания и редактирования worklets; Worklet → Create. См. [§9.3](chapter-09-03.md).

---

## Г

**Гибридный сценарий** — одновременное использование PowerCenter (on-premise) и Informatica Cloud (IICS) в одной организации; сосуществование по слоям или доменам, поэтапная миграция. См. [§1.2](chapter-01-02.md).

---

## С

**Сверка данных** — проверка корректности загрузки: сравнение row count source и target, анализ Session Log (Source rows read, Target rows loaded, Rejected rows), выборочная проверка ключевых полей; обеспечивает контроль качества ETL. См. [§12.4](chapter-12-04.md).
