# §7.3 Хуки (Hooks)

В [§7.2](chapter-07-02.md) мы разобрали сенсоры. Для подключения к внешним системам (БД, HTTP API, облачные хранилища) в Airflow используются **хуки (Hooks)** — абстракция, которая хранит параметры подключения (обычно через Airflow **Connections**) и предоставляет методы для работы с системой. В этом разделе — что такое Hook, как использовать его в **PythonOperator**, и краткие примеры **PostgresHook** и **HttpHook**. Variables и Connections (где хранятся учётные данные) — в [§7.5](chapter-07-05.md). См. [Глоссарий](glossary.md).

---

## 7.3.1. Назначение хуков

**Hook** инкапсулирует логику подключения к внешней системе: не нужно хардкодить хост, порт, логин и пароль в коде DAG — они задаются в **Connection** в UI или в конфиге, а в коде указывается только **conn_id**. Hook при первом обращении получает параметры из Connection, устанавливает соединение (или возвращает клиент) и предоставляет методы: выполнить запрос, прочитать/записать данные, закрыть соединение. Так секреты не попадают в репозиторий, а смена хоста или учётных данных делается в одном месте (Connection). Операторы и сенсоры для БД/API внутри часто используют те же хуки.

---

## 7.3.2. Использование в PythonOperator

Хук создаётся внутри callable задачи (или в методе poke сенсора): получаете экземпляр Hook по **conn_id**, вызываете методы — чтение, запись, выполнение запроса. Соединение управляется хуком (контекстный менеджер get_conn или аналог); после выхода из контекста соединение закрывается. Пример общей схемы:

```python
def task_with_hook(**context):
    hook = SomeHook(conn_id="my_connection")
    conn = hook.get_conn()
    # работа через conn или методы hook
    # ...
```

Конкретный API зависит от типа хука. Не создавайте Hook на уровне модуля DAG (при загрузке файла): подключение должно устанавливаться **в момент выполнения задачи**, когда уже известен контекст run и доступны Connections.

---

## 7.3.3. PostgresHook (кратко)

**PostgresHook** — подключение к PostgreSQL. Требует провайдер `apache-airflow-providers-postgres` и Connection типа Postgres с указанием host, schema (база), login, password, port. В коде:

```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

def read_from_pg(**context):
    hook = PostgresHook(postgres_conn_id="source_db")
    conn = hook.get_conn()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM raw.sales WHERE date = %s", (context["ds"],))
    rows = cursor.fetchall()
    # обработка или запись в файл/другую БД
    conn.close()
    return len(rows)
```

Удобные методы: **get_records(sql)** — выполнить запрос и вернуть список строк; **run(sql)** — выполнить SQL без возврата результата; **get_pandas_df(sql)** — вернуть результат как DataFrame (при установленном pandas). Для записи больших объёмов часто используют **copy_expert** или вставку батчами. Подробности — в документации провайдера.

---

## 7.3.4. HttpHook (кратко)

**HttpHook** — выполнение HTTP-запросов (REST API). Connection хранит хост (и при необходимости схему, логин/пароль для базовой авторизации). В коде получаете клиент и делаете get/post:

```python
from airflow.providers.http.hooks.http import HttpHook

def call_api(**context):
    hook = HttpHook(method="GET", http_conn_id="my_api")
    response = hook.run(endpoint="/data", data={"date": context["ds"]})
    # разбор response
    return response.json()
```

Параметры **run**: endpoint, data, headers и т.д. — по документации провайдера. Для сложной авторизации (OAuth, токены) параметры часто задают в Connection (extra) или передают через Variables ([§7.5](chapter-07-05.md)).

---

## 7.3.5. Другие хуки (кратко)

В провайдерах Airflow есть хуки для многих систем: **MySqlHook**, **SnowflakeHook**, **S3Hook**, **GCSHook**, **SlackHook** и др. Общая идея та же: **conn_id** указывает на Connection, Hook даёт методы для работы с системой. Использование в PythonOperator — создание хука внутри callable и вызов методов. Документация по каждому провайдеру описывает параметры Connection и API хука.

---

## Ключевое

- **Hook** — абстракция подключения к внешней системе; параметры берутся из **Connection** (conn_id), секреты не хранятся в коде.
- Хук создаётся **внутри задачи** (callable в PythonOperator или в сенсоре), а не при загрузке DAG.
- **PostgresHook** — подключение к PostgreSQL, get_conn(), get_records(), run(), get_pandas_df(); Connection типа Postgres.
- **HttpHook** — HTTP-запросы к API; Connection с хостом, в run() передаётся endpoint и параметры.
- Остальные хуки (S3, GCS, Slack и т.д.) — по документации провайдеров; использование через conn_id и методы хука.

В [§7.4](chapter-07-04.md) — передача данных между тасками: XCom (push и pull), ограничения по размеру, когда передавать путь к файлу или ключ в БД.
