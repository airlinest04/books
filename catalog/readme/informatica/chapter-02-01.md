# §2.1 Repository Server

В [§1.4](chapter-01-04.md) мы сопоставили этапы ETL с компонентами Informatica и указали, что метаданные хранятся в репозитории. В [§1.3](chapter-01-03.md) мы ввели Repository и Repository Service в обзоре архитектуры. Теперь углубимся в **Repository Server**: что именно хранится в репозитории, как устроены домены и репозитории, как клиенты подключаются к репозиторию и какие сервисы обеспечивают работу. Вы научитесь понимать иерархию Domain → Repository Service → Repository и процесс подключения клиентов. Подробнее организация объектов в папках — в [§2.2](chapter-02-02.md). См. [Глоссарий](glossary.md).

---

## 2.1.1. Repository Server: хранение метаданных

**Repository Server** в контексте PowerCenter — это совокупность **Repository** (реляционная БД с метаданными) и **Repository Service** (процесс, управляющий доступом к этой БД). Термин «Repository Server» часто используют как синоним Repository Service или всей подсистемы хранения метаданных. См. [Глоссарий](glossary.md).

Репозиторий хранит **метаданные** — данные о данных: не сами строки ETL, а описания того, как их извлекать, преобразовывать и загружать. Метаданные описывают объекты, создаваемые в клиентских приложениях:

| Тип объекта | Где создаётся | Что хранится |
|-------------|---------------|--------------|
| **Source** | Designer (Source Analyzer) | Структура таблицы или файла: имена колонок, типы данных |
| **Target** | Designer (Target Designer) | Структура целевой таблицы или файла |
| **Transformation** | Designer (Transformation Developer, Mapping Designer) | Логика трансформации: порты, выражения, условия |
| **Mapping** | Designer (Mapping Designer) | Связи между источниками, трансформациями и приёмниками |
| **Workflow, Session, Schedule** | Workflow Manager | Определения workflow, задач, расписаний |
| **Connection** | Workflow Manager | Параметры подключения к БД и файлам (без паролей в открытом виде при корректной настройке) |
| **Folder** | Repository Manager | Организация объектов в папки |

Репозиторий также хранит права доступа пользователей, информацию о выполнении (логи workflow и сессий, статусы) и при включённом version control — несколько версий объектов. Integration Service использует эти метаданные при выполнении сессий. Источник: [Understanding the Repository Overview](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-the-repository-overview.html), [Understanding Metadata](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata.html).

---

## 2.1.2. Репозиторий как реляционная БД

Репозиторий физически — **реляционная база данных**. Поддерживаемые СУБД: Oracle, Microsoft SQL Server, Sybase, IBM DB2. При установке PowerCenter администратор создаёт пустую БД (или использует существующую) и запускает скрипты инициализации репозитория — создаются таблицы, в которых будут храниться метаданные.

Ключевые особенности:

- **Repository Service** общается с БД через **нативные драйверы** (не ODBC для основных операций); это обеспечивает производительность при большом объёме метаданных.
- **Клиенты** (Designer, Workflow Manager, Workflow Monitor, Integration Service) **не подключаются напрямую** к БД репозитория; они обращаются только к Repository Service по TCP/IP.
- Repository Service защищает метаданные: управляет подключениями, блокировками объектов, уведомляет о изменениях другими пользователями.

Один экземпляр Repository Service обслуживает **один** репозиторий (одну БД). В одном домене может быть несколько репозиториев — каждый со своим Repository Service. Источник: [Repository Architecture](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/repository-architecture.html).

---

## 2.1.3. Domain и иерархия сервисов

**Domain** (домен) — первичная административная единица PowerCenter. В домене объединены:

- **Nodes** (узлы) — логические представления серверов;
- **Gateway Node** — точка входа: принимает запросы от клиентов и направляет их к нужным сервисам;
- **Service Manager** — управляет доменом (аутентификация, запуск/остановка сервисов);
- **Application Services** — Repository Service, Integration Service, Reporting Service.

Иерархия:

```
Domain
  ├── Node 1 (Gateway Node)
  ├── Node 2
  ├── Node 3
  │
  ├── Repository Service A  →  Repository (БД) A
  ├── Repository Service B  →  Repository (БД) B
  ├── Integration Service X
  └── Integration Service Y
```

Один домен может содержать несколько Repository Service (и соответственно несколько репозиториев) и несколько Integration Service. Integration Service привязывается к репозиторию: он читает метаданные workflow и маппингов из конкретного репозитория. Клиент подключается к домену (указывает хост и порт Gateway Node), затем выбирает репозиторий для работы. См. [§1.3.2](chapter-01-03.md).

---

## 2.1.4. Процесс подключения клиента к репозиторию

Клиент (Designer, Workflow Manager, Workflow Monitor, pmrep, infacmd) подключается к репозиторию через следующую цепочку. Источник: [Repository Connectivity](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/repository-connectivity.html).

1. **Клиент отправляет запрос на подключение** к master gateway node (точка входа в домен). Указываются хост, порт домена, имя репозитория, учётные данные.

2. **Service Manager** возвращает клиенту хост и порт узла, на котором работает Repository Service для выбранного репозитория. При высокой доступности (HA) может быть настроен резервный узел.

3. **Клиент устанавливает соединение** с процессом Repository Service по TCP/IP.

4. **Repository Service** выполняет операции с БД репозитория от имени клиента: чтение и запись метаданных, блокировки, уведомления.

Порт Repository Service настраивается при установке; по умолчанию используется порт из диапазона, выделенного для PowerCenter. Клиенты общаются с Repository Service по TCP/IP; Repository Service общается с БД репозитория по нативному протоколу СУБД.

---

## 2.1.5. Repository Service: процессы и масштабирование

**Repository Service process** — экземпляр Repository Service, выполняющийся на конкретном узле. Один процесс обслуживает один репозиторий; при необходимости можно настроить **несколько процессов** Repository Service для одного репозитория на разных узлах — это повышает производительность и отказоустойчивость.

Repository Service — многопоточный процесс: получает запросы от клиентов, выполняет транзакции с БД репозитория, управляет блокировками. При одновременной работе многих разработчиков нагрузка на репозиторий растёт; несколько процессов распределяют её.

Важно: Repository Service **не хранит** метаданные в своей памяти — он лишь посредник между клиентами и БД. Все данные хранятся в таблицах репозитория.

---

## 2.1.6. Несколько репозиториев и Repository Domain

В одном домене может быть **несколько репозиториев**. Типичные сценарии:

- **Разделение по окружениям:** репозиторий DEV для разработки, TEST для тестирования, PROD для production; разные команды работают с разными репозиториями.
- **Разделение по проектам или подразделениям:** изоляция метаданных между командами.

**Repository Domain** (домен репозиториев) — группа репозиториев в клиенте PowerCenter, между которыми возможен обмен метаданными. Repository Domain использует **global repository** (глобальный репозиторий) — специальный репозиторий для обмена. При настройке shared folders объекты из папки в одном репозитории можно использовать в других репозиториях домена. Это позволяет переиспользовать Sources, Targets, Mapplets между проектами.

**Repository Domain** не следует путать с **PowerCenter Domain** (административным доменом с узлами и сервисами): это разные концепции. Repository Domain — логическая группировка репозиториев для обмена метаданными; PowerCenter Domain — инфраструктурная единица. Источник: [Understanding the Repository Overview](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-the-repository-overview.html).

---

## 2.1.7. Инструменты администрирования репозитория

| Инструмент | Назначение |
|------------|------------|
| **Repository Manager** | Создание папок, перемещение объектов, права доступа, экспорт/импорт; организация структуры репозитория. |
| **Informatica Administrator** (Admin Console) | Управление доменом: создание и настройка Repository Service, Integration Service, узлов; запуск и остановка сервисов; пользователи и группы. |
| **pmrep** | Командная строка для операций с метаданными: экспорт, импорт, сравнение, развёртывание; автоматизация и скрипты. |
| **infacmd** | Командная строка для управления сервисами: создание Repository Service, Integration Service, запуск workflow и т.д. |

Repository Manager — клиентское приложение для разработчиков и администраторов метаданных. Administrator — веб-консоль для инфраструктурных задач. pmrep и infacmd — для автоматизации и CI/CD. Подробнее экспорт, импорт и развёртывание — в [§2.4](chapter-02-04.md).

---

## 2.1.8. Version control и расширения метаданных

При наличии опции **team-based development** в репозитории можно включить **version control** (версионирование). В версионированном репозитории хранятся несколько версий одного объекта; доступны сравнение версий, отслеживание изменений, метки (labels) и развёртывание выбранной версии. Это поддерживает практики change management при командной разработке.

**Metadata extensions** — возможность связывать с объектами репозитория дополнительную информацию (например, имя автора, дата создания, кастомные атрибуты). Расширения не влияют на выполнение ETL, но полезны для документирования и аудита.

---

## 2.1.9. Типичные ошибки при работе с репозиторием

- **Путать Repository и Repository Service:** Repository — это БД с таблицами метаданных; Repository Service — процесс, обеспечивающий доступ к ней. Без запущенного Repository Service клиенты не смогут подключиться к репозиторию.
- **Подключаться напрямую к БД репозитория** для изменения метаданных: это обходит блокировки и целостность; изменения через клиенты (Designer, Workflow Manager, pmrep) или Repository Service.
- **Игнорировать блокировки:** при одновременном редактировании одного объекта другим пользователем Repository Service блокирует объект; нужно обновить или отменить изменения.
- **Путать PowerCenter Domain и Repository Domain:** первый — инфраструктура (узлы, сервисы); второй — логическая группа репозиториев для обмена метаданными.

---

## Ключевое

- **Repository Server** — Repository (реляционная БД с метаданными) + Repository Service (процесс доступа). Хранятся Sources, Targets, Mappings, Workflows, Connections, папки, права.
- **Репозиторий** — Oracle, SQL Server, Sybase, DB2; клиенты обращаются только через Repository Service по TCP/IP.
- **Domain** объединяет Nodes, Gateway Node, Service Manager, Repository Service, Integration Service. Один домен — несколько репозиториев и сервисов.
- **Подключение клиента:** запрос к Gateway Node → Service Manager возвращает хост/порт Repository Service → клиент соединяется с Repository Service → операции с БД.
- **Несколько репозиториев:** по окружениям (DEV/TEST/PROD) или проектам. **Repository Domain** — группа репозиториев с обменом через global repository.
- **Администрирование:** Repository Manager, Administrator, pmrep, infacmd. Version control и metadata extensions при соответствующих опциях.

В [§2.2](chapter-02-02.md) мы разберём организацию объектов в папках, типы объектов (Sources, Targets, Mappings, Workflows) и права доступа.
