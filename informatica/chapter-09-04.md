# §9.4 Зависимости и связи между задачами

В [§9.3](chapter-09-03.md) мы рассмотрели Worklets и параметры. **Связи (links)** между задачами определяют порядок выполнения workflow и worklet. В этом разделе — Link Task, последовательное и параллельное связывание, условные связи (link conditions), ограничение на циклы. Подробнее расписание — в [§10.1](chapter-10-01.md). См. [Глоссарий](glossary.md).

---

## 9.4.1. Workflow Links: назначение

**Links** (связи) соединяют задачи в workflow или worklet. Integration Service выполняет следующую задачу только после завершения предыдущей и при выполнении условия связи (если задано). Источник: [Workflow Links](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links.html).

**Ограничение:** Workflow Manager не допускает циклов (loops). Каждая связь выполняется только один раз. Источник: [Workflow Links](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links.html).

Без условия связи Integration Service по умолчанию выполняет следующую задачу после завершения предыдущей. Источник: [Workflow Links](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links.html).

---

## 9.4.2. Создание связей

**Link Tasks (ручное связывание):** Tasks → Link Tasks; щёлкнуть первую задачу и перетащить на вторую. Появляется связь между задачами. Источник: [Linking Two Tasks](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/linking-two-tasks.html).

**Link Sequential (последовательно):** выбрать первую задачу, Ctrl+щелчок по следующим в нужном порядке; Tasks → Link Sequential. Задачи связываются в цепочку: Task1 → Task2 → Task3 → … Источник: [Linking Tasks Sequentially](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/linking-tasks-sequentially.html).

**Link Concurrent (параллельно):** выбрать одну задачу, Ctrl+щелчок по остальным; Tasks → Link Concurrent. От выбранной задачи создаются связи ко всем остальным; они выполняются параллельно. Источник: [Linking Tasks Concurrently](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/linking-tasks-concurrently.html).

---

## 9.4.3. Последовательное выполнение

При последовательном связывании каждая следующая задача запускается только после успешного завершения предыдущей. Источник: [Linking Tasks Sequentially](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/linking-tasks-sequentially.html).

**Пример:** Start → Session1 → Session2 → Session3. Session2 не начнётся, пока Session1 не завершится; Session3 — пока Session2 не завершится.

---

## 9.4.4. Параллельное выполнение

При параллельном связывании одна задача связывается с несколькими; Integration Service запускает их параллельно (при наличии ресурсов). Источник: [Linking Tasks Concurrently](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/linking-tasks-concurrently.html).

**Пример:** Start → Session1, Session2, Session3. Session1, Session2 и Session3 выполняются одновременно после Start.

---

## 9.4.5. Link Conditions (условные связи)

**Link condition** — выражение, определяющее, выполнять ли следующую задачу. Если условие True — Integration Service выполняет задачу; если False — не выполняет. Источник: [Workflow Links](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links.html).

В условии используются предопределённые или пользовательские переменные workflow/worklet. Expression Editor предоставляет переменные, функции и операторы. Источник: [Creating Link Conditions](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/creating-link-conditions.html).

**Пример:** запускать Session2 только если в Session1 нет failed rows у target:

```text
$s_STORES_CA.TgtFailedRows = 0
```

Источник: [Example of Link Conditions](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links/creating-link-conditions/example-of-link-conditions.html).

Условия создают **ветвление**: из одной задачи — несколько связей с разными условиями; выполнится только та ветка, условие которой True. Результаты оценки условий записываются в workflow log. Источник: [Workflow Links](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflows-and-worklets/workflow-links.html).

---

## 9.4.6. Decision task и ветвление

**Decision** — задача условного ветвления. Из Decision выходят несколько связей с разными условиями; Integration Service оценивает условия и выполняет только те ветки, где условие True. Используется вместе с link conditions для сложной логики. См. [§9.1](chapter-09-01.md).

---

## 9.4.7. Типичные ошибки

- **Циклы в связях:** PowerCenter не допускает циклов; граф выполнения должен быть ациклическим.
- **Start без связи:** хотя бы одна задача должна быть связана с Start; иначе workflow не выполнит ни одной задачи.
- **Задача без входящей связи:** задача без входящей связи (кроме Start) не будет выполнена.
- **Неверное условие связи:** при ошибке в выражении Integration Service может не выполнить задачу или завершить workflow с ошибкой; проверять через Validate в Expression Editor.

---

## Ключевое

- **Links** — связи между задачами; определяют порядок выполнения; циклы запрещены.
- **Link Sequential** — последовательная цепочка; **Link Concurrent** — параллельные ветки от одной задачи.
- **Link condition** — выражение True/False; при True следующая задача выполняется; при False — нет.
- **Ветвление** — несколько связей с разными условиями; Decision task для условной логики.
- **Workflow log** — результаты оценки условий записываются в лог.

В [§10.1](chapter-10-01.md) мы перейдём к расписанию и запуску: Schedule, настройка времени выполнения, рекуррентность.
