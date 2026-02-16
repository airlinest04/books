# §10.2 Запуск и мониторинг

В [§10.1](chapter-10-01.md) мы рассмотрели расписание. **Запуск** workflow возможен вручную или по расписанию; **мониторинг** — через Workflow Monitor. В этом разделе — ручной запуск (Workflow Manager, Workflow Monitor, pmcmd), Workflow Monitor (представления, окна), статусы workflow и задач, Session Log и Workflow Log, Stop и Abort. Подробнее параметры — в [§10.3](chapter-10-03.md). См. [Глоссарий](glossary.md).

---

## 10.2.1. Ручной запуск

Workflow можно запустить вручную из Workflow Manager, Workflow Monitor или через **pmcmd**. Источник: [Manual Workflow Runs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/manual-workflow-runs.html).

**Workflow Manager / Workflow Monitor:** правый клик по workflow в Navigator → Start Workflow или Start Advanced Workflow. Start Advanced Workflow позволяет переопределить Integration Service, OS Profile, выбрать instance при concurrent execution. Источник: [Running an Entire Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/manual-workflow-runs/running-an-entire-workflow.html).

Перед запуском необходимо выбрать Integration Service (в свойствах workflow или через Assign Integration Service). Источник: [Manual Workflow Runs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/manual-workflow-runs.html).

Можно запускать весь workflow, часть workflow или отдельную задачу. Источник: [Manual Workflow Runs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/manual-workflow-runs.html).

---

## 10.2.2. Workflow Monitor: обзор

**Workflow Monitor** — клиентское приложение для мониторинга выполнения workflow. Отображает workflow, выполнившиеся хотя бы один раз. Источник: [Workflow Monitor Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-monitor-overview.html).

**Окна:**
- **Navigator** — репозитории, Integration Services, объекты.
- **Output** — сообщения от Integration Service и Repository Service.
- **Properties** — детали по сервисам, workflow, worklet, задачам.
- **Time** — прогресс выполнения workflow.

**Представления:** Gantt Chart (хронология) и Task view (отчёт по workflow run). Переключение — вкладки внизу. Источник: [Workflow Monitor Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-monitor-overview.html).

Workflow Monitor получает данные от Integration Service и Repository Service; читает историю из репозитория. Время отображается относительно часов Integration Service node. Источник: [Workflow Monitor Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-monitor-overview.html).

---

## 10.2.3. Статусы workflow и задач

Источник: [Workflow and Task Status](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-and-task-status.html).

| Статус | Описание |
|--------|----------|
| **Running** | Integration Service выполняет workflow или задачу |
| **Succeeded** | Успешное завершение |
| **Failed** | Ошибка при выполнении; recovery недоступен |
| **Stopped** | Остановка пользователем (Stop); recovery возможен при включённой опции |
| **Aborted** | Прерывание пользователем (Abort); DTM process завершён; recovery возможен |
| **Suspended** | Приостановка при ошибке задачи (Suspend on Error); recovery возможен |
| **Terminated** | Неожиданное завершение Integration Service; recovery возможен |
| **Waiting** | Ожидание ресурсов (например, лимит concurrent sessions) |
| **Scheduled** | Запланирован на будущее время |
| **Disabled** | Workflow/задача отключены; не выполняются |

**Промежуточные:** Aborting, Stopping, Suspending, Terminating, Preparing to Run. **Unknown Status** — при потере связи с Integration Service или таймауте. Источник: [Workflow and Task Status](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-and-task-status.html).

---

## 10.2.4. Stop и Abort

**Stop:** Integration Service останавливает задачу и все задачи в её цепочке; параллельные задачи продолжают выполняться. Источник: [Workflow and Task Status](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-and-task-status.html).

**Abort:** Integration Service завершает DTM process и прерывает задачу. Источник: [Workflow and Task Status](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-and-task-status.html).

Stop и Abort доступны в Workflow Monitor или через pmcmd. См. [§1.3.7](chapter-01-03.md).

---

## 10.2.5. Session Log и Workflow Log

**Session Log** — лог выполнения сессии: чтение/запись строк, ошибки трансформаций, производительность. Источник: [Session and Workflow Logs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-and-workflow-logs.html).

**Workflow Log** — лог выполнения workflow: задачи, связи, ошибки. Источник: [Session and Workflow Logs](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-and-workflow-logs.html).

Integration Service генерирует log events; Log Agent собирает их на узлах. Workflow Monitor отображает логи через Log Events window; можно просматривать текстовые log files. Источник: [Workflow Monitor Overview](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-monitor-overview.html), [§1.3.7](chapter-01-03.md).

---

## 10.2.6. Просмотр выполненных записей

В Workflow Monitor: Task view — отчёт по workflow run; фильтрация по статусу (Edit → List Tasks в Gantt Chart view). Properties window — детали выбранного workflow или задачи: время выполнения, количество строк, ошибки. Источник: [Workflow and Task Status](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-and-task-status.html).

Session Log содержит метрики: Source rows read, Target rows loaded, Rejected rows и др. См. [§11.4](chapter-11-04.md).

---

## 10.2.7. Типичные ошибки

- **Workflow не отображается в Monitor:** Workflow Monitor показывает только workflow, выполнившиеся хотя бы один раз.
- **Unknown Status:** проверить доступность Integration Service, таймауты, сетевое соединение.
- **Путать Stop и Abort:** Stop — корректная остановка; Abort — принудительное завершение процесса.
- **Искать логи не в том timezone:** время в Workflow Monitor — относительно Integration Service node.

---

## Ключевое

- **Запуск:** Workflow Manager, Workflow Monitor, pmcmd; Start Workflow / Start Advanced Workflow.
- **Workflow Monitor:** Navigator, Output, Properties, Time; Gantt Chart и Task view.
- **Статусы:** Running, Succeeded, Failed, Stopped, Aborted, Suspended, Terminated, Waiting.
- **Stop** — остановка цепочки; **Abort** — принудительное завершение DTM.
- **Session Log** и **Workflow Log** — анализ ошибок и производительности.

В [§10.3](chapter-10-03.md) мы разберём параметры и переопределение: Parameter File, переопределение connections, условный запуск.
