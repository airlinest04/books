# §9.1 Workflow: определение

В [§8.4](chapter-08-04.md) мы рассмотрели Mapplets. **Workflow** — набор инструкций, разбитых на задачи (tasks), которые Integration Service использует для извлечения, преобразования и загрузки данных. В этом разделе — определение Workflow, типы задач, Workflow Manager и Workflow Designer, связь с Session и Mapping. Подробнее Session — в [§9.2](chapter-09-02.md). См. [Глоссарий](glossary.md).

---

## 9.1.1. Определение Workflow

**Workflow** — набор инструкций, разбитых на задачи (tasks), которые Integration Service выполняет при запуске. Источник: [Objects Created in the Workflow Manager](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata/objects-created-in-the-workflow-manager.html).

Workflow определяет, **как** и **в каком порядке** выполнять задачи: сессии (перемещение данных), команды ОС, уведомления по email, условные ветвления и др. Источник: [Creating Sessions and Workflows](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows.html).

**Связь:** Workflow содержит Session tasks; каждая Session выполняет один Mapping. Mapping — метаданные (Designer); Session — инструкции по выполнению (Workflow Manager); Workflow — оркестрация задач. Источник: [Sessions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/sessions-overview.html).

---

## 9.1.2. Типы задач (Workflow tasks)

**Workflow tasks** — инструкции, которые Integration Service выполняет при запуске workflow. Источник: [Objects Created in the Workflow Manager](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata/objects-created-in-the-workflow-manager.html).

| Тип задачи | Назначение |
|------------|------------|
| **Start** | Точка входа workflow; все workflow начинаются с Start task |
| **Session** | Выполнение маппинга; перемещение данных из источников в приёмники |
| **Command** | Выполнение команды ОС (shell, batch) |
| **Email** | Отправка уведомления по email (On-Success, On-Failure) |
| **Decision** | Условное ветвление по результату предыдущей задачи |
| **Timer** | Задержка или ожидание до заданного времени |
| **Assignment** | Присвоение значения переменной workflow |
| **Worklet** | Переиспользуемая группа задач; вкладывается в workflow |

Session — основная задача для ETL; остальные — вспомогательные (уведомления, пред-/пост-обработка, ветвление). Источник: [Creating Sessions and Workflows](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows.html), [§1.3.6](chapter-01-03.md).

---

## 9.1.3. Workflow Manager и Workflow Designer

**Workflow Manager** — клиентское приложение PowerCenter для создания и редактирования workflow, задач, расписаний и connections. Источник: [§1.3](chapter-01-03.md), [Objects Created in the Workflow Manager](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata/objects-created-in-the-workflow-manager.html).

**Workflow Designer** — панель Workflow Manager для визуального построения workflow: перетаскивание задач, связывание (Link Task), настройка свойств. Источник: [Creating a Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-workflow.html).

**Создание workflow:**
1. Открыть Workflow Designer.
2. Перетащить Session (или другую задачу) из Navigator в рабочую область.
3. Связать Start с первой задачей; связать задачи между собой.
4. Выбрать Integration Service для выполнения.
5. Сохранить в репозиторий.

Все workflow начинаются с **Start task**; связи (links) определяют порядок выполнения. Источник: [Creating a Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-workflow.html).

---

## 9.1.4. Последовательное и параллельное выполнение

Workflow может содержать несколько Session tasks. Источник: [Sessions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sessions/sessions-overview.html).

| Режим | Реализация |
|-------|------------|
| **Последовательно** | Start → Session1 → Session2 → …; каждая следующая запускается после успешного завершения предыдущей |
| **Параллельно** | Несколько веток от Start или Decision; Integration Service выполняет ветки параллельно (при наличии ресурсов) |

Связи между задачами задают граф выполнения; условные ветви — через Decision task. Источник: [Creating Sessions and Workflows](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows.html).

---

## 9.1.5. Workflow и Integration Service

При запуске workflow пользователь (или расписание) указывает **Integration Service**, который будет выполнять задачи. Workflow Manager передаёт управление Integration Service; Integration Service читает метаданные workflow и session из репозитория, подключается к источникам и приёмникам по Connections, выполняет маппинги. Источник: [§1.3.6](chapter-01-03.md), [Creating a Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-workflow.html).

Workflow хранится в репозитории; при каждом запуске создаётся **workflow run** (экземпляр выполнения) с уникальным идентификатором. См. [§10.2](chapter-10-02.md).

---

## 9.1.6. Типичные ошибки

- **Workflow без Session:** для ETL нужна хотя бы одна Session task; workflow только с Command/Email не выполняет маппинг.
- **Session без Mapping:** Session должна ссылаться на существующий Mapping; иначе ошибка при сохранении или запуске.
- **Не связать Start с первой задачей:** без связи от Start задача не выполнится.
- **Integration Service не запущен:** при создании workflow выбирают Integration Service; если сервис не запущен, workflow не выполнится.

---

## Ключевое

- **Workflow** — набор инструкций (tasks) для Integration Service; оркестрация выполнения.
- **Типы задач:** Start, Session (маппинг), Command, Email, Decision, Timer, Assignment, Worklet.
- **Workflow Manager** — создание workflow; **Workflow Designer** — визуальное построение; связи между задачами.
- **Session** — задача выполнения маппинга; одна Session — один Mapping.
- **Последовательно/параллельно** — через связи и ветвление.

В [§9.2](chapter-09-02.md) мы разберём Session: связь с маппингом, настройки connections, параметры и переопределения.
