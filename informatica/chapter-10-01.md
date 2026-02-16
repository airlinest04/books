# §10.1 Расписание (Schedule)

В [§9.4](chapter-09-04.md) мы рассмотрели зависимости и связи между задачами. **Schedule** (расписание) определяет, когда Integration Service запускает workflow. В этом разделе — Workflow Scheduler, Run Options, Schedule Options, Repeat Options (Customized Repeat), Start/End Options, reusable scheduler. Подробнее запуск и мониторинг — в [§10.2](chapter-10-02.md). См. [Глоссарий](glossary.md).

---

## 10.1.1. Workflow Scheduler: назначение

**Workflow Scheduler** — объект репозитория, содержащий настройки расписания: как и когда запускать workflow. Каждый workflow имеет связанный scheduler. Источник: [Workflow Schedulers](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-schedulers.html).

**По умолчанию** workflow выполняется **on demand** (по требованию): Integration Service запускает workflow при ручном старте. Источник: [Creating a Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/getting-started/tutorial-lesson-3/creating-sessions-and-workflows/creating-a-workflow.html).

Можно настроить: непрерывный запуск, повтор по времени или интервалу, ручной запуск. Источник: [Workflow Schedulers](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-schedulers.html).

---

## 10.1.2. Run Options

**Run Options** — как запускать workflow. Источник: [Workflow Scheduler Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-scheduler-properties.html).

| Опция | Описание |
|-------|----------|
| **Run On Demand** | Ручной запуск; Integration Service выполняет workflow при явном старте |
| **Run On Integration Service Initialization** | Запуск при инициализации Integration Service; следующий запуск — по Schedule Options |
| **Run Continuously** | Запуск при инициализации; следующий — сразу после завершения предыдущего |

**Run Continuously:** при редактировании workflow необходимо остановить или снять расписание, сохранить, затем перезапустить. Источник: [Workflow Scheduler Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-scheduler-properties.html).

---

## 10.1.3. Schedule Options

**Schedule Options** — тип расписания. Требуется при Run On Integration Service Initialization или при отсутствии выбора в Run Options. Источник: [Workflow Scheduler Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-scheduler-properties.html).

| Опция | Описание |
|-------|----------|
| **Run Once** | Один запуск в заданное время |
| **Run Every** | Запуск через заданные интервалы |
| **Customized Repeat** | Запуск в указанные даты и время; настройка Repeat Options |

---

## 10.1.4. Repeat Options (Customized Repeat)

При **Customized Repeat** настраивают детали повторения. Источник: [Repeat Options for Schedulers](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-scheduler-properties/repeat-options-for-schedulers.html).

**Repeat Every:** числовой интервал с частотой — Days, Weeks или Months.

**Weekly:** выбор дней недели для запуска.

**Monthly:**
- **Run On Day** — конкретные даты месяца (например, 1, 15, 31); при 31 в коротких месяцах — последний день.
- **Run On The** — неделя месяца (First, Second, Last и др.) и день недели (например, последняя среда месяца).

**Daily Frequency:**
- **Run Once** — один раз в выбранный день в заданное Start Time.
- **Run Every** — с интервалом (час, минута); первый запуск — по Start Time, далее — через интервал.

---

## 10.1.5. Start и End Options

**Start Options:** дата и время начала расписания (Start Date, Start Time). Источник: [Workflow Scheduler Properties](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-scheduler-properties.html).

**End Options** (для Run Every и Customized Repeat):
- **End On** — дата окончания расписания.
- **End After** — после заданного числа запусков.
- **Forever** — расписание действует, пока workflow не завершится с ошибкой.

---

## 10.1.6. Настройка расписания

Workflows → Edit → вкладка **Scheduler**. Источник: [Scheduling a Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/scheduling-a-workflow.html).

| Режим | Описание |
|-------|----------|
| **Non-reusable** | Отдельные настройки расписания для workflow |
| **Reusable** | Выбор существующего reusable scheduler; общие настройки для нескольких workflow в папке |

Правый клик по workflow в Navigator → **Schedule Workflow** — перезапланировать workflow на исходное расписание. Источник: [Scheduling a Workflow](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/scheduling-a-workflow.html).

---

## 10.1.7. Ограничения

При нескольких instance одного workflow и настроенном расписании Integration Service запускает **все** instance в запланированное время. Нельзя задать разное время для разных instance. Источник: [Workflow Schedulers](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/scheduling-and-running-workflows/workflow-schedulers.html).

---

## 10.1.8. Типичные ошибки

- **Ожидать cron-синтаксис:** PowerCenter использует GUI scheduler; нет прямого ввода cron-выражений.
- **Run Continuously без учёта длительности:** при долгом выполнении workflow следующий запуск откладывается; возможны накопления очереди.
- **Забыть Schedule Workflow после изменения:** после изменения расписания workflow нужно перезапланировать (Schedule Workflow).

---

## Ключевое

- **Workflow Scheduler** — настройки расписания; по умолчанию Run On Demand.
- **Run Options:** Run On Demand, Run On Integration Service Initialization, Run Continuously.
- **Schedule Options:** Run Once, Run Every, Customized Repeat.
- **Repeat Options:** Repeat Every (Days/Weeks/Months), Weekly, Monthly, Daily Frequency.
- **Start/End Options:** Start Date/Time; End On, End After, Forever.
- **Reusable scheduler** — общие настройки для нескольких workflow.

В [§10.2](chapter-10-02.md) мы разберём запуск и мониторинг: Workflow Monitor, статус задач, логи, просмотр выполненных записей.
