# §6.4 Связывание шагов

В [§6.2](chapter-06-02.md) и [§6.3](chapter-06-03.md) мы описали таски извлечения, преобразования и загрузки. Теперь нужно **связать их в один DAG**: задать зависимости extract >> transform >> load и настроить **обработку сбоев** — повторные попытки (retries) и при необходимости уведомления (alert) при падении. В этом разделе — граф зависимостей для пайплайна загрузки и параметры retry и оповещений. Проверка выполнения в UI — в [§6.5](chapter-06-05.md).

---

## 6.4.1. Зависимости: extract >> transform >> load

Порядок выполнения задаётся оператором `>>` ([§4.4](chapter-04-04.md)): сначала extract, после его успешного завершения — transform, после transform — load. В коде DAG после объявления всех операторов добавьте одну строку:

```python
extract >> transform >> load
```

Если transform и load объединены в один таск (transform_and_load), цепочка будет:

```python
extract >> transform_and_load
```

Планировщик не поставит transform в очередь, пока extract не завершится успешно; load не запустится, пока не завершится transform. При падении extract задачи transform и load для этого run не выполнятся; при падении transform не выполнится load. Это соответствует логике пайплайна: нет смысла преобразовывать или загружать то, что не извлечено.

---

## 6.4.2. Полный фрагмент DAG (каркас)

Сводный каркас DAG с тремя шагами и зависимостями:

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

with DAG(
    dag_id="sales_load",
    schedule_interval="0 2 * * *",
    start_date=datetime(2025, 1, 1),
    catchup=False,
    default_args={
        "retries": 2,
        "retry_delay": timedelta(minutes=5),
    },
) as dag:
    extract = BashOperator(
        task_id="extract",
        bash_command="python /scripts/extract_sales.py --date {{ ds }}",
    )
    transform = PythonOperator(
        task_id="transform",
        python_callable=transform_sales,
    )
    load = PythonOperator(
        task_id="load",
        python_callable=load_to_raw,
    )

    extract >> transform >> load
```

Функции `transform_sales` и `load_to_raw` должны быть определены (или импортированы) в том же файле; при необходимости передать соединения или пути через Variables/Connections ([§7.5](chapter-07-05.md)).

---

## 6.4.3. Обработка сбоев: retries

При временных сбоях (сеть, недоступность API или БД) задача может упасть. Чтобы не требовать ручного перезапуска, задайте **retries** и **retry_delay** ([§2.3](chapter-02-03.md)). Их можно указать в **default_args** DAG — тогда они применяются ко всем задачам — или у отдельного оператора.

В примере выше в `default_args` заданы `retries=2` и `retry_delay=timedelta(minutes=5)`: после падения задача будет повторена до 2 раз с паузой 5 минут. Для отдельных задач можно переопределить, например, увеличить число попыток для extract (часто зависящего от внешних систем):

```python
extract = BashOperator(
    task_id="extract",
    bash_command="...",
    retries=3,
    retry_delay=timedelta(minutes=10),
)
```

Убедитесь, что задачи **идемпотентны** ([§6.1](chapter-06-01.md)): повторный запуск за ту же дату не должен портить данные.

---

## 6.4.4. Уведомления при сбое (alert)

После исчерпания retry задача переходит в статус **failed**. Чтобы ответственный узнал об этом, настройте **оповещения**. В Airflow это делается через callback-функции (on_failure_callback, on_retry_callback) или через интеграции (например, отправка в Slack, email, PagerDuty). В callback передаётся контекст (dag_id, task_id, logical_date, ссылка на логи); внутри callback можно отправить сообщение в мессенджер или в систему мониторинга.

Пример идеи (конкретный API зависит от вашего канала):

```python
def on_failure_alert(context):
    # отправить в Slack/email: context["dag_id"], context["task_id"], context["logical_date"]
    ...

extract = BashOperator(
    task_id="extract",
    bash_command="...",
    on_failure_callback=on_failure_alert,
)
```

Можно задать **on_failure_callback** в default_args, чтобы алерт срабатывал при падении любой задачи DAG. Детали настройки каналов (webhook, email) см. в документации Airflow и провайдеров.

---

## 6.4.5. Типичные ошибки

- **Забыть задать зависимость** — задача окажется «висячей» или выполнится не в том порядке; всегда явно указывайте extract >> transform >> load (или свой граф).
- **Retry без идемпотентности** — повторный запуск load за ту же дату может продублировать данные; сначала обеспечьте перезапись среза или upsert.
- **Слишком частые алерты** — при большом retry алерт лучше отправлять после окончательного падения (on_failure_callback), а не на каждую попытку retry, чтобы не засыпать уведомлениями.

---

## Ключевое

- Зависимости задаются одной строкой: **extract >> transform >> load** (или extract >> transform_and_load при объединённом шаге).
- **retries** и **retry_delay** задают повтор при сбое; удобно вынести в default_args. Задачи должны быть идемпотентны.
- **on_failure_callback** (и при необходимости on_retry_callback) позволяет отправить уведомление при падении задачи; настраивается в операторе или в default_args.

В [§6.5](chapter-06-05.md) — проверка в UI: успешный прогон, просмотр логов и повтор при ошибке.
