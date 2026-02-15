# §4.3 Task (задача)

В [§4.2](chapter-04-02.md) мы разобрали параметры DAG. Вершины графа — это **задачи (tasks)**. В этом разделе даётся определение задачи в Airflow: задача — это **экземпляр оператора (Operator)**; она **принадлежит DAG** и имеет **уникальный task_id** в рамках этого DAG. Показаны минимальные примеры задач с BashOperator и PythonOperator; зависимости между задачами — в [§4.4](chapter-04-04.md); подробнее об операторах — в [главе 7](chapter-07-01.md).

---

## 4.3.1. Определение: задача как экземпляр оператора

**Task (задача)** в Airflow — это единица работы, которую планировщик ставит в очередь и которую выполняет Executor (или воркер). В коде задача представлена **экземпляром класса-оператора** (Operator): например, `BashOperator`, `PythonOperator`, `PostgresOperator`. При создании оператора вы передаёте как минимум:

- **task_id** — строковый идентификатор задачи, уникальный в рамках одного DAG.
- Параметры, специфичные для оператора: команда для выполнения, вызываемая функция, запрос к БД и т.д.
- **dag** — объект DAG, к которому принадлежит задача (явно или через контекст `with DAG(...)`).

Один и тот же оператор можно использовать несколько раз в одном DAG с разными `task_id` и параметрами. Каждый экземпляр — отдельная вершина графа. См. [Глоссарий](glossary.md).

---

## 4.3.2. Задача принадлежит DAG

Каждая задача должна быть привязана ровно к одному DAG. Это делается одним из способов:

- **Через контекстный менеджер:** задачи создаются внутри `with DAG(...):` и передают `dag` неявно (если оператор принимает `dag` из контекста) или явно, если вы получаете объект через `with DAG(...) as dag:` и передаёте `dag=dag`.
- **Явная передача:** при создании оператора указать параметр `dag=my_dag`.

Без привязки к DAG задача не попадёт в граф и не будет планироваться. В типичном коде все задачи объявляются внутри одного блока `with DAG(...):`, и DAG подставляется автоматически.

---

## 4.3.3. Уникальный task_id в рамках DAG

**task_id** — обязательный строковый аргумент оператора. В рамках **одного DAG** все задачи должны иметь разные `task_id`. Повторение `task_id` в одном DAG приводит к ошибке или к тому, что одна задача перезапишет другую в графе. В разных DAG могут быть задачи с одинаковым `task_id` — конфликта нет, так как задача идентифицируется парой (dag_id, task_id). Рекомендуется осмысленные имена: `extract_sales`, `load_staging`, `notify` и т.п.

---

## 4.3.4. Пример: BashOperator

**BashOperator** выполняет bash-команду (или команду в shell). Минимальный пример:

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id="example_with_tasks",
    schedule_interval=None,
    start_date=datetime(2025, 1, 1),
) as dag:
    hello = BashOperator(
        task_id="say_hello",
        bash_command="echo 'Hello from Airflow'",
    )
```

Здесь задача `say_hello` принадлежит `dag` (благодаря контексту `with DAG(...) as dag` и тому, что оператор по умолчанию подхватывает DAG из контекста). При выполнении Airflow запустит команду `echo 'Hello from Airflow'`. Среда: Airflow 2.x.

---

## 4.3.5. Пример: PythonOperator

**PythonOperator** вызывает произвольную Python-функцию. Функция должна быть вызываемой без аргументов или с аргументами, передаваемыми через `op_kwargs` / `op_args`. Пример:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def print_date(**context):
    logical_date = context.get("ds")  # дата запуска в формате YYYY-MM-DD
    print(f"Logical date: {logical_date}")

with DAG(
    dag_id="example_python_task",
    schedule_interval=None,
    start_date=datetime(2025, 1, 1),
) as dag:
    run_python = PythonOperator(
        task_id="print_logical_date",
        python_callable=print_date,
    )
```

Задача `print_logical_date` при выполнении вызовет функцию `print_date`; Airflow передаёт в callable контекст (в том числе `ds` — логическая дата). Подробнее о контексте и шаблонах — в [§7.1](chapter-07-01.md).

---

## 4.3.6. default_args для задач DAG

Общие параметры для всех задач DAG можно задать в **default_args** — словаре, передаваемом в конструктор `DAG()`. Типичные ключи: `retries`, `retry_delay`, `owner`, `email_on_failure`. Эти значения подставляются в каждую задачу DAG, если в самой задаче параметр не переопределён. Пример:

```python
with DAG(
    dag_id="example_default_args",
    schedule_interval=None,
    start_date=datetime(2025, 1, 1),
    default_args={
        "retries": 2,
        "retry_delay": timedelta(minutes=5),
        "owner": "data-team",
    },
) as dag:
    t1 = BashOperator(task_id="task_one", bash_command="echo 1")
    # t1 унаследует retries=2, retry_delay, owner из default_args
```

---

## 4.3.7. Типичные ошибки

- **Два одинаковых task_id в одном DAG** — граф будет некорректен или одна задача затрёт другую; задайте уникальные идентификаторы.
- **Задача не привязана к DAG** — при создании оператора вне блока `with DAG(...)` нужно передать `dag=...`; иначе задача не появится в пайплайне.
- **Импорт оператора** — убедитесь, что импортируете нужный класс (например, `from airflow.operators.bash import BashOperator`); при ошибке импорта файл не загрузится (см. [§3.4](chapter-03-04.md)).

---

## Ключевое

- **Task (задача)** — экземпляр оператора (BashOperator, PythonOperator и др.) с уникальным **task_id** в рамках DAG и привязкой к одному **DAG**.
- Задача создаётся внутри `with DAG(...):` или с явной передачей `dag=...`; без DAG задача не планируется.
- **default_args** в DAG задаёт общие параметры (retries, retry_delay, owner и т.д.) для всех задач этого DAG.
- Операторы различаются по действию (команда shell, вызов Python, запрос к БД); подробнее — в [главе 7](chapter-07-01.md).

В [§4.4](chapter-04-04.md) разберём зависимости между тасками: оператор `>>`, цепочки и ветвления, set_downstream/set_upstream.
