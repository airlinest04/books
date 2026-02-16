# §2.3 Connections и переменные

В [§2.2](chapter-02-02.md) мы разобрали папки и объекты репозитория. **Connection** (подключение) — объект, определяющий параметры доступа к БД или файлам; Integration Service использует его при выполнении сессий для извлечения и загрузки данных. **Переменные** и **параметры** позволяют менять значения между запусками (connections, пути к файлам, настройки) без изменения метаданных. В этом разделе мы рассмотрим типы connections, создание connection к БД, переменные $Source и $Target, session parameters, mapping parameters/variables и parameter file. Подробнее переопределение при развёртывании — в [§10.3](chapter-10-03.md). См. [Глоссарий](glossary.md).

---

## 2.3.1. Connection: назначение и типы

**Connection** (Connection object) — глобальный объект репозитория, определяющий параметры подключения к внешнему ресурсу. Integration Service использует connection при выполнении сессии: для чтения из реляционных источников, записи в приёмники, доступа к Lookup-таблицам, вызова Stored Procedure, работы с файлами через FTP. Connections создаются и редактируются в **Workflow Manager**; права на них назначаются отдельно (как для глобальных объектов). Источник: [Connection Objects Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview.html).

Типы connections (по документации Informatica). Источник: [Connection Types](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview/connection-types.html).

| Тип | Описание |
|-----|----------|
| **Relational** | Подключение к реляционной БД (Oracle, SQL Server, PostgreSQL, DB2 и др.) для источников, приёмников, Lookup, Stored Procedure. Подтипы: Oracle, Microsoft SQL Server, ODBC и др. |
| **FTP** | Подключение к FTP или SFTP для доступа к файлам (источники, приёмники) через FTP. |
| **Loader** | Подключение к внешнему загрузчику (Teradata FastLoad, DB2 Autoloader, Oracle SQL*Loader и др.) для bulk load. |
| **Queue** | Подключение к очереди сообщений (WebSphere MQ, MSMQ). |
| **Application** | Подключение к приложению (Netezza, SAP NetWeaver, PowerExchange и др.). |
| **None** | Для плоских файлов и XML, хранящихся локально; connection не требуется. |

Для типичного сценария ETL (таблицы БД → таблицы БД) используются **Relational** connections для источника и приёмника. Для файлов на локальной файловой системе Integration Service — connection не нужен; путь указывается в настройках сессии. См. [§3.4](chapter-03-04.md).

---

## 2.3.2. Создание Relational connection

Пошаговая инструкция создания connection к тестовой БД (задание из TOC):

1. Открыть **Workflow Manager**, подключиться к репозиторию.
2. Меню **Connections → Relational** (или контекстное меню в дереве Connections).
3. В диалоге **Connection Browser** нажать **New**.
4. Заполнить параметры:
   - **Connection name** — уникальное имя (например, `conn_oracle_dev`).
   - **Username** — пользователь БД.
   - **Password** — пароль (при использовании OS profile или parameter file можно оставить пустым).
   - **Connect string** — строка подключения (хост, порт, имя службы/базы). Для Oracle — `host:port/service_name`; для SQL Server — иначе. Зависит от СУБД и драйвера.
5. При необходимости выбрать **Native connect string** или **ODBC**.
6. Сохранить connection.

Connection сохраняется в репозитории как глобальный объект. В настройках Session (вкладка Mapping → Connections) для каждого источника и приёмника выбирается connection. Integration Service при запуске сессии подключается к БД по указанному connection. Подробнее строки подключения — в документации Informatica по [Native Connect Strings](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview/native-connect-strings.html).

---

## 2.3.3. $Source и $Target: встроенные переменные подключения

**$Source** и **$Target** — встроенные переменные PowerCenter, указывающие, какой connection использовать для операций с источником и приёмником. Источник: [Connection Variable Values](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview/connection-variable-values.html).

- **$Source** — connection для реляционных источников, Lookup и Stored Procedure (если в маппинге указано использование $Source).
- **$Target** — connection для реляционных приёмников.

В настройках Session (Properties или Mapping → Connections) задаётся **$Source Connection Value** и **$Target Connection Value**: выбирается конкретный connection object или session parameter (например, `$DBConnectionSource`), значение которого задаётся в parameter file. Это позволяет менять connection между окружениями (DEV/TEST/PROD) без изменения Session.

Если $Source или $Target не заданы явно, Integration Service определяет connection автоматически:
- один источник → connection, указанный для источника;
- один приёмник → connection, указанный для приёмника;
- несколько приёмников или unconnected Lookup → сессия завершится с ошибкой, нужно задать явно.

---

## 2.3.4. Session parameters: пользовательские и встроенные

**Session parameters** — значения, которые могут меняться между запусками сессии. Источник: [Working with Session Parameters](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameters-and-variables-in-sessions/working-with-session-parameters.html).

**Пользовательские (user-defined)** параметры задаются в Session или Workflow и получают значения из **parameter file**. Примеры:

| Префикс / тип | Пример | Описание |
|---------------|--------|----------|
| **$DBConnection*** | $DBConnectionSource | Connection к реляционной БД (источник, приёмник, Lookup, Stored Procedure). |
| **$AppConnection*** | $AppConnectionSAP | Connection к приложению. |
| **$InputFile*** | $InputFileCustomers | Имя файла-источника. |
| **$OutputFile*** | $OutputFileResults | Имя файла-приёмника. |
| **$Param*** | $ParamTableOwner | Общий параметр (владелец таблицы, префикс, путь и т.д.). |
| **$PMSessionLogFile** | — | Имя файла лога сессии. |
| **$DynamicPartitionCount** | — | Количество партиций. |

**Встроенные (built-in)** параметры задаются Integration Service при выполнении; их нельзя переопределить в parameter file. Примеры: `$PMFolderName`, `$PMSessionName`, `$PMMappingName`, `$PMWorkflowRunId`, `$PMSourceQualifierName@numAffectedRows` (количество прочитанных строк) и др. Используются в post-session commands, email, SQL.

---

## 2.3.5. Mapping parameters и mapping variables

**Mapping parameter** — параметр, определённый в маппинге (Designer); значение задаётся при выполнении сессии (в Session или parameter file). Используется в выражениях трансформаций: фильтры, формулы, SQL-override. Позволяет параметризовать логику маппинга (например, дата среза, код региона).

**Mapping variable** — переменная в маппинге; в отличие от параметра может изменяться во время выполнения (например, счётчик). Имеет начальное значение и правила обновления (Set Variable).

И параметры, и переменные маппинга задаются в Designer; значения при запуске — в Session (Config Object → Parameter and Variable values) или в parameter file. Подробнее — в [§5.4](chapter-05-04.md), [§10.3](chapter-10-03.md).

---

## 2.3.6. Parameter file: структура и использование

**Parameter file** — текстовый файл со списком параметров и переменных и их значений. Формат: `name=value` на каждой строке; группы параметров предваряются заголовком, идентифицирующим workflow или session. Источник: [Parameter Files Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/parameter-files-overview.html).

Пример структуры:

```text
[FolderName.WorkflowName]
$ParamWorkflowVar=value1

[FolderName.WorkflowName.SessionName]
$DBConnectionSource=conn_oracle_dev
$DBConnectionTarget=conn_postgres_prod
$ParamLoadDate=2025-02-16
```

Имена папок и сессий в parameter file **чувствительны к регистру**. Integration Service читает parameter file при старте workflow/session и подставляет значения вместо параметров. Parameter file указывается в свойствах Workflow или Session; при запуске через pmcmd можно передать другой parameter file, переопределяющий настройки.

Типичные сценарии:
- **Разные окружения:** один маппинг и session, разные parameter files для DEV/TEST/PROD с разными connections.
- **Динамические пути и даты:** `$InputFileSales=/data/sales_$ParamLoadDate.csv`; `$ParamLoadDate` задаётся при запуске.
- **Переопределение connection attributes:** в parameter file можно переопределить отдельные атрибуты connection (хост, пользователь) без создания нового connection object.

---

## 2.3.7. Workflow variables

**Workflow variable** — переменная на уровне workflow; используется в задачах (Command, Decision, Email) и может передаваться в session через parameter assignment. Workflow variables задаются в Workflow Manager; значения — в parameter file под заголовком `[FolderName.WorkflowName]` или через Variable Assignment в workflow.

Worklet может иметь свои переменные; при вложении worklet в workflow переменные worklet получают значения из workflow или parameter file. Подробнее — в [§9.3](chapter-09-03.md).

---

## 2.3.8. Переопределение connection в parameter file

В parameter file можно переопределить атрибуты connection (хост, порт, пользователь, пароль, connect string) без создания нового connection object. Это удобно при развёртывании в разные окружения: один connection в репозитории, фактические значения — в parameter file для каждого окружения.

Синтаксис (упрощённо):

```text
[FolderName.SessionName]
$DBConnectionSource.UserName=dev_user
$DBConnectionSource.Password=dev_pass
$DBConnectionSource.ConnectString=dev_host:1521/devdb
```

Integration Service при выполнении использует переопределённые значения вместо сохранённых в connection object. Подробнее — в документации [Overriding Connection Attributes in the Parameter File](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/overriding-connection-attributes-in-the-parameter-file.html).

---

## 2.3.9. Безопасность: пароли и credentials

Пароли в connection object могут храниться в репозитории (зашифрованными) или извлекаться из **OS Profile** (учётные данные ОС). Для повышения безопасности рекомендуется:

- использовать **parameter file** с паролями, хранящийся в защищённом месте с ограниченным доступом;
- или **OS Profile** — Integration Service получает credentials из учётной записи ОС, под которой запущен;
- не хранить пароли в открытом виде в репозитории при возможности.

Parameter file с паролями не должен попадать в системы контроля версий с открытым доступом. См. документацию Informatica по [Database User Names and Passwords](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/connection-objects/connection-objects-overview/database-user-names-and-passwords.html).

---

## 2.3.10. Типичные ошибки при работе с connections и переменными

- **Не задать connection для сессии:** при реляционных источнике и приёмнике обязательно указать connection в Session; иначе сессия завершится с ошибкой.
- **Путать Source/Target definition и Connection:** Source (в Designer) — структура таблицы; Connection (в Workflow Manager) — параметры доступа. Оба нужны: структура для маппинга, connection для выполнения.
- **Ошибки в регистре в parameter file:** имена папок и сессий чувствительны к регистру; неверный регистр — параметры не применятся.
- **Не определить user-defined parameter в parameter file:** если в Session используется `$DBConnectionSource`, значение должно быть в parameter file (или задано в Session); иначе сессия может упасть или использовать пустую строку.
- **Использовать built-in параметр в parameter file:** встроенные параметры задаются Integration Service; попытка задать их в parameter file не имеет эффекта.

---

## Ключевое

- **Connection** — глобальный объект с параметрами подключения к БД, FTP, Loader, Queue, Application. Создаётся в Workflow Manager; используется Integration Service при выполнении сессий.
- **Типы:** Relational (Oracle, SQL Server, ODBC и др.), FTP, Loader, Queue, Application. Для локальных файлов — None.
- **$Source и $Target** — встроенные переменные для указания connection источника и приёмника; задаются в Session или через session parameter.
- **Session parameters:** user-defined (значения из parameter file) и built-in (задаются Integration Service). Префиксы: $DBConnection*, $InputFile*, $OutputFile*, $Param* и др.
- **Mapping parameters/variables** — параметризация логики маппинга; значения при запуске — в Session или parameter file.
- **Parameter file** — текстовый файл `name=value`; группы по заголовкам [Folder.Workflow.Session]. Позволяет менять connections, пути, даты между запусками и окружениями.
- **Безопасность:** пароли в parameter file или OS Profile; не хранить в открытом виде в репозитории.

В [§2.4](chapter-02-04.md) мы разберём версионирование и развёртывание: экспорт/импорт, миграция между окружениями, best practices.
