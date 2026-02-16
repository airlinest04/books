# §10.3 EXCEPT

В [§10.2](chapter-10-02.md) мы рассмотрели INTERSECT — пересечение множеств. **EXCEPT** возвращает строки из первого запроса, **которых нет во втором** — разность множеств. Порядок запросов важен: A EXCEPT B не совпадает с B EXCEPT A. В этом разделе разберём синтаксис, EXCEPT и EXCEPT ALL, а также типичные сценарии. Комбинирование нескольких операций — в [§10.4](chapter-10-04.md). См. [документацию SELECT](https://www.postgresql.org/docs/current/sql-select.html).

---

## 10.3.1. Синтаксис

```sql
запрос1
EXCEPT [ALL]
запрос2
[ORDER BY ...]
[LIMIT ...];
```

Результат — строки из `запрос1`, которых нет в `запрос2`. Имена столбцов берутся из первого запроса. Правила совместимости столбцов те же, что у UNION и INTERSECT.

---

## 10.3.2. EXCEPT и EXCEPT ALL

- **EXCEPT** (эквивалентно EXCEPT DISTINCT) — каждая строка в результате встречается **один раз**; строки из первого запроса, отсутствующие во втором, без дубликатов.
- **EXCEPT ALL** — для каждой строки вычитается количество её вхождений во втором запросе. Если строка встречается в первом 3 раза, во втором 1 раз, в результате будет 2 экземпляра. Если во втором не меньше, чем в первом, строка в результат не попадает.

Пример: клиенты, которые делали заказы, но никогда не возвращали товар:

```sql
SELECT customer_id FROM orders
EXCEPT
SELECT customer_id FROM refunds;
```

Только те `customer_id`, которые есть в `orders` и отсутствуют в `refunds`.

---

## 10.3.3. Порядок запросов важен

EXCEPT не коммутативен. Результат зависит от того, какой запрос первый, какой второй.

```sql
-- Клиенты с заказами, но без возвратов:
SELECT customer_id FROM orders
EXCEPT
SELECT customer_id FROM refunds;

-- Клиенты с возвратами, но без заказов (аномалия или другая таблица):
SELECT customer_id FROM refunds
EXCEPT
SELECT customer_id FROM orders;
```

В первом случае — «только покупатели», во втором — «только возвращавшие» (если такие есть по модели данных).

---

## 10.3.4. Сравнение с NOT IN и NOT EXISTS

Для одной колонки EXCEPT можно заменить подзапросом:

```sql
-- Через EXCEPT:
SELECT DISTINCT customer_id FROM orders
EXCEPT
SELECT customer_id FROM refunds;

-- Через NOT IN (осторожно с NULL в refunds.customer_id):
SELECT DISTINCT customer_id FROM orders
WHERE customer_id NOT IN (SELECT customer_id FROM refunds WHERE customer_id IS NOT NULL);

-- Через NOT EXISTS (безопаснее при NULL):
SELECT DISTINCT c.id FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)
  AND NOT EXISTS (SELECT 1 FROM refunds r WHERE r.customer_id = c.id);
```

EXCEPT короче, когда сравниваются целые строки. NOT EXISTS удобен, когда нужна связь с другой таблицей (например, `customers`).

---

## 10.3.5. Пример: категории без дешёвых товаров

Категории, в которых нет товаров дешевле 100:

```sql
SELECT category_id FROM products
EXCEPT
SELECT category_id FROM products WHERE price < 100;
```

Результат — категории, где все товары стоят 100 и выше. Альтернатива через NOT IN или NOT EXISTS по подзапросу.

---

## Ключевое

- **EXCEPT** возвращает строки из первого запроса, которых нет во втором — разность множеств.
- Порядок запросов важен: A EXCEPT B ≠ B EXCEPT A.
- EXCEPT устраняет дубликаты; **EXCEPT ALL** вычитает вхождения (количество = в первом минус во втором, но не меньше нуля).
- Для одной колонки можно использовать NOT IN или NOT EXISTS; EXCEPT удобен для сравнения целых строк.

В [§10.4](chapter-10-04.md) рассмотрим порядок выполнения и скобки при комбинировании UNION, INTERSECT и EXCEPT.
