# §9.2 Session: связь с маппингом

В [§9.1](chapter-09-01.md) мы рассмотрели Workflow. **Session** — задача workflow, содержащая инструкции о том, как Integration Service перемещает данные по маппингу. В этом разделе — связь Session и Mapping, создание Session, вкладки свойств, настройка Connections и переопределение параметров. Подробнее параметры — в [§10.3](chapter-10-03.md). См. [Глоссарий](glossary.md).

---

## 9.2.1. Session: определение

**Session** — набор инструкций, сообщающих Integration Service, **как** и **когда** перемещать данные из источников в приёмники. Session — тип задачи (task), как и другие задачи Workflow Manager. Источник: [Sessions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/sessions-overview.html).

Integration Service использует инструкции, настроенные в Session и Mapping: Session задаёт connections, пути к файлам, режим загрузки; Mapping — логику трансформаций. Источник: [Creating Sessions and Workflows](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows.html).

---

## 9.2.2. Session и Mapping: связь

| Объект | Где создаётся | Что описывает |
|--------|---------------|---------------|
| **Mapping** | Designer | Логика ETL: источники, трансформации, приёмники, связи |
| **Session** | Workflow Manager | Параметры выполнения: Connection, пути, партиционирование |

Session **ссылается** на Mapping. Один Mapping может использоваться в нескольких Session (например, с разными Connection для dev и prod). Mapping не знает о Connection; Session привязывает маппинг к конкретным источникам и приёмникам. Источник: [§4.1.8](chapter-04-01.md), [Session Task](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/session-task.html).

При создании Session выбирают Mapping; после изменения Mapping Workflow Manager может выдать предупреждение о невалидности Session. Источник: [Editing a Session](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/editing-a-session.html).

---

## 9.2.3. Создание Session

Session создают в **Task Developer** (reusable) или **Workflow Designer** (non-reusable). Источник: [Creating a Session Task](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/session-task/creating-a-session-task.html)

**Шаги:**
1. Tasks → Create → Session Task.
2. Ввести имя Session (символ точки в имени не допускается).
3. Выбрать Mapping из диалога Mappings.
4. Настроить Connections, Properties и др.

Reusable Session — в Task Developer; используется в нескольких workflow. Non-reusable — в Workflow Designer; только в одном workflow. Источник: [Creating a Reusable Session](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-reusable-session.html).

---

## 9.2.4. Вкладки свойств Session

При двойном щелчке по Session открываются свойства. Источник: [Editing a Session](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/editing-a-session.html).

| Вкладка | Назначение |
|---------|------------|
| **General** | Имя, Mapping, описание, Integration Service, ресурсы |
| **Properties** | Лог файл, test load, сортировка, производительность |
| **Config Object** | Продвинутые настройки, логи, обработка ошибок |
| **Mapping** | Connections для источников и приёмников, переопределение свойств трансформаций, партиционирование |
| **Components** | Pre- и post-session команды, email-уведомления |

**Mapping tab** — ключевая для ETL: здесь задаются Connection для каждого источника (Source Qualifier), приёмника и Lookup. Источник: [Creating a Reusable Session](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-reusable-session.html).

---

## 9.2.5. Connections в Session

**Connection** — объект Workflow Manager с параметрами доступа к БД, FTP и др. Integration Service использует Connection при выполнении Session для чтения и записи. Источник: [§2.3](chapter-02-03.md).

На вкладке **Mapping** в Session для каждого источника (Source Qualifier), приёмника и Lookup выбирается Connection. Источник: [Creating a Reusable Session](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-reusable-session.html).

**$Source и $Target** — встроенные переменные; можно задать один Connection для всех источников ($Source) и один для всех приёмников ($Target). Значения — конкретный Connection или session parameter (например, `$DBConnectionSource`), задаваемый в parameter file. Источник: [§2.3.3](chapter-02-03.md).

**Target Load Type:** Normal (построчная запись с возможностью Update) или Bulk (массовая загрузка). Источник: [Creating a Reusable Session](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-reusable-session.html).

---

## 9.2.6. Переопределение параметров

Session может переопределять параметры, заданные в Mapping: source/target location, source/target type, error tracing, атрибуты трансформаций. Источник: [Sessions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/sessions-overview.html).

**Mapping parameters** — значения задаются в Session (Config Object → Parameter and Variable values) или в parameter file. Подробнее — в [§10.3](chapter-10-03.md).

---

## 9.2.7. Pre- и Post-Session команды

На вкладке **Components** настраиваются: pre-session shell commands, post-session shell commands, On-Success и On-Failure email. Источник: [Sessions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/sessions-overview.html), [Pre- and Post-Session Commands](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/pre--and-post-session-commands.html).

---

## 9.2.8. Типичные ошибки

- **Session без Connection для реляционного источника/приёмника:** при relational Source и Target обязательно указать Connection; иначе ошибка при выполнении.
- **Выбор несуществующего Mapping:** Mapping должен существовать в той же папке или в shared folder.
- **Точка в имени Session:** PowerCenter не допускает символ точки в имени Session task.
- **Изменение Mapping без проверки Session:** при изменении структуры Mapping (удаление портов, источников) Session может стать Invalid; проверить после изменений.

---

## Ключевое

- **Session** — инструкции для Integration Service: как перемещать данные по маппингу; ссылается на Mapping.
- **Mapping** — логика; **Session** — параметры выполнения (Connection, пути, режим загрузки).
- **Создание:** Task Developer (reusable) или Workflow Designer (non-reusable); выбор Mapping.
- **Mapping tab** — Connections для источников, приёмников, Lookup; $Source, $Target.
- **Вкладки:** General, Properties, Config Object, Mapping, Components.

В [§9.3](chapter-09-03.md) мы разберём Worklets и параметры: переиспользуемые группы задач, вложение worklet в workflow, параметры workflow.
