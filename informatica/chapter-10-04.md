# §10.4 Обработка сбоев и retry

В [§10.3](chapter-10-03.md) мы рассмотрели параметры и переопределение. **Обработка сбоев** — настройки, определяющие поведение при ошибках сессии: порог ошибок, стратегии восстановления (recovery), уведомления по email. В этом разделе — Error Handling Settings (Stop On Errors, порог), стратегии recovery (Resume, Restart, Fail), post-session email, Suspend on Error и best practices. Подробнее производительность — в [§11.1](chapter-11-01.md). См. [Глоссарий](glossary.md).

---

## 10.4.1. Error Handling Settings

**Error Handling Settings** — настройки в Session Config Object (Config Object tab → Error Handling), определяющие, при каком количестве ошибок сессия останавливается и как обрабатываются ошибки pre-session/post-session команд и SQL. Источник: [Error Handling Settings](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/session-configuration-object/error-handling-settings.html).

| Настройка | Описание |
|-----------|----------|
| **Stop On Errors** | Количество non-fatal ошибок (reader, writer, transformation), после которых Integration Service останавливает сессию. 0 — ошибки не останавливают сессию. Можно использовать `$PMSessionErrorThreshold`. |
| **On Stored Procedure Error** | Stop Session или Continue Session при ошибке pre/post-session stored procedure. По умолчанию — Stop Session. |
| **On Pre-Session Command Task Error** | Stop Session или Continue Session при ошибке pre-session shell command. По умолчанию — Stop Session. |
| **On Pre-Post SQL Error** | Stop Session или Continue Session при ошибке pre/post-session SQL. По умолчанию — Stop Session. |
| **Error Log Type** | Relational, file или none — куда писать row-level ошибки. |
| **Log Row Data** | Логировать ли данные строк при ошибке. |

Integration Service ведёт отдельный счётчик ошибок для каждого source, target и transformation. При партиционировании — отдельный порог для каждой партиции. Источник: [Threshold Errors](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/stopping-and-aborting/types-of-errors/threshold-errors.html).

---

## 10.4.2. Стратегии recovery (retry)

PowerCenter не имеет настройки «retry N раз» на уровне сессии. Вместо этого используются **стратегии recovery** при восстановлении workflow после сбоя (Stop, Abort, Terminate). Настраиваются в Session → Properties → Recovery. Источник: [Session Task Strategies](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/workflow-recovery/configuring-task-recovery/task-recovery-strategies/session-task-strategies.html).

| Стратегия | Поведение |
|-----------|-----------|
| **Resume from the last checkpoint** | Integration Service сохраняет состояние сессии и target recovery tables; при Abort/Stop/Terminate возобновляет с точки прерывания. Нельзя использовать с mapping variables. |
| **Restart task** | При recovery workflow сессия запускается заново. Может потребоваться удаление частично загруженных данных или маппинг, пропускающий дубликаты. |
| **Fail task and continue workflow** | Сессия не восстанавливается; статус Failed; workflow продолжает выполнение остальных задач. |

Стратегия **Restart** — ближайший аналог retry: при recovery workflow сессия выполняется повторно. Recovery выполняется вручную (Workflow Monitor → Recover) или автоматически (при включённой опции). Источник: [Session Task Strategies](https://docs.informatica.com/data-integration/powercenter/10-5/advanced-workflow-guide/workflow-recovery/configuring-task-recovery/task-recovery-strategies/session-task-strategies.html).

---

## 10.4.3. Post-session email

**Post-session email** — уведомление по email при успешном или неуспешном завершении сессии. Настраивается в Session → Properties → General Options → Success Email / Failure Email. Источник: [Working with Post-Session Email](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sending-email/working-with-post-session-email.html).

Можно указать reusable или non-reusable Email task. Integration Service отправляет email после выполнения post-session commands и stored procedures. При ошибке отправки email сессия не падает; сообщение пишется в Log Service. Источник: [Working with Post-Session Email](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sending-email/working-with-post-session-email.html).

**Переменные для email:** `%s` — имя сессии, `%e` — статус, `%l` — загруженные строки, `%r` — rejected rows, `%g` — прикрепить session log, `%t` — детали source/target. Источник: [Email Variables and Format Tags](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/sending-email/working-with-post-session-email/email-variables-and-format-tags.html).

---

## 10.4.4. Suspend on Error

При включённой опции **Suspend workflow on error** (свойство задачи) Integration Service при сбое задачи переводит workflow в статус **Suspended** вместо Failed. Это позволяет выполнить recovery: исправить причину сбоя и возобновить workflow с точки прерывания. См. [§10.2.3](chapter-10-02.md).

Опция настраивается в свойствах задачи (Session, Command и др.) в Workflow Designer. Источник: [Workflow and Task Status](https://docs.informatica.com/data-integration/powercenter/10-5/workflow-basics-guide/workflow-monitor/workflow-and-task-status.html).

---

## 10.4.5. Поведение при разных типах ошибок

Integration Service обрабатывает сбои по-разному в зависимости от причины. Источник: [Integration Service Handling for Session Failure](https://docs.informatica.com/data-integration/powercenter/10-4-1/advanced-workflow-guide/stopping-and-aborting/integration-service-handling-for-session-failure.html).

| Причина | Поведение |
|---------|-----------|
| **Error threshold (reader)** | Останавливает чтение; продолжает обработку и запись; коммитит данные. |
| **Error threshold (writer)** | Останавливает чтение и запись; откатывает незакоммиченные данные. |
| **Fatal error (БД)** | Останавливает; откат; commit/rollback может не завершиться. |
| **Stop command** | Останавливает задачу и цепочку; параллельные задачи продолжают выполняться. |
| **Abort command** | Завершает DTM process; при необходимости — rollback. |

---

## 10.4.6. Best practices

- **Stop On Errors:** задать разумный порог (например, 1000) для non-fatal ошибок; 0 — только если допустимы любые ошибки.
- **Error Log:** включить Error Log (relational или file) для анализа rejected rows; Log Row Data — при необходимости отладки.
- **Post-session email:** настроить Failure Email для критичных сессий; использовать `%g` для прикрепления session log.
- **Recovery strategy:** Resume — для длинных сессий с checkpoint; Restart — при идемпотентной загрузке; Fail — если сессия не критична и workflow должен продолжиться.
- **Suspend on Error:** для production workflow, требующих ручного вмешательства при сбое, — включить Suspend вместо немедленного Failed.

---

## 10.4.7. Типичные ошибки

- **Stop On Errors = 0 и множество rejected rows:** сессия завершится «успешно», но данные могут быть неполными; мониторить rejected rows.
- **Resume с mapping variables:** стратегия Resume не поддерживает mapping variables; использовать Restart.
- **Restart без идемпотентности:** при Restart возможны дубликаты; маппинг должен обрабатывать повторную загрузку (например, MERGE, проверка ключа).
- **Крупные вложения в email:** `%g` с verbose tracing создаёт большие файлы; учитывать лимиты почтового сервера.

---

## Ключевое

- **Error Handling Settings:** Stop On Errors (порог), On Stored Procedure/Pre-Session Command/Pre-Post SQL Error; Error Log Type.
- **Recovery strategies:** Resume (с checkpoint), Restart (повторный запуск), Fail (продолжить workflow).
- **Post-session email:** Success Email и Failure Email; переменные %s, %e, %l, %r, %g.
- **Suspend on Error** — приостановка workflow для ручного recovery вместо Failed.
- **Best practices:** порог ошибок, error log, Failure Email, выбор recovery strategy по сценарию.

В [§11.1](chapter-11-01.md) мы разберём партиционирование сессий для повышения производительности.
