# §10.3 Параметры и переопределение

В [§10.2](chapter-10-02.md) мы рассмотрели запуск и мониторинг. **Parameter file** и переопределение параметров позволяют менять значения при запуске без изменения метаданных — для разных окружений (DEV/TEST/PROD), разных источников и целей. В этом разделе — настройка parameter file (Workflow/Session), переопределение через pmcmd, переопределение connection attributes, mapping parameters в Session Config Object и условный запуск. Подробнее обработка сбоев — в [§10.4](chapter-10-04.md). См. [Глоссарий](glossary.md).

---

## 10.3.1. Parameter file: где задаётся

Parameter file можно указать на уровне **Workflow** или **Session**. Integration Service применяет значения при запуске; при конфликте (оба уровня задают один параметр) действует приоритет. Источник: [Parameter Files Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/parameter-files-overview.html).

**Workflow:** Workflow Manager → Workflow → Edit → Properties → General Options → Parameter Filename. Путь — прямой или через service process variable (например, `$PMRootDir`). Источник: [Entering a Parameter File in the Workflow Properties](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/configuring-the-parameter-file-name-and-location/using-a-parameter-file-with-workflows-or-sessions/entering-a-parameter-file-in-the-workflow-properties.html).

**Session:** Session → Edit → Properties → General Options → Parameter Filename. Можно указать путь или workflow/worklet variable — тогда имя файла задаётся в workflow parameter file. Источник: [Entering a Parameter File in the Session Properties](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/configuring-the-parameter-file-name-and-location/using-a-parameter-file-with-workflows-or-sessions/entering-a-parameter-file-in-the-session-properties.html).

---

## 10.3.2. Переопределение через pmcmd

При запуске через **pmcmd** можно передать другой parameter file, переопределяя значение из свойств Workflow/Session. Опция `-paramfile` задаёт путь к файлу. Источник: [Using a Parameter File with pmcmd](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/configuring-the-parameter-file-name-and-location/using-a-parameter-file-with-pmcmd.html).

**startworkflow:**

```bash
pmcmd startworkflow -uv USERNAME -pv PASSWORD -s SALES:6258 -f east -w wSalesAvg -paramfile '$PMRootDir/myfile.txt' workflowA
```

**starttask:**

```bash
pmcmd starttask -uv USERNAME -pv PASSWORD -s SALES:6258 -f east -w wSalesAvg -paramfile '$PMRootDir/myfile.txt' taskA
```

Опция `-localparamfile` задаёт parameter file на локальной машине, когда нет доступа к файлам на Integration Service node. Источник: [Using a Parameter File with pmcmd](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/configuring-the-parameter-file-name-and-location/using-a-parameter-file-with-pmcmd.html).

---

## 10.3.3. Переопределение connection attributes

Если connection задан через session parameter (например, `$DBConnectionSource`, `$FTPConnectionMyFTP`), можно переопределить **атрибуты connection** в parameter file. Поддерживаются типы: FTP (`$FTPConnection*`), Queue (`$QueueConnection*`), Loader (`$LoaderConnection*`), Application (`$AppConnection*`). Источник: [Overriding Connection Attributes in the Parameter File](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/overriding-connection-attributes-in-the-parameter-file.html).

Шаблон **ConnectionParam.prm** (в каталоге `server/bin`) перечисляет атрибуты для каждого типа connection. Копируют нужный фрагмент в parameter file и подставляют значения. Источник: [Overriding Connection Attributes in the Parameter File](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/overriding-connection-attributes-in-the-parameter-file.html).

**Пример для FTP:** переопределение Remote Filename и Is Staged:

```text
[MyFolder.WF:wf_MyWorkflow.ST:s_MySession]
$FTPConnectionMyFTPConn=FTP_Conn1
$Param_FTPConnectionMyFTPConn_Remote_Filename=ftp_src.txt
$Param_FTPConnectionMyFTPConn_Is_Staged=YES
```

Пробелы или кавычки до/после знака `=` интерпретируются как часть имени или значения. Если атрибут не задан, используется значение из connection object. Источник: [Overriding Connection Attributes in the Parameter File](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/overriding-connection-attributes-in-the-parameter-file.html).

Для **Relational** connections переопределение connection object выполняется через `$DBConnection*`: в parameter file указывают имя другого connection (например, `$DBConnectionSource=conn_oracle_prod`). См. [§2.3.3](chapter-02-03.md).

---

## 10.3.4. Mapping parameters в Session Config Object

**Mapping parameters** и **mapping variables** задаются в Designer; значения при запуске — в Session (Config Object) или parameter file. Источник: [§2.3.5](chapter-02-03.md).

В Workflow Manager: Session → Edit → Mapping tab → Config Object → Parameter and Variable values. Здесь задают значения для mapping parameters и mapping variables; можно использовать session parameter (например, `$ParamTableOwner`), значение которого берётся из parameter file. Источник: [Working with Session Parameters](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameters-and-variables-in-sessions/working-with-session-parameters.html).

В parameter file — в секции session:

```text
[FolderName.WorkflowName.SessionName]
$ParamTableOwner=SCHEMA_PROD
$ParamCutoffDate=2025-02-16
```

Mapping parameters используются в выражениях маппинга (Filter, Expression, SQL-override). См. [§5.4](chapter-05-04.md).

---

## 10.3.5. Условный запуск

**Условный запуск** — выполнение задачи только при выполнении условия. Реализуется через **link conditions** (условия на связях между задачами) и **workflow variables**. См. [§9.4](chapter-09-04.md).

**Link condition:** выражение на связи; если True — следующая задача выполняется; если False — пропускается. Используются предопределённые переменные (например, `$s_SessionName.TgtFailedRows`) или workflow variables. Источник: [Creating Link Conditions](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/creating-link-conditions.html).

**Workflow variables** в parameter file позволяют менять логику: например, `$ParamRunStaging=YES` — в link condition проверять эту переменную и выполнять Session загрузки staging только при `YES`. Источник: [Parameter Files Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/parameter-files-overview.html).

---

## 10.3.6. Типичные ошибки

- **Регистр в parameter file:** имена папок, workflow, session чувствительны к регистру; несовпадение — параметр не применится.
- **Путь к parameter file:** при pmcmd путь должен быть доступен на Integration Service node; `$PMRootDir` и другие переменные разрешаются на сервере.
- **Дублирование параметров:** при задании на Workflow и Session — приоритет у Session; при pmcmd -paramfile — переопределяет оба.
- **Connection attributes:** шаблон ConnectionParam.prm — точные имена; опечатка в `$Param_FTPConnection*` — атрибут не применится.

---

## Ключевое

- **Parameter file** задаётся в Workflow или Session (Properties → General Options → Parameter Filename).
- **pmcmd** `-paramfile` и `-localparamfile` переопределяют parameter file при запуске.
- **Connection attributes** переопределяются для FTP, Queue, Loader, Application через `$Param_*ConnectionName_*`; шаблон — ConnectionParam.prm.
- **Mapping parameters** — в Session Config Object или parameter file; используются в выражениях маппинга.
- **Условный запуск** — link conditions и workflow variables; значения из parameter file.

В [§10.4](chapter-10-04.md) мы разберём обработку сбоев и retry: настройка повторных попыток на Session, уведомления, best practices.
