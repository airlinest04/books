# §7.1 Операторы (Operators)

В [главе 6](chapter-06-01.md) мы использовали BashOperator и PythonOperator для построения дага загрузки. Здесь операторы разбираются подробнее: **базовый класс BaseOperator**, два самых частых оператора — **BashOperator** и **PythonOperator**, **передача аргументов** в callable и в команду, и **шаблоны Jinja** для подстановки даты запуска и других переменных контекста. Сенсоры и хуки — в [§7.2](chapter-07-02.md) и [§7.3](chapter-07-03.md). См. [Глоссарий](glossary.md).

---

## 7.1.1. Базовый класс: BaseOperator

Все операторы в Airflow наследуются от **BaseOperator**. От него задачи получают общие параметры:

- **task_id** — уникальный идентификатор задачи в рамках DAG.
- **dag** — DAG, к которому принадлежит задача (явно или из контекста).
- **retries**, **retry_delay** — число повторных попыток и задержка между ними при сбое.
- **execution_timeout** — максимальное время выполнения задачи (по истечении задача прерывается).
- **owner** — владелец задачи (строка, для отображения в UI).
- **email**, **email_on_retry**, **email_on_failure** — уведомления по почте.
- **on_failure_callback**, **on_retry_callback**, **on_success_callback** — колбэки при смене статуса.

Конкретный оператор добавляет свои аргументы: у BashOperator — **bash_command**, у PythonOperator — **python_callable**, **op_args**, **op_kwargs** и т.д. При создании задачи вы передаёте и общие параметры (из default_args или явно), и специфичные для оператора.

---

## 7.1.2. BashOperator

**BashOperator** выполняет команду в shell. Основной параметр — **bash_command**: строка с командой (или список строк, которые объединяются). Команда выполняется в отдельном процессе; при ненулевом коде возврата задача считается проваленной.

```python
from airflow.operators.bash import BashOperator

task = BashOperator(
    task_id="run_script",
    bash_command="python /path/to/script.py --date {{ ds }}",
    dag=dag,
)
```

**Шаблонирование:** поле **bash_command** по умолчанию обрабатывается как **Jinja-шаблон**. В нём можно использовать переменные контекста run: **{{ ds }}** (логическая дата YYYY-MM-DD), **{{ data_interval_start }}**, **{{ data_interval_end }}**, **{{ dag }}**, **{{ ti }}** и др. Список переменных см. в документации Airflow (Template reference). Таким образом дата и другие параметры подставляются при выполнении задачи, а не при загрузке DAG.

Если шаблонирование не нужно, в части версий можно отключить его для этого поля (template_ext=None или аналог); по умолчанию bash_command шаблонируется.

---

## 7.1.3. PythonOperator

**PythonOperator** вызывает произвольную **Python-функцию** (callable). Основные параметры:

- **python_callable** — функция или иной вызываемый объект (без скобок: `my_func`, не `my_func()`).
- **op_args** — список позиционных аргументов, передаваемых в callable после контекста (если callable принимает **context).
- **op_kwargs** — словарь именованных аргументов, передаваемых в callable.

Callable может иметь сигнатуру без аргументов или с **\*\*context** (рекомендуется): Airflow передаёт контекст выполнения (dag, task, logical_date, ti, ds и т.д.). Возвращаемое значение callable при необходимости записывается в **XCom** и может быть прочитано следующими задачами ([§7.4](chapter-07-04.md)).

```python
from airflow.operators.python import PythonOperator

def my_task(**context):
    ds = context["ds"]
    ti = context["ti"]
    # логика задачи
    return "result_for_xcom"

task = PythonOperator(
    task_id="my_task",
    python_callable=my_task,
    op_kwargs={"extra": "value"},
    dag=dag,
)
```

**Шаблонирование:** у PythonOperator шаблонируются не все поля; типично шаблонируются **templates_dict** (словарь, значения которого обрабатываются как Jinja) и в части операторов — отдельные строковые поля. Контекст (в том числе дата) в callable лучше брать из аргумента **context**, а не из шаблонов.

---

## 7.1.4. Передача аргументов в callable (op_args, op_kwargs)

Чтобы передать в **python_callable** данные, не входящие в контекст Airflow, используйте **op_args** и **op_kwargs**:

```python
def process(date_str, prefix):
    path = f"{prefix}/data_{date_str}.csv"
    # ...

task = PythonOperator(
    task_id="process",
    python_callable=process,
    op_args=[ "{{ ds }}" ],           # позиционные: date_str
    op_kwargs={ "prefix": "/data" },  # именованные: prefix
    dag=dag,
)
```

Значения в **op_args** и **op_kwargs** могут быть шаблонами (например, `"{{ ds }}"`); они будут отрендерены при выполнении задачи. Callable получит уже подставленные значения. Важно: если callable объявлен как `def f(**context)`, то op_args и op_kwargs передаются в том же вызове; порядок аргументов в документации Airflow: сначала позиционные из op_args, затем контекст (если callable принимает *args и **context). Уточняйте сигнатуру по версии (например, callable может получать *args, **kwargs, где kwargs содержит и context, и op_kwargs).

---

## 7.1.5. Шаблоны Jinja для даты и контекста

В полях, поддерживающих **Jinja** (bash_command у BashOperator, templates_dict, часть полей других операторов), доступны переменные контекста. Часто используемые:

| Переменная | Описание |
|------------|----------|
| **{{ ds }}** | Логическая дата run в формате YYYY-MM-DD. |
| **{{ ds_nodash }}** | То же без дефисов: YYYYMMDD. |
| **{{ data_interval_start }}**, **{{ data_interval_end }}** | Начало и конец data interval (Airflow 2.2+). |
| **{{ ti }}** | TaskInstance (xcom_pull, и т.д.). |
| **{{ dag }}** | Объект DAG. |
| **{{ params }}** | Параметры run (если переданы при триггере). |

Пример в bash_command:

```python
BashOperator(
    task_id="export",
    bash_command="echo 'Date: {{ ds }}' && python export.py --date {{ ds }} --output /out/data_{{ ds_nodash }}.csv",
    dag=dag,
)
```

Использование даты из контекста гарантирует, что задача привязана к логической дате run, а не к текущему времени ([§5.3](chapter-05-03.md)).

---

## 7.1.6. Другие операторы (кратко)

Кроме BashOperator и PythonOperator в экосистеме Airflow есть множество операторов для конкретных систем: **PostgresOperator**, **HttpOperator**, **KubernetesPodOperator**, операторы провайдеров (Google, AWS, Azure) и т.д. Они инкапсулируют действие (выполнить SQL, отправить HTTP-запрос, запустить под в Kubernetes) и часто принимают **connection_id** для учётных данных. Подробности — в документации провайдеров; хуки для подключений — в [§7.3](chapter-07-03.md).

---

## Ключевое

- Все операторы наследуют **BaseOperator** и получают общие параметры: task_id, dag, retries, retry_delay, callbacks и др.
- **BashOperator** выполняет shell-команду; **bash_command** шаблонируется Jinja ({{ ds }}, {{ ti }} и др.).
- **PythonOperator** вызывает **python_callable**; аргументы передаются через **op_args** и **op_kwargs**; контекст — через **kwargs (context)**; возврат значения пишется в XCom.
- В шаблонах используйте **{{ ds }}** и другие переменные контекста для привязки к логической дате run.

В [§7.2](chapter-07-02.md) — сенсоры (Sensors): ожидание условия (файл, запись в БД), режимы poke и reschedule.
