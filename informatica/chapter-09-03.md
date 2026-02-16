# §9.3 Worklets и параметры

В [§9.2](chapter-09-02.md) мы рассмотрели Session и связь с маппингом. **Worklet** — объект, представляющий набор задач для переиспользования логики workflow в нескольких workflow. **Параметры** Workflow и Session позволяют менять значения между запусками. В этом разделе — Worklet (создание, вложение, вложенные worklets), workflow variables, session parameters и parameter file. Подробнее parameter file — в [§10.3](chapter-10-03.md). См. [Глоссарий](glossary.md).

---

## 9.3.1. Worklet: определение

**Worklet** — объект, представляющий набор задач, созданных для переиспользования логики workflow в нескольких workflow. Источник: [Working with Worklets](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets.html).

Worklet создаётся в **Worklet Designer**. Для запуска worklet его включают в workflow (parent workflow). Integration Service при выполнении worklet разворачивает его и выполняет все задачи и связи внутри worklet; информация о выполнении записывается в workflow log. Источник: [Working with Worklets](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets.html).

---

## 9.3.2. Reusable и non-reusable Worklet

| Тип | Создание | Использование |
|-----|----------|---------------|
| **Reusable** | Worklet Designer → Worklet → Create | В нескольких workflow; отображается в Navigator → Worklets |
| **Non-reusable** | Workflow Designer или Worklet Designer → Tasks → Create → Worklet | Только в том workflow/worklet, где создан |

Reusable worklet — для общей логики (например, загрузка staging, уведомления). Non-reusable — для логики, специфичной для одного workflow. Источник: [Creating a Reusable Worklet](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets/creating-a-reusable-worklet.html).

---

## 9.3.3. Вложение Worklet

Worklet вкладывается в workflow: перетаскивание из Navigator или Tasks → Insert Worklet. Workflow, содержащий worklet, называется **parent workflow**. Источник: [Working with Worklets](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets.html).

**Вложение worklet в worklet (nesting):** worklet может содержать другой worklet. Integration Service выполняет вложенный worklet из родительского worklet. Источник: [Nesting Worklets](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets/nesting-worklets.html).

**Пример:** worklet загрузки в staging; вложенный worklet загрузки из staging в DWH. Или группировка нескольких worklet по функции в один worklet для упрощения сложного workflow. Источник: [Nesting Worklets](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets/nesting-worklets.html).

---

## 9.3.4. Worklet Designer

Worklet Designer создаёт Start task при создании worklet. Далее добавляют Session, Command, Decision и др.; связывают задачи (Link Task). Worklet Designer аналогичен Workflow Designer по структуре. Источник: [Creating a Reusable Worklet](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/working-with-worklets/creating-a-reusable-worklet.html).

---

## 9.3.5. Параметры Workflow и Session

**Parameter file** — список параметров и переменных с значениями; определяет свойства для workflow, worklet или session. Integration Service применяет значения при запуске workflow/session. Источник: [Parameter Files Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/parameter-files-overview.html).

| Уровень | Тип | Примеры |
|---------|-----|---------|
| **Workflow** | Workflow variables | $ParamWorkflowVar — переменные workflow |
| **Worklet** | Worklet variables | Переменные worklet; при вложении получают значения из parent workflow или parameter file |
| **Session** | Session parameters | $DBConnectionSource, $InputFile*, $OutputFile*, $Param* |

Parameter file может содержать секции для workflow, worklet и session. Заголовок секции идентифицирует объект: `[FolderName.WorkflowName]`, `[FolderName.WorkflowName.SessionName]`. Источник: [Parameter Files Overview](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameter-files/parameter-files-overview.html), [§2.3.6](chapter-02-03.md).

---

## 9.3.6. Session parameters: кратко

**User-defined** — значения из parameter file; префиксы: $DBConnection*, $InputFile*, $OutputFile*, $Param*, $PMSessionLogFile, $DynamicPartitionCount. Источник: [Working with Session Parameters](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameters-and-variables-in-sessions/working-with-session-parameters.html).

**Built-in** — задаются Integration Service; нельзя переопределить. Примеры: $PMFolderName, $PMSessionName, $PMMappingName, $PMWorkflowRunId, $PMSourceQualifierName@numAffectedRows. Используются в post-session commands, email. Источник: [§2.3.4](chapter-02-03.md).

Имена в parameter file **чувствительны к регистру**. Parameter file указывается в свойствах Workflow или Session; при запуске через pmcmd можно передать другой parameter file. Источник: [Working with Session Parameters](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/parameters-and-variables-in-sessions/working-with-session-parameters.html).

---

## 9.3.7. Типичные ошибки

- **Worklet без Start:** Worklet Designer создаёт Start при создании; не удалять.
- **Циклы в связях:** PowerCenter не допускает циклов; каждая связь выполняется один раз.
- **Переменные worklet без значения:** при вложении worklet переменные должны получать значения из parent workflow или parameter file.
- **Регистр в parameter file:** имена папок, workflow, session чувствительны к регистру; несовпадение — параметр не применится.

---

## Ключевое

- **Worklet** — переиспользуемая группа задач workflow; создаётся в Worklet Designer; включается в workflow (parent workflow).
- **Reusable** — в нескольких workflow; **non-reusable** — только в одном.
- **Nesting** — worklet может содержать другой worklet.
- **Parameter file** — workflow variables, worklet variables, session parameters; заголовки секций; чувствительность к регистру.
- **Session parameters:** user-defined (из parameter file) и built-in (Integration Service).

В [§9.4](chapter-09-04.md) мы разберём зависимости и связи между задачами: Link Task, условные связи, порядок выполнения.
