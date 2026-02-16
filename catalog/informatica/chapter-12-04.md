# §12.4 Workflow и проверка

В [§12.3](chapter-12-03.md) мы настроили загрузку в Target. **Workflow** оркестрирует выполнение Session; **проверка** — запуск, мониторинг в Workflow Monitor и сверка данных. В этом разделе — создание Session и Workflow для пайплайна загрузки, настройка расписания, запуск и проверка в Monitor, сверка данных (row count, регрессия). Завершаем главу 12 практикой полного цикла ETL. См. [Глоссарий](glossary.md).

---

## 12.4.1. Session: связь с маппингом

**Session** — задача Workflow, выполняющая один Mapping. Создаётся в Workflow Manager: Workflows → Create Session (или Session → Create) → выбор Mapping. Session настраивает: $Source/$Target connections, parameter file, Target Load Type, Treat source rows as и др. См. [§9.2](chapter-09-02.md).

**Для пайплайна загрузки в DWH:**
- Mapping tab: Connections → $Source, $Target (или session parameters для разных окружений).
- Mapping tab: Target properties → Insert, Update, Truncate, Bulk/Normal.
- Properties tab: Treat source rows as = Data Driven (при Update Strategy).
- Config Object: parameter file, Error Handling, при необходимости.

---

## 12.4.2. Workflow: структура

**Workflow** — Start → Session (и при необходимости Command, Email, Decision). См. [§9.1](chapter-09-01.md).

**Типовая структура для загрузки в Staging:**
```
Start → Session_LoadStaging
```

**Двухшаговый пайплайн (Staging → DWH):**
```
Start → Session_LoadStaging → Session_LoadDWH
```

**С pre-session Command (TRUNCATE):** Command task перед Session или pre-session command в Session. **С уведомлением при сбое:** Session → link → Email (On-Failure) или post-session Failure Email. См. [§9.4](chapter-09-04.md), [§10.4](chapter-10-04.md).

---

## 12.4.3. Schedule

**Workflow Scheduler** — когда запускать. Для ежедневной загрузки: Run Options → Run On Demand или Run Every; Schedule Options → Customized Repeat; Repeat Every 1 Day; Start/End Options. См. [§10.1](chapter-10-01.md).

**Parameter file:** при инкрементальной загрузке `$$LastRunDate` задаётся в parameter file; значение обновляется между запусками (через post-session command или внешний скрипт).

---

## 12.4.4. Запуск и проверка в Monitor

**Запуск:** Workflow Manager или Workflow Monitor → правый клик по workflow → Start Workflow. Или pmcmd: `pmcmd startworkflow -uv USER -pv PASS -s HOST:PORT -f FOLDER -w WORKFLOW workflow_name`. См. [§10.2](chapter-10-02.md).

**Workflow Monitor:**
- **Gantt Chart** — хронология выполнения; статус задач.
- **Task view** — отчёт по workflow run; фильтрация по статусу.
- **Properties** — детали: время, Source rows read, Target rows loaded, Rejected rows.
- **Session Log** — ошибки, performance counters.

**Проверка успешности:** статус workflow и Session = Succeeded; Rejected rows = 0 (или в пределах ожидаемых).

---

## 12.4.5. Сверка данных

**Сверка данных** — проверка корректности загрузки: объём и целостность.

| Метод | Описание |
|-------|----------|
| **Row count** | Сравнение `SELECT COUNT(*)` по source и target; при полной загрузке — совпадение (с учётом Filter). |
| **Session Log** | Source rows read, Target rows loaded, Rejected rows; при Rejected > 0 — анализ reject file. |
| **Выборочная проверка** | Сравнение ключевых полей (SUM, AVG) по source и target. |
| **Регрессия** | При повторных запусках — стабильность row count; при инкрементальной — рост target. |

**Идемпотентность:** повторный запуск с теми же данными — тот же row count; без дубликатов. См. [§12.1.3](chapter-12-01.md).

---

## 12.4.6. Чек-лист перед production

- [ ] **Connections:** $Source, $Target указывают на корректные БД (или session parameters для parameter file).
- [ ] **Parameter file:** при инкрементальной загрузке — `$$LastRunDate` и др.; путь доступен Integration Service.
- [ ] **Treat source rows as:** Data Driven при Update Strategy.
- [ ] **Error Handling:** Stop On Errors, Failure Email для критичных сессий.
- [ ] **Schedule:** время запуска, учёт зависимостей (источник доступен).
- [ ] **Сверка:** тестовый прогон; row count, Rejected rows; при необходимости — выборочная проверка.

---

## Ключевое

- **Session** — связь с Mapping; Connections, Target properties, Treat source rows as.
- **Workflow** — Start → Session; при необходимости Command, Email, несколько Session.
- **Schedule** — Run On Demand, Run Every или Customized Repeat; parameter file для инкрементальной загрузки.
- **Monitor** — Gantt Chart, Task view, Properties, Session Log; статус Succeeded, Rejected rows.
- **Сверка** — row count, Session Log метрики, выборочная проверка; идемпотентность при повторном запуске.

Глава 12 завершена. В [§13.1](chapter-13-01.md) мы перейдём к обзору Informatica Cloud (IICS) и расширению возможностей PowerCenter.
