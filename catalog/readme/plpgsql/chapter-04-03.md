# §4.3 Получение информации об ошибке

В [§4.2](chapter-04-02.md) мы разобрали условия в WHEN. В обработчике часто нужно **узнать код и текст ошибки** — для логирования, для условной логики или для перевыброса с дополнением. В этом разделе рассмотрим встроенные переменные **SQLSTATE** и **SQLERRM** в обработчике исключений и команду **GET STACKED DIAGNOSTICS** для сохранения деталей ошибки в переменные. Явный вызов исключения (RAISE) — в [§4.4](chapter-04-04.md). См. [Глоссарий](glossary.md).

---

## 4.3.1. SQLSTATE и SQLERRM в обработчике

Внутри блока **EXCEPTION** (в операторах любого из обработчиков WHEN ... THEN) доступны две встроенные переменные (по документации PostgreSQL):

- **SQLSTATE** — пятисимвольный код ошибки (тот же, что в приложении A документации). Тип — text. Соответствует коду перехваченного исключения.
- **SQLERRM** — текст сообщения об ошибке, связанный с этим исключением. Тип — text. Может зависеть от локали и версии сервера.

**Вне** обработчиков исключений эти переменные **не определены**; обращение к ним вне EXCEPTION приводит к неопределённому поведению или ошибке. Использовать их имеет смысл только в коде между WHEN ... THEN и следующим WHEN или END. См. [Глоссарий](glossary.md).

Пример: логирование и перевыброс с контекстом:

```sql
EXCEPTION
  WHEN OTHERS THEN
    RAISE NOTICE 'Ошибка: % (код: %)', SQLERRM, SQLSTATE;
    RAISE;   -- перевыбросить текущее исключение (см. §4.4)
END;
```

---

## 4.3.2. Сохранение в переменные для логирования

Чтобы передать информацию об ошибке в таблицу логов, в OUT-параметр или в другой блок, значения **SQLSTATE** и **SQLERRM** нужно присвоить локальным переменным **внутри** обработчика. После выхода из блока EXCEPTION встроенные переменные уже недоступны, а скопированные значения остаются в переменных. См. [Глоссарий](glossary.md).

Пример:

```sql
DECLARE
  err_code text;
  err_msg  text;
BEGIN
  -- операторы
EXCEPTION
  WHEN OTHERS THEN
    err_code := SQLSTATE;
    err_msg  := SQLERRM;
    INSERT INTO error_log (code, message, occurred_at)
    VALUES (err_code, err_msg, now());
    RAISE;
END;
```

Тип переменных — text (или varchar с достаточной длиной). Для SQLSTATE достаточно 5 символов; для SQLERRM лучше зарезервировать больший размер (например, text).

---

## 4.3.3. GET STACKED DIAGNOSTICS

Для получения **расширенной** информации об ошибке в обработчике используется команда **GET STACKED DIAGNOSTICS** (по документации PostgreSQL). Она присваивает выбранные поля диагностики переменным. Синтаксис:

```
GET STACKED DIAGNOSTICS переменная { = | := } элемент [ , ... ];
```

**Элемент** — ключевое слово, идентифицирующее поле (см. таблицу ниже). **Переменная** должна иметь подходящий тип (обычно text). Каждый элемент запрашивается отдельной парой переменная = элемент. См. [Глоссарий](glossary.md).

Основные элементы (по документации PostgreSQL, таблица Error Diagnostics Items):

| Элемент | Тип | Описание |
|---------|-----|----------|
| RETURNED_SQLSTATE | text | Код SQLSTATE исключения |
| COLUMN_NAME | text | Имя столбца, связанного с ошибкой (если применимо) |
| CONSTRAINT_NAME | text | Имя ограничения (если применимо) |
| PG_DATATYPE_NAME | text | Имя типа данных (если применимо) |
| MESSAGE_TEXT | text | Текст сообщения об ошибке |
| TABLE_NAME | text | Имя таблицы (если применимо) |
| SCHEMA_NAME | text | Имя схемы (если применимо) |
| PG_EXCEPTION_DETAIL | text | Детали (строка DETAIL из сообщения) |
| PG_EXCEPTION_HINT | text | Подсказка (строка HINT) |
| PG_EXCEPTION_CONTEXT | text | Контекст (стек вызовов до точки ошибки) |

Пример:

```sql
EXCEPTION
  WHEN OTHERS THEN
    GET STACKED DIAGNOSTICS
      v_state   = RETURNED_SQLSTATE,
      v_msg     = MESSAGE_TEXT,
      v_detail  = PG_EXCEPTION_DETAIL,
      v_context = PG_EXCEPTION_CONTEXT;
    RAISE NOTICE 'Ошибка: % % %', v_state, v_msg, v_detail;
    RAISE NOTICE 'Контекст: %', v_context;
    RAISE;
END;
```

GET STACKED DIAGNOSTICS имеет смысл вызывать только **внутри** обработчика EXCEPTION; снаружи «стековая» информация об последнем исключении недоступна.

---

## 4.3.4. Отличие от GET DIAGNOSTICS (CURRENT)

**GET DIAGNOSTICS** (или GET CURRENT DIAGNOSTICS) возвращает **текущие** показатели выполнения (например, ROW_COUNT после последней SQL-команды, PG_CONTEXT). Это не информация об исключении, а общий статус. **GET STACKED DIAGNOSTICS** возвращает данные именно о **перехваченном исключении** и вызывается только в обработчике EXCEPTION. Не путать эти две команды. См. документацию PostgreSQL, разделы 41.5.5 (Obtaining the Result Status) и 41.6.8.1 (Obtaining Information about an Error).

---

## Ключевое

- В обработчике EXCEPTION доступны **SQLSTATE** (код ошибки) и **SQLERRM** (текст сообщения); вне обработчика они не определены.
- Для логирования или передачи наружу сохраняйте их в локальные переменные **внутри** обработчика.
- **GET STACKED DIAGNOSTICS** присваивает переменным детали перехваченного исключения: RETURNED_SQLSTATE, MESSAGE_TEXT, PG_EXCEPTION_DETAIL, PG_EXCEPTION_HINT, PG_EXCEPTION_CONTEXT, а также COLUMN_NAME, CONSTRAINT_NAME, TABLE_NAME, SCHEMA_NAME и др. при наличии.
- GET STACKED DIAGNOSTICS вызывают только в обработчике; не путать с GET DIAGNOSTICS (текущий статус, ROW_COUNT и т.д.).

В [§4.4](chapter-04-04.md) мы разберём RAISE: явный вызов исключения, уровни сообщений (EXCEPTION, WARNING, NOTICE) и перевыброс (RAISE без параметров).
