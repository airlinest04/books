# §13.1 Явные курсоры

В [§12.3](chapter-12-03.md) мы завершили главу о триггерах и пакетах. **SELECT INTO** ([§9.3](chapter-09-03.md)) возвращает ровно одну строку; для обхода **нескольких** строк результата запроса в PL/SQL используют **курсор**. **Явный курсор** — именованный курсор, объявленный в блоке или пакете: ему сопоставлен запрос SELECT, затем курсор **открывают** (OPEN), **читают** строки (FETCH) и **закрывают** (CLOSE). В этом разделе — объявление курсора (**CURSOR** ... **IS** SELECT), **OPEN**, **FETCH INTO**, **CLOSE**, атрибуты **%NOTFOUND**, **%ROWCOUNT** и типичный цикл обхода. Курсорные переменные — в [§13.2](chapter-13-02.md). См. [Глоссарий](glossary.md).

---

## 13.1.1. Объявление и открытие курсора

**Курсор** — указатель на результат запроса SELECT: набор строк, которые можно читать по одной. **Явный курсор** объявляется в DECLARE с привязкой к конкретному запросу. См. [Глоссарий](glossary.md).

Синтаксис объявления:

```plsql
CURSOR имя_курсора [ (параметр1 тип [, ...]) ] IS
  SELECT ... ;
```

Параметры курсора (опционально) передаются в запрос как константы при OPEN. Пример курсора без параметров:

```plsql
DECLARE
  CURSOR cur_emps IS
    SELECT employee_id, last_name, salary FROM employees WHERE department_id = 10;
  r_emp cur_emps%ROWTYPE;
BEGIN
  OPEN cur_emps;   -- выполнение запроса, курсор готов к чтению
  -- FETCH ...
  CLOSE cur_emps;
END;
/
```

**OPEN** имя_курсора **[**(аргументы)**]** — выполнить запрос курсора и подготовить его к FETCH. Если у курсора есть параметры, они передаются при OPEN. После OPEN можно вызывать FETCH до исчерпания строк или явного CLOSE. См. [Глоссарий](glossary.md). Среда: Oracle PL/SQL.

---

## 13.1.2. FETCH и CLOSE

**FETCH** имя_курсора **INTO** список_переменных **;** — прочитать **следующую** строку результата и поместить значения столбцов в переменные (или в одну переменную типа %ROWTYPE курсора). Количество и порядок переменных должны соответствовать столбцам SELECT. Если строк больше нет, FETCH не заполняет переменные и атрибут курсора **%NOTFOUND** становится TRUE. См. [Глоссарий](glossary.md).

**CLOSE** имя_курсора **;** — закрыть курсор и освободить ресурсы. После CLOSE курсор снова можно открыть (OPEN). Нельзя FETCH из закрытого курсора. Рекомендуется всегда закрывать курсор (в блоке EXCEPTION или в конце ветки), иначе возможна утечка ресурсов. См. [Глоссарий](glossary.md).

Пример с записью %ROWTYPE:

```plsql
DECLARE
  CURSOR cur_emps IS SELECT employee_id, last_name, salary FROM employees WHERE department_id = 10;
  r_emp cur_emps%ROWTYPE;
BEGIN
  OPEN cur_emps;
  LOOP
    FETCH cur_emps INTO r_emp;
    EXIT WHEN cur_emps%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(r_emp.employee_id || ' ' || r_emp.last_name || ' ' || r_emp.salary);
  END LOOP;
  CLOSE cur_emps;
END;
/
```

См. [Глоссарий](glossary.md).

---

## 13.1.3. Атрибуты курсора %NOTFOUND, %FOUND, %ROWCOUNT

После каждого FETCH доступны атрибуты курсора:

- **%NOTFOUND** — TRUE, если последний FETCH не вернул строку (данные закончились). Используется для выхода из цикла. См. [Глоссарий](glossary.md).
- **%FOUND** — TRUE, если последний FETCH вернул строку (противоположно %NOTFOUND).
- **%ROWCOUNT** — число строк, уже прочитанных из курсора с момента последнего OPEN.

До первого FETCH %NOTFOUND имеет значение NULL (в условии трактуется как ложь), %ROWCOUNT — 0. Проверку на конец данных делают **после** FETCH: FETCH ... ; EXIT WHEN курсор%NOTFOUND; См. [Глоссарий](glossary.md).

---

## 13.1.4. Параметризованный курсор

Курсор может иметь параметры: они перечисляются в объявлении в скобках и используются в теле SELECT как константы. При **OPEN** передают фактические значения. Разные вызовы OPEN с разными аргументами дают разные наборы данных. См. [Глоссарий](glossary.md).

Пример:

```plsql
DECLARE
  CURSOR cur_dept(p_dept_id NUMBER) IS
    SELECT employee_id, last_name FROM employees WHERE department_id = p_dept_id;
  r cur_dept%ROWTYPE;
BEGIN
  OPEN cur_dept(10);
  LOOP
    FETCH cur_dept INTO r;
    EXIT WHEN cur_dept%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(r.last_name);
  END LOOP;
  CLOSE cur_dept;
  OPEN cur_dept(20);  -- другой отдел
  -- ...
  CLOSE cur_dept;
END;
/
```

См. [Глоссарий](glossary.md).

---

## Ключевое

- **Явный курсор** объявляется в DECLARE: **CURSOR** имя **[**(параметры)**] IS** SELECT ... ; Управление: **OPEN** имя **[**(аргументы)**]** ; **FETCH** имя **INTO** переменные **;** **CLOSE** имя **;** См. [Глоссарий](glossary.md).
- **FETCH** читает следующую строку; при отсутствии строк **%NOTFOUND** = TRUE. Типичный цикл: LOOP → FETCH → EXIT WHEN курсор%NOTFOUND → обработка → END LOOP; затем CLOSE.
- **%ROWCOUNT** — число прочитанных строк с момента OPEN. **%FOUND** — TRUE, если последний FETCH вернул строку.
- Параметризованный курсор: параметры в объявлении, значения передаются в OPEN. После CLOSE курсор можно открыть снова (с теми же или другими аргументами).

В [§13.2](chapter-13-02.md) мы рассмотрим курсорные переменные: курсор, привязываемый к разным запросам в runtime.
