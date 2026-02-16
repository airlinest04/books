# §6.5 MERGE (UPSERT)

В [§6.4](chapter-06-04.md) мы рассмотрели меры предосторожности при UPDATE и DELETE. **UPSERT** (update or insert) — типичная задача: при наличии строки обновить её, при отсутствии — вставить. PostgreSQL предлагает два подхода: **MERGE** (стандарт SQL, начиная с версии 15) и **INSERT ... ON CONFLICT** (расширение PostgreSQL). В этом разделе разберём синтаксис MERGE, WHEN MATCHED и WHEN NOT MATCHED, альтернативу ON CONFLICT и области применения. Представления — в [§6.6](chapter-06-06.md). См. [документацию MERGE](https://www.postgresql.org/docs/current/sql-merge.html) и [INSERT ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html).

---

## 6.5.1. Задача UPSERT

Часто требуется «вставить или обновить» по уникальному ключу: если строка с таким ключом есть — обновить её столбцы; если нет — вставить новую. Без специального синтаксиса пришлось бы проверять существование (SELECT) и выполнять INSERT или UPDATE отдельно, что громоздко и создаёт гонки при параллельных транзакциях.

UPSERT решает задачу атомарно: одна команда либо вставляет, либо обновляет.

---

## 6.5.2. MERGE — обзор

Команда **MERGE** выполняет соединение источника данных с целевой таблицей по условию и в зависимости от совпадения применяет действия: INSERT, UPDATE или DELETE. Синтаксис по [документации](https://www.postgresql.org/docs/current/sql-merge.html):

```sql
MERGE INTO целевая_таблица [ AS псевдоним_цели ]
USING источник ON условие_соединения
WHEN MATCHED [ AND условие ] THEN { UPDATE SET ... | DELETE | DO NOTHING }
WHEN NOT MATCHED [ BY TARGET ] [ AND условие ] THEN { INSERT ... | DO NOTHING }
[ WHEN NOT MATCHED BY SOURCE [ AND условие ] THEN { UPDATE SET ... | DELETE | DO NOTHING } ]
[ RETURNING ... ];
```

- **целевая_таблица** — таблица, в которую сливаются данные;
- **источник** — таблица, представление или подзапрос (SELECT / VALUES);
- **условие_соединения** — как в JOIN: задаёт, когда строка источника совпадает со строкой цели (обычно по уникальному ключу);
- **WHEN MATCHED** — строка источника нашла совпадение в цели: можно обновить, удалить или пропустить;
- **WHEN NOT MATCHED [BY TARGET]** — строка источника не нашла совпадения в цели: можно вставить новую строку;
- **WHEN NOT MATCHED BY SOURCE** — строка цели не совпала ни с одной строкой источника: можно обновить или удалить её (удобно для синхронизации со справочником).

MERGE доступен в PostgreSQL начиная с версии 15.

---

## 6.5.3. Базовый MERGE: вставка или обновление

Пример: таблица `customer_accounts` с балансом; при поступлении транзакции нужно либо обновить баланс существующего клиента, либо создать запись для нового. По [примеру из документации](https://www.postgresql.org/docs/current/sql-merge.html):

```sql
MERGE INTO customer_account ca
USING recent_transactions t
ON t.customer_id = ca.customer_id
WHEN MATCHED THEN
  UPDATE SET balance = balance + transaction_value
WHEN NOT MATCHED THEN
  INSERT (customer_id, balance)
  VALUES (t.customer_id, t.transaction_value);
```

- Для каждой строки из `recent_transactions` ищется строка в `customer_account` по `customer_id`.
- При совпадении (`WHEN MATCHED`) — обновляется `balance`.
- При отсутствии совпадения (`WHEN NOT MATCHED`) — вставляется новая строка.

В `INSERT` можно ссылаться только на столбцы источника (`t.customer_id`, `t.transaction_value`); в `UPDATE SET` — на столбцы и цели, и источника.

---

## 6.5.4. Условия в WHEN и несколько веток

В каждую ветку можно добавить `AND условие`. Первая подходящая ветка выполняется для строки. Пример: обновление остатка товара; при нулевом остатке — удаление; новые товары — только при положительном приходе:

```sql
MERGE INTO wines w
USING wine_stock_changes s
ON s.winename = w.winename
WHEN NOT MATCHED AND s.stock_delta > 0 THEN
  INSERT VALUES (s.winename, s.stock_delta)
WHEN MATCHED AND w.stock + s.stock_delta > 0 THEN
  UPDATE SET stock = w.stock + s.stock_delta
WHEN MATCHED THEN
  DELETE;
```

Обратите внимание: ветки одного типа (например, две `WHEN MATCHED`) обрабатываются по порядку; вторая может сработать только если первая не подошла по условию.

---

## 6.5.5. WHEN NOT MATCHED BY SOURCE — синхронизация

**WHEN NOT MATCHED BY SOURCE** срабатывает для строк целевой таблицы, для которых нет соответствующей строки в источнике. Это позволяет синхронизировать целевую таблицу со справочником: вставить новые, обновить изменившиеся, удалить отсутствующие. По [документации](https://www.postgresql.org/docs/current/sql-merge.html):

```sql
MERGE INTO wines w
USING new_wine_list s
ON s.winename = w.winename
WHEN NOT MATCHED BY TARGET THEN
  INSERT VALUES (s.winename, s.stock)
WHEN MATCHED AND w.stock != s.stock THEN
  UPDATE SET stock = s.stock
WHEN NOT MATCHED BY SOURCE THEN
  DELETE;
```

Здесь `new_wine_list` — актуальный список; строки, которых нет в источнике, удаляются из `wines`.

При использовании `WHEN NOT MATCHED BY SOURCE` и `WHEN NOT MATCHED [BY TARGET]` выполняется полное соединение (FULL JOIN) источника и цели; в `ON` должен использоваться оператор, поддерживающий hash- или merge-join.

---

## 6.5.6. RETURNING и merge_action()

MERGE поддерживает **RETURNING**. По умолчанию для INSERT и UPDATE возвращаются новые значения, для DELETE — старые. Функция **merge_action()** возвращает тип выполненного действия: `'INSERT'`, `'UPDATE'` или `'DELETE'`:

```sql
MERGE INTO wines w
USING wine_stock_changes s
ON s.winename = w.winename
WHEN NOT MATCHED AND s.stock_delta > 0 THEN
  INSERT VALUES (s.winename, s.stock_delta)
WHEN MATCHED AND w.stock + s.stock_delta > 0 THEN
  UPDATE SET stock = w.stock + s.stock_delta
WHEN MATCHED THEN
  DELETE
RETURNING merge_action(), w.winename, old.stock AS old_stock, new.stock AS new_stock;
```

---

## 6.5.7. INSERT ... ON CONFLICT — альтернатива для простого upsert

Для случая «вставить или обновить при конфликте по уникальному ключу» в PostgreSQL давно используется **INSERT ... ON CONFLICT** (см. [документацию INSERT](https://www.postgresql.org/docs/current/sql-insert.html)). Он работает во всех современных версиях и не требует MERGE.

Синтаксис:

```sql
INSERT INTO таблица (столбцы)
VALUES (значения)
ON CONFLICT (столбец_уникальности) DO UPDATE SET
  столбец = EXCLUDED.столбец;
```

`EXCLUDED` — псевдоним строки, которую пытались вставить. Пример:

```sql
INSERT INTO distributors (did, dname)
VALUES (5, 'Gizmo Transglobal')
ON CONFLICT (did) DO UPDATE SET dname = EXCLUDED.dname;
```

При конфликте по `did` выполняется обновление `dname`. Для «ничего не делать при конфликте»:

```sql
INSERT INTO distributors (did, dname) VALUES (7, 'Redline GmbH')
ON CONFLICT (did) DO NOTHING;
```

Можно указать ограничение по имени: `ON CONFLICT ON CONSTRAINT имя_ограничения`. В `DO UPDATE` допускается `WHERE` для дополнительной фильтрации обновляемых строк.

---

## 6.5.8. MERGE и ON CONFLICT: когда что использовать

| Критерий | MERGE | INSERT ... ON CONFLICT |
|----------|-------|------------------------|
| Версия PostgreSQL | 15+ | 9.5+ |
| Стандарт SQL | Да | Расширение PostgreSQL |
| Простой upsert (вставить/обновить) | Да | Да, компактнее |
| Удаление при отсутствии в источнике | Да (WHEN NOT MATCHED BY SOURCE) | Нет |
| Несколько веток с разными условиями | Да | Ограничено (DO UPDATE / DO NOTHING) |
| Источник — запрос или таблица | Да | Только вставляемые строки |

Для простого upsert по одному ключу часто удобнее `ON CONFLICT`: синтаксис короче, работает в старых версиях. Для сложной синхронизации (вставка, обновление, удаление в одной команде) предпочтителен MERGE.

---

## 6.5.9. Ограничения и предупреждения

- Целевая таблица MERGE не должна быть материализованным представлением, внешней таблицей или таблицей с правилами (rules).
- В условии `ON` следует использовать только столбцы, участвующие в сопоставлении с источником; ссылки только на столбцы цели могут давать неожиданный результат.
- Важно, чтобы соединение производило не более одной пары (источник, цель) на каждую строку цели; иначе возможны ошибки уникальности или кардинальности.
- При параллельных MERGE и других изменениях целевой таблицы действуют обычные правила изоляции транзакций.

---

## Ключевое

- **UPSERT** — вставка или обновление в зависимости от наличия строки по уникальному ключу.
- **MERGE** (PostgreSQL 15+) — стандартная команда: соединение источника с целью, ветки WHEN MATCHED, WHEN NOT MATCHED [BY TARGET], WHEN NOT MATCHED BY SOURCE; действия INSERT, UPDATE, DELETE.
- **INSERT ... ON CONFLICT** — расширение PostgreSQL для простого upsert; `DO UPDATE SET` при конфликте, `EXCLUDED` для значений вставляемой строки, `DO NOTHING` для игнорирования.
- MERGE удобен для сложной синхронизации; ON CONFLICT — для простого «вставить или обновить» в любой современной версии.

В [§6.6](chapter-06-06.md) рассмотрим представления (views) — сохранённые запросы как виртуальные таблицы.
