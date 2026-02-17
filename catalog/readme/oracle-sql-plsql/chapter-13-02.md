# §13.2 Курсорные переменные

В [§13.1](chapter-13-01.md) мы разобрали явные курсоры: запрос жёстко задан в объявлении. **Курсорная переменная** — переменная типа **REF CURSOR**: ей можно присвоить (открыть) **разные** запросы в runtime. Одна и та же переменная в разных ветках кода может указывать на разные SELECT; курсорную переменную можно передавать в подпрограммы как параметр и возвращать из функции. В этом разделе — тип **SYS_REFCURSOR** и объявление типа **REF CURSOR**, **OPEN** переменная **FOR** SELECT, **FETCH** и **CLOSE**, передача курсорной переменной в процедуру. Обработка исключений — в [§13.3](chapter-13-03.md). См. [Глоссарий](glossary.md).

---

## 13.2.1. Тип REF CURSOR и SYS_REFCURSOR

**Курсорная переменная** — переменная, хранящая ссылку на курсор (результат запроса). Её тип — **REF CURSOR**. В Oracle предопределён тип **SYS_REFCURSOR** — слабо типизированный REF CURSOR: переменная может быть открыта для любого запроса. См. [Глоссарий](glossary.md).

Объявление:

```plsql
имя_переменной SYS_REFCURSOR;
```

Либо объявляют именованный тип REF CURSOR с возвращаемым типом записи (strong ref cursor), например: **TYPE t_emp_cur IS REF CURSOR RETURN employees%ROWTYPE;** Тогда переменная такого типа можно открывать только для запросов, структура результата которых совпадает с employees%ROWTYPE. Для гибкости часто используют SYS_REFCURSOR. См. [Глоссарий](glossary.md). Среда: Oracle PL/SQL.

---

## 13.2.2. OPEN FOR и FETCH

**OPEN** курсорная_переменная **FOR** запрос **;** — выполнить запрос и связать результат с переменной. Запрос может быть строкой (динамический SQL) или статическим SELECT. После OPEN работают те же **FETCH** и **CLOSE**, что и для явного курсора, но указывают **переменную**, а не имя курсора. См. [Глоссарий](glossary.md).

Пример: открыть курсорную переменную для одного из запросов в зависимости от условия:

```plsql
DECLARE
  cv     SYS_REFCURSOR;
  r_emp  employees%ROWTYPE;
  v_dept NUMBER := 10;
BEGIN
  OPEN cv FOR
    SELECT * FROM employees WHERE department_id = v_dept;
  LOOP
    FETCH cv INTO r_emp;
    EXIT WHEN cv%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(r_emp.last_name);
  END LOOP;
  CLOSE cv;
END;
/
```

Если бы запрос зависел от параметра (разные столбцы или таблицы), можно было бы формировать строку запроса и открывать **OPEN** cv **FOR** строка_запроса **(USING** биндинги**)** (динамический SQL). См. [Глоссарий](glossary.md).

---

## 13.2.3. Передача курсорной переменной в подпрограмму

Курсорную переменную можно передавать как параметр (IN OUT или OUT): процедура открывает курсор (OPEN ... FOR) и вызывающий код выполняет FETCH и CLOSE. Либо вызывающий открывает курсор и передаёт переменную в процедуру для обхода. Тип параметра — SYS_REFCURSOR (или объявленный REF CURSOR тип). См. [Глоссарий](glossary.md).

Пример: процедура открывает курсор по переданному отделу и возвращает курсорную переменную вызывающему:

```plsql
CREATE OR REPLACE PROCEDURE get_emps_by_dept (
  p_dept_id IN  employees.department_id%TYPE,
  p_cur     OUT SYS_REFCURSOR
) IS
BEGIN
  OPEN p_cur FOR
    SELECT employee_id, last_name, salary
      FROM employees
     WHERE department_id = p_dept_id;
END;
/

-- вызов
DECLARE
  cv  SYS_REFCURSOR;
  r_id   employees.employee_id%TYPE;
  r_name employees.last_name%TYPE;
  r_sal  employees.salary%TYPE;
BEGIN
  get_emps_by_dept(10, cv);
  LOOP
    FETCH cv INTO r_id, r_name, r_sal;
    EXIT WHEN cv%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(r_name);
  END LOOP;
  CLOSE cv;
END;
/
```

Вызывающий обязан закрыть курсор (CLOSE). См. [Глоссарий](glossary.md).

---

## Ключевое

- **Курсорная переменная** — переменная типа **REF CURSOR** (например, **SYS_REFCURSOR**): привязка к запросу в runtime. **OPEN** переменная **FOR** SELECT ... ; **FETCH** переменная **INTO** ... ; **CLOSE** переменная **;** См. [Глоссарий](glossary.md).
- **SYS_REFCURSOR** — предопределённый слабо типизированный REF CURSOR; можно открыть для любого запроса. Strong REF CURSOR (TYPE ... IS REF CURSOR RETURN ...) ограничивает структуру результата.
- Курсорную переменную можно передавать в подпрограммы (IN OUT, OUT); типичный сценарий — процедура открывает курсор и возвращает его вызывающему для FETCH и CLOSE.
- Закрывать курсор (CLOSE) должен тот, кто будет им пользоваться после открытия (обычно вызывающий, если открытие в процедуре).

В [§13.3](chapter-13-03.md) мы разберём обработку исключений: предопределённые и пользовательские исключения, RAISE, PRAGMA EXCEPTION_INIT.
