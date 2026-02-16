# §1.3 Архитектура и компоненты

В [§1.2](chapter-01-02.md) мы сравнили PowerCenter и Informatica Cloud. Теперь переходим к архитектуре **PowerCenter**: какие компоненты входят в платформу, как они связаны и как данные и метаданные перемещаются между ними. Вы узнаете назначение Repository Server, Integration Service, Designer, Workflow Manager и Workflow Monitor, а также роль домена (Domain), узлов (Nodes) и Service Manager. Понимание архитектуры необходимо для настройки окружения, отладки и планирования производительности. Подробнее репозиторий и метаданные — в [главе 2](chapter-02-01.md). См. [Глоссарий](glossary.md).

---

## 1.3.1. Обзор архитектуры PowerCenter

Архитектура PowerCenter построена по принципу **Service Oriented Architecture (SOA)**: платформа состоит из набора сервисов и клиентских приложений, взаимодействующих через репозиторий метаданных. Источник: [Informatica PowerCenter Architecture Tutorial](https://guru99.com/informatica-architecture-tutorial.html).

| Уровень | Компоненты |
|---------|------------|
| **Клиентские приложения** | Designer, Workflow Manager, Workflow Monitor, Repository Manager, Administrator |
| **Серверные сервисы** | Repository Service, Integration Service, Reporting Service |
| **Инфраструктура** | Domain, Nodes, Gateway Node, Service Manager |
| **Хранилище метаданных** | Repository (реляционная БД: Oracle, SQL Server, Sybase и др.) |

Клиентские приложения устанавливаются на рабочие станции разработчиков и администраторов; серверные сервисы — на узлах (Nodes) в домене. Репозиторий хранится в отдельной реляционной СУБД; Repository Service обеспечивает доступ к метаданным. Integration Service выполняет ETL-сессии: читает данные из источников, применяет трансформации согласно маппингу и записывает в приёмники. См. [Repository Architecture](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/repository-architecture.html).

---

## 1.3.2. Domain и инфраструктура

**Domain** (домен) — базовая административная единица PowerCenter. Домен объединяет узлы (Nodes) и сервисы; в нём задаются настройки аутентификации, авторизации, логирования и управления пользователями. Управление доменом выполняется через веб-консоль **Administrator** (Informatica Admin Console). См. [Глоссарий](glossary.md).

**Node** (узел) — логическое представление вычислительной платформы (сервера) в домене. На узле выполняются процессы сервисов (Repository Service, Integration Service). В домене может быть несколько узлов; один из них — **Gateway Node** — принимает запросы от клиентских приложений и маршрутизирует их к нужным узлам и сервисам.

**Service Manager** — сервис, управляющий доменом: аутентификация, авторизация, логирование, запуск и остановка прикладных сервисов, управление пользователями и группами. Service Manager работает на каждом узле.

Ключевые свойства домена (настраиваются в Admin Console):

| Свойство | Описание |
|----------|----------|
| **Resilience timeout** | Время (секунды), в течение которого прикладной сервис пытается переподключиться при сбое. |
| **Restart Period** | Максимальное время перезапуска сервиса. |
| **Dispatch Mode** | Политика балансировки нагрузки при распределении задач по узлам. |
| **Database type/host/port** | Тип и параметры БД, в которой хранятся метаданные домена. |

Домен — точка входа для всех клиентов: Designer, Workflow Manager, Workflow Monitor подключаются к домену (через Gateway Node), а не напрямую к репозиторию или Integration Service.

---

## 1.3.3. Repository и Repository Service

**Repository** (репозиторий) — реляционная база данных (Oracle, SQL Server, Sybase, DB2 и др.), в которой хранятся **метаданные** PowerCenter: определения источников (Sources), приёмников (Targets), маппингов (Mappings), workflow, соединений (Connections), переменных и параметров. Репозиторий не хранит сами данные ETL — только инструкции по их извлечению, преобразованию и загрузке. См. [Глоссарий](glossary.md).

**Repository Service** — серверный процесс, управляющий доступом к репозиторию. Один экземпляр Repository Service обслуживает **один** репозиторий; при необходимости может выполняться на нескольких узлах для повышения производительности. Repository Service:

- принимает запросы на чтение и запись метаданных от клиентов (Designer, Workflow Manager, Workflow Monitor, Integration Service);
- обеспечивает **блокировки объектов** (object locking), чтобы несколько пользователей не изменяли один объект одновременно;
- поддерживает **версионирование** объектов (при включённой функции version control).

Клиентские приложения и Integration Service обращаются к репозиторию **только через** Repository Service; прямого доступа к таблицам репозитория у них нет. Источник: [Repository Architecture](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/repository-architecture.html).

Состояния объектов в репозитории:

| Состояние | Описание |
|-----------|----------|
| **Valid** | Объект корректен по синтаксису и правилам; может использоваться в выполнении workflow. |
| **Invalid** | Объект не соответствует стандартам или правилам; не может быть выполнен до исправления. |
| **Impacted** | Объект сам корректен, но зависит от дочернего объекта, который Invalid; косвенно затронут ошибкой. |

---

## 1.3.4. Integration Service

**Integration Service** — исполняющий движок PowerCenter. Он выполняет workflow и сессии (Sessions): читает метаданные маппинга и workflow из репозитория, подключается к источникам и приёмникам, извлекает данные, применяет трансформации и загружает результат в целевые таблицы или файлы. См. [Глоссарий](glossary.md).

Последовательность работы Integration Service при запуске workflow:

1. Пользователь запускает workflow (вручную или по расписанию).
2. Integration Service получает уведомление о необходимости выполнить workflow.
3. Integration Service подключается к репозиторию и читает метаданные workflow: какие задачи (Tasks) входят в workflow, в каком порядке, какие маппинги выполняются.
4. Для каждой задачи типа Session Integration Service читает метаданные маппинга: источники, трансформации, приёмники, соединения.
5. Integration Service подключается к источникам (БД, файлы), извлекает данные, выполняет трансформации в памяти (или с pushdown в СУБД) и записывает в приёмники.
6. По завершении Integration Service записывает в репозиторий статус выполнения (успех, сбой, прервано), генерирует Session Log и Workflow Log.

Integration Service может объединять данные из разных источников (например, таблица Oracle и плоский файл), выполнять сложные трансформации и загружать в несколько приёмников. Масштабирование: при использовании Grid несколько узлов Integration Service могут выполнять сессии параллельно. Источник: [Guru99 Informatica Architecture](https://guru99.com/informatica-architecture-tutorial.html).

---

## 1.3.5. Designer

**Designer** — клиентское приложение для создания и редактирования **маппингов** (Mappings). В Designer разработчик:

- импортирует или создаёт определения источников (Sources) и приёмников (Targets);
- создаёт маппинги: добавляет источники, трансформации (Expression, Filter, Lookup, Joiner, Aggregator и др.) и приёмники;
- связывает порты (поля) между объектами, задаёт выражения и условия;
- сохраняет объекты в репозиторий.

Designer состоит из нескольких панелей (по версиям): Source Analyzer (импорт и просмотр источников), Target Designer (приёмники), Transformation Developer (создание переиспользуемых трансформаций), Mapplet Designer (Mapplets), Mapping Designer (создание маппингов). Все объекты сохраняются в репозиторий через Repository Service. Designer не выполняет маппинги — он только создаёт метаданные; выполнение выполняет Integration Service. См. [§4.1](chapter-04-01.md), [§4.2](chapter-04-02.md).

---

## 1.3.6. Workflow Manager

**Workflow Manager** — клиентское приложение для создания и управления **workflow** (рабочими процессами). В Workflow Manager разработчик:

- создаёт workflow, добавляет задачи (Tasks): Session (выполнение маппинга), Command (выполнение команды ОС), Email (отправка письма), Decision (условная ветвление) и др.;
- задаёт связи между задачами (последовательность, параллельные ветви, условия);
- настраивает расписание (Schedule) для автоматического запуска;
- указывает соединения (Connections) для источников и приёмников в сессиях;
- сохраняет workflow в репозиторий.

Workflow Manager также управляет объектами соединений (Connection objects) — определениями подключений к БД и файлам, которые используются в сессиях. При запуске workflow пользователь выбирает Integration Service, который будет выполнять задачи; Workflow Manager передаёт управление Integration Service. См. [§9.1](chapter-09-01.md), [§9.2](chapter-09-02.md).

---

## 1.3.7. Workflow Monitor

**Workflow Monitor** — клиентское приложение для **мониторинга** выполнения workflow. В Workflow Monitor пользователь:

- просматривает статус запущенных и завершённых workflow (Running, Succeeded, Failed, Aborted, Stopped);
- видит историю запусков, время выполнения, количество обработанных строк;
- открывает Session Log и Workflow Log для анализа ошибок и производительности;
- может останавливать (Stop) или прерывать (Abort) выполняющийся workflow.

Workflow Monitor читает информацию из репозитория: Integration Service записывает туда статус и логи по мере выполнения. Workflow Monitor не управляет выполнением напрямую — он только отображает данные, записанные Integration Service. См. [§10.2](chapter-10-02.md).

---

## 1.3.8. Repository Manager и Administrator

**Repository Manager** — клиентское приложение для организации и администрирования объектов репозитория. С его помощью создают папки (Folders), перемещают объекты между папками, назначают права доступа, выполняют экспорт и импорт (деплой между окружениями). Repository Manager полезен при настройке структуры проекта и миграции между dev/test/prod.

**Administrator** (Informatica Admin Console) — веб-консоль для управления доменом: узлы, сервисы, пользователи, настройки. Через Administrator запускают и останавливают Repository Service и Integration Service, просматривают логи домена, настраивают resilience и dispatch mode. Administrator — инструмент администратора инфраструктуры, а не разработчика маппингов.

---

## 1.3.9. Потоки данных и метаданных

Понимание потоков помогает при отладке и планировании архитектуры.

**Поток метаданных (разработка):**

```
Designer / Workflow Manager
         |
         v
   Repository Service
         |
         v
   Repository (БД)
```

Разработчик создаёт Sources, Targets, Mappings, Workflows в Designer и Workflow Manager; объекты сохраняются в репозиторий через Repository Service. Метаданные хранятся в таблицах репозитория.

**Поток при выполнении workflow:**

```
Пользователь / Schedule
         |
         v
   Workflow Manager -> Integration Service
         |
         v
   Integration Service читает метаданные из Repository (через Repository Service)
         |
         v
   Integration Service подключается к Source (БД, файл)
         |
         v
   Извлечение -> Трансформация (в памяти или pushdown)
         |
         v
   Integration Service подключается к Target (БД, файл)
         |
         v
   Загрузка; запись статуса и логов в Repository
```

**Поток при мониторинге:**

```
Integration Service пишет статус и логи в Repository
         |
         v
   Workflow Monitor читает из Repository (через Repository Service)
         |
         v
   Отображение статуса, логов, метрик
```

Важно: **данные ETL** (строки из таблиц, содержимое файлов) проходят через Integration Service и не хранятся в репозитории. Репозиторий хранит только метаданные — «инструкции», по которым Integration Service выполняет извлечение, преобразование и загрузку.

---

## 1.3.10. Связность клиентов и серверов

Клиентские приложения (Designer, Workflow Manager, Workflow Monitor) устанавливаются на рабочих станциях разработчиков. Для работы им необходима сетевая связность:

| Направление | Протокол / способ |
|-------------|-------------------|
| **Клиент → Domain (Gateway Node)** | TCP/IP; клиент подключается к домену для аутентификации и маршрутизации. |
| **Клиент → Repository Service** | Через Domain; запросы метаданных (чтение, запись). |
| **Клиент → Integration Service** | Через Domain; запуск workflow, получение статуса. |
| **Клиент → Sources/Targets** | ODBC/JDBC или нативные драйверы; для импорта метаданных источников и приёмников (структура таблиц, колонки) Designer подключается к БД. |

Integration Service при выполнении сессии подключается к источникам и приёмникам напрямую (по Connection objects); данные не проходят через клиентские машины. Flat file targets по умолчанию создаются на сервере, где работает Integration Service; при необходимости файлы можно передать по FTP или другим способом.

---

## 1.3.11. Типичные ошибки при понимании архитектуры

- **Путать Repository и Integration Service:** Repository хранит метаданные; Integration Service выполняет ETL. Без Repository Service клиенты не смогут читать/писать метаданные; без Integration Service workflow не будут выполняться.
- **Считать, что данные хранятся в репозитории:** репозиторий хранит только метаданные (определения маппингов, workflow, соединений); сами данные проходят через Integration Service от источников к приёмникам.
- **Игнорировать роль Domain:** все клиенты и сервисы работают в контексте домена; при сбое Gateway Node или Service Manager доступ к платформе нарушается.
- **Ожидать, что Designer выполняет маппинги:** Designer только создаёт и сохраняет метаданные; выполнение — прерогатива Integration Service.

---

## Ключевое

- **Архитектура PowerCenter** — Service Oriented Architecture: клиентские приложения (Designer, Workflow Manager, Workflow Monitor) и серверные сервисы (Repository Service, Integration Service) взаимодействуют через репозиторий метаданных.
- **Domain** — базовая административная единица; объединяет Nodes, Gateway Node, Service Manager; точка входа для всех клиентов.
- **Repository** — реляционная БД с метаданными (Sources, Targets, Mappings, Workflows, Connections); **Repository Service** управляет доступом, блокировками, версионированием.
- **Integration Service** — исполняющий движок: читает метаданные из репозитория, выполняет сессии (извлечение, преобразование, загрузка), записывает статус и логи.
- **Designer** — создание маппингов; **Workflow Manager** — создание workflow и расписаний; **Workflow Monitor** — мониторинг выполнения. Данные ETL не хранятся в репозитории; они проходят через Integration Service от источников к приёмникам.

В [§1.4](chapter-01-04.md) мы сопоставим этапы ETL (Extract, Transform, Load) с компонентами Informatica и свяжем материал с теорией ETL из других книг серии.
