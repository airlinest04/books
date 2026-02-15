# §10.1 Подключение к БД

В [§9.6](chapter-09-06.md) мы разобрали сортировку и ранжирование. Данные часто хранятся в **реляционных базах данных** (PostgreSQL, SQLite, MySQL и др.). Чтобы читать их в Pandas, нужен **драйвер** для конкретной СУБД и удобный способ выполнять запросы и получать результат таблицей. В этом разделе — установка **SQLAlchemy** и драйвера **psycopg2** для PostgreSQL, **создание движка** (engine) подключения и базовое **чтение** результата запроса в DataFrame через **pd.read_sql()**. Подробнее о запросах, параметризации и записи в таблицу — в [§10.2](chapter-10-02.md).

---

## 10.1.1. SQLAlchemy и драйверы

**SQLAlchemy** — библиотека Python для работы с БД: единый API для подключения, выполнения SQL и работы с результатами. Она не подключается к БД напрямую, а использует **драйвер** (DBAPI-драйвер) для конкретной СУБД. Для **PostgreSQL** обычно берут **psycopg2** (или **psycopg2-binary** — сборка без отдельной установки libpq). Для SQLite драйвер встроен в Python; для MySQL — например, PyMySQL или mysqlclient.

В контексте Pandas чаще всего используют **движок** SQLAlchemy (create_engine) как «подключение» для **pd.read_sql()** и **df.to_sql()**: запрос передаётся в БД, результат возвращается в виде DataFrame. Сам SQL пишется в виде строки (SELECT …); параметризация и запись в таблицу рассмотрены в [§10.2](chapter-10-02.md).

---

## 10.1.2. Установка

Установить SQLAlchemy и драйвер для PostgreSQL можно через pip:

```bash
pip install sqlalchemy psycopg2-binary
```

**psycopg2-binary** — предсобранный пакет; для продакшена иногда предпочитают **psycopg2** (требует установки библиотеки libpq в системе). Для SQLite достаточно **sqlalchemy** (драйвер sqlite3 входит в стандартную библиотеку Python).

---

## 10.1.3. Создание движка (create_engine)

**create_engine(url, ...)** из SQLAlchemy создаёт объект **Engine** — фабрику подключений к БД. Он не открывает соединение сразу; соединение создаётся при первом запросе (например, при вызове read_sql). Один движок можно использовать многократно для многих запросов.

**URL подключения** задаёт тип БД, параметры (хост, порт, имя БД, пользователь, пароль). Формат: **dialect+driver://user:password@host:port/database**. Для PostgreSQL с psycopg2:

```text
postgresql+psycopg2://user:password@host:port/dbname
```

Пример: **postgresql+psycopg2://scott:tiger@localhost:5432/mydb**. Для SQLite URL — путь к файлу: **sqlite:///path/to/file.db** (три слэша для абсолютного или относительного пути).

Пароль и другие секреты не следует хранить в коде; обычно их берут из переменных окружения или конфигурации.

```python
from sqlalchemy import create_engine
import os

# Пример: параметры из переменных окружения
user = os.environ.get("PGUSER", "user")
password = os.environ.get("PGPASSWORD", "password")
host = os.environ.get("PGHOST", "localhost")
port = os.environ.get("PGPORT", "5432")
dbname = os.environ.get("PGDATABASE", "mydb")

url = f"postgresql+psycopg2://{user}:{password}@{host}:{port}/{dbname}"
engine = create_engine(url)
```

Для SQLite (без сети, один файл):

```python
engine = create_engine("sqlite:///data.db")
```

---

## 10.1.4. Чтение в DataFrame: read_sql()

**pd.read_sql(sql, con, index_col=None, params=None, ...)** выполняет SQL-запрос **sql** через подключение **con** (обычно объект **engine** или соединение SQLAlchemy) и возвращает **DataFrame**: столбцы — колонки результата запроса, строки — строки результата.

- **sql** — строка с SQL (как правило, SELECT). В [§10.2](chapter-10-02.md) — параметризованные запросы и запись в таблицу (to_sql).
- **con** — движок **engine** или объект соединения (connection). Рекомендуется передавать engine: **pd.read_sql("SELECT * FROM t LIMIT 10", engine)**.
- **index_col** — имя столбца (или список), который использовать как индекс строк результирующего DataFrame; по умолчанию целочисленный индекс 0, 1, 2, …
- **params** — параметры для подстановки в запрос (см. §10.2).

Один вызов read_sql выполняет один запрос; для сложной выгрузки можно объединять несколько запросов или использовать один SELECT с JOIN/подзапросами.

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("sqlite:///example.db")
df = pd.read_sql("SELECT * FROM users LIMIT 100", engine)
# df — DataFrame с результатом запроса
```

Если таблица или БД ещё не созданы, сначала их создают средствами СУБД или скриптами миграций; Pandas и SQLAlchemy здесь только читают и при необходимости пишут данные (to_sql — в §10.2).

---

## Ключевое

- Для работы с БД из Python используются **SQLAlchemy** (единый API) и **драйвер** для СУБД (для PostgreSQL — **psycopg2** или **psycopg2-binary**).
- **create_engine(url)** создаёт движок подключения; URL задаёт СУБД, хост, БД, пользователя и пароль (секреты лучше брать из переменных окружения).
- **pd.read_sql(sql, con=engine)** выполняет SQL-запрос и возвращает результат в виде **DataFrame**; **con** — движок или соединение.

В [§10.2](chapter-10-02.md) разберём запросы SELECT в read_sql, параметризованные запросы и запись таблицы в БД (to_sql).
