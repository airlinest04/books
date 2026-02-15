# §6.7 Массовая загрузка (COPY)

В [§6.6](chapter-06-06.md) мы рассмотрели представления. Для загрузки и выгрузки больших объёмов данных команда **COPY** эффективнее множества INSERT. COPY работает с файлами напрямую на стороне сервера; для доступа к файлам клиента используется **\copy** в psql. В этом разделе разберём COPY FROM и COPY TO, форматы text и CSV, типичные опции и отличия COPY от \copy. Часть II (агрегация) начинается с [§7.1](chapter-07-01.md). См. [документацию COPY](https://www.postgresql.org/docs/current/sql-copy.html).

---

## 6.7.1. Зачем нужен COPY

INSERT удобен для небольшого числа строк. При загрузке тысяч или миллионов записей из CSV или текстового файла COPY работает значительно быстрее: данные передаются в «сыром» виде, без разбора отдельных предложений SQL.

COPY выполняет **копирование между таблицей и файлом**: FROM — загрузка из файла в таблицу, TO — выгрузка из таблицы в файл. Запись добавляется к существующим данным в таблице; заменить содержимое нельзя (для этого предварительно TRUNCATE или DELETE).

---

## 6.7.2. COPY FROM — загрузка в таблицу

Синтаксис по [документации](https://www.postgresql.org/docs/current/sql-copy.html):

```sql
COPY имя_таблицы [ (столбец1 [, столбец2, ...] ) ]
FROM { 'путь_к_файлу' | PROGRAM 'команда' | STDIN }
[ [ WITH ] ( опция [, ...] ) ]
[ WHERE условие ];
```

Файл указывается с точки зрения **сервера**: путь должен быть абсолютным (для COPY TO — обязательно), файл должен быть доступен пользователю, под которым работает PostgreSQL. Пользователю нужны права `pg_read_server_files` или роль суперпользователя.

Пример:

```sql
COPY products (name, price, category_id) FROM '/var/data/products.csv' WITH (FORMAT csv, HEADER true);
```

Список столбцов необязателен; при его отсутствии используются все столбцы таблицы (кроме сгенерированных) в порядке объявления. Файл должен содержать значения в том же порядке. Столбцы, не указанные в списке, получают DEFAULT или NULL.

---

## 6.7.3. COPY TO — выгрузка в файл

```sql
COPY { имя_таблицы [ (столбец1 [, ...] ) ] | ( запрос ) }
TO { 'путь_к_файлу' | PROGRAM 'команда' | STDOUT }
[ [ WITH ] ( опция [, ...] ) ];
```

Можно выгружать таблицу целиком или результат запроса:

```sql
COPY products TO '/var/backup/products.csv' WITH (FORMAT csv, HEADER true);

COPY (SELECT id, name FROM products WHERE category_id = 1) TO '/var/backup/category1.csv' WITH CSV;
```

Файл создаётся на сервере. При указании STDOUT данные идут в клиентское подключение (например, в psql).

---

## 6.7.4. Форматы: text, CSV, binary

- **text** (по умолчанию) — текстовый формат: столбцы разделены разделителем (по умолчанию табуляция), NULL — `\N`, конец данных — `\.` при COPY FROM STDIN.
- **csv** — CSV: разделитель запятая, значения в кавычках при необходимости, стандартные правила экранирования.
- **binary** — бинарный формат PostgreSQL; быстрее, но не переносим между архитектурами и версиями.

Для обмена с другими программами обычно используют CSV:

```sql
COPY products FROM '/data/products.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',');
COPY products TO STDOUT WITH (FORMAT csv, HEADER true);
```

---

## 6.7.5. Типичные опции

| Опция | Назначение | Пример |
|-------|------------|--------|
| FORMAT | text, csv, binary | `FORMAT csv` |
| DELIMITER | Разделитель столбцов | `DELIMITER ';'` |
| NULL | Строка для NULL | `NULL ''` (пустая строка) |
| HEADER | Первая строка — заголовки | `HEADER true` |
| ENCODING | Кодировка файла | `ENCODING 'UTF8'` |
| ON_ERROR | stop (по умолчанию) или ignore | `ON_ERROR ignore` |
| REJECT_LIMIT | Лимит ошибок при ON_ERROR ignore | `REJECT_LIMIT 100` |

Пример загрузки CSV с заголовком и обработкой ошибок:

```sql
COPY logs FROM '/var/logs/import.csv' WITH (
  FORMAT csv,
  HEADER true,
  DELIMITER ',',
  NULL '',
  ENCODING 'UTF8',
  ON_ERROR ignore,
  REJECT_LIMIT 1000
);
```

При `ON_ERROR ignore` ошибочные строки пропускаются; REJECT_LIMIT ограничивает допустимое число пропусков, после чего COPY прерывается.

---

## 6.7.6. WHERE при COPY FROM

Начиная с PostgreSQL 15, в COPY FROM можно указать **WHERE**: в таблицу попадут только строки, удовлетворяющие условию:

```sql
COPY logs FROM '/var/logs/data.csv' WITH (FORMAT csv)
WHERE level = 'ERROR';
```

В WHERE нельзя использовать подзапросы.

---

## 6.7.7. STDIN и STDOUT

При **STDIN** и **STDOUT** данные передаются по соединению клиент–сервер. Файл на диске не используется; это удобно для передачи данных между процессами или при работе через сеть.

Пример в psql — загрузка из stdin:

```sql
COPY products FROM STDIN WITH (FORMAT csv, HEADER true);
```

После выполнения команды psql ожидает ввод строк; ввод завершается строкой `\.` или Ctrl+D.

---

## 6.7.8. \copy в psql — файлы на стороне клиента

Команда **COPY** с путём к файлу выполняется **на сервере**: файл должен находиться на машине с PostgreSQL. Если файл на вашем компьютере (клиенте), используйте мета-команду **\copy** в psql. Она запускает COPY ... FROM STDIN / TO STDOUT и передаёт данные через соединение.

```text
\copy products FROM '/home/user/products.csv' WITH (FORMAT csv, HEADER true)
\copy (SELECT * FROM products) TO '/home/user/export.csv' WITH CSV
```

Синтаксис после \copy такой же, как у COPY; путь к файлу — локальный для клиента. Права на файл проверяются на стороне клиента.

---

## 6.7.9. PROGRAM — выполнение команды

Вместо файла можно указать **PROGRAM**: сервер выполняет команду и читает из stdout (COPY FROM) или пишет в stdin (COPY TO). Команда выполняется от имени пользователя PostgreSQL.

```sql
COPY products FROM PROGRAM 'gunzip -c /var/data/products.csv.gz' WITH (FORMAT csv);
COPY products TO PROGRAM 'gzip > /var/backup/products.csv.gz' WITH CSV;
```

Требуются права `pg_execute_server_program`. Строки с аргументами из ненадёжных источников нужно экранировать.

---

## 6.7.10. Ограничения и замечания

- Файлы в COPY (не \copy) должны быть доступны **серверу**, не клиенту.
- При COPY FROM срабатывают триггеры и проверки ограничений; правила (rules) — нет.
- При сбое COPY FROM часть строк может остаться в таблице в «удалённом» состоянии; для освобождения места выполняют VACUUM.
- COPY TO не поддерживает партиционированные таблицы и наследование напрямую; используйте подзапрос: `COPY (SELECT * FROM таблица) TO ...`.
- Таблицы с row-level security не поддерживают COPY FROM; применяйте INSERT.

---

## Ключевое

- **COPY** — массовая загрузка и выгрузка между таблицей и файлом; эффективнее множества INSERT.
- **COPY FROM** — загрузка из файла, **COPY TO** — выгрузка в файл или в запрос.
- Путь к файлу в COPY — на стороне сервера; для клиентских файлов используется **\copy** в psql.
- Форматы: **text** (по умолчанию), **csv**, **binary**.
- Важные опции: FORMAT, DELIMITER, NULL, HEADER, ENCODING, ON_ERROR, REJECT_LIMIT.
- **STDIN** / **STDOUT** — передача данных по соединению; **PROGRAM** — выполнение команды на сервере.

В [§7.1](chapter-07-01.md) начнём часть II — агрегатные функции (SUM, COUNT, AVG и др.).
