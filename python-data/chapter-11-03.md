# §11.3 Пример: регрессия и классификация

В [§11.2](chapter-11-02.md) мы разобрали API Scikit-learn: fit(), predict(), train_test_split(). В этом разделе — **два минимальных примера**: **регрессия** (LinearRegression) с метрикой **MSE** и **классификация** (LogisticRegression или дерево решений) с метрикой **accuracy**. Данные предполагаются уже подготовленными (числовые признаки, без пропусков); подготовка данных и валидация подробнее — в [§12.2](chapter-12-02.md) и [§12.3](chapter-12-03.md).

---

## 11.3.1. Регрессия: LinearRegression и MSE

**Регрессия** — предсказание непрерывной величины. Модель **LinearRegression** (sklearn.linear_model) строит линейную зависимость целевой переменной от признаков: **y ≈ X @ coef + intercept**. После **fit(X_train, y_train)** коэффициенты хранятся в атрибутах **model.coef_** (вектор по признакам) и **model.intercept_** (свободный член). **predict(X_test)** возвращает предсказания для тестовой выборки.

**Метрика качества** регрессии — средняя квадратичная ошибка **MSE** (Mean Squared Error): среднее от квадратов разностей (y_true - y_pred). Функция **mean_squared_error(y_true, y_pred)** из **sklearn.metrics** вычисляет MSE; корень из неё — **RMSE** (в тех же единицах, что и y). Чем меньше MSE, тем в среднем ближе предсказания к истине (при этом одна модель может лучше другой по MSE, но хуже по другим метрикам — MAE, R²).

```python
from sklearn.linear_model import LinearRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)
model = LinearRegression()
model.fit(X_train, y_train)
y_pred = model.predict(X_test)
mse = mean_squared_error(y_test, y_pred)
# rmse = mse ** 0.5
```

---

## 11.3.2. Классификация: LogisticRegression или дерево и accuracy

**Классификация** — предсказание метки класса (категория). **LogisticRegression** (sklearn.linear_model) — линейная модель для вероятностей классов; **predict(X)** возвращает класс с максимальной оцененной вероятностью. **DecisionTreeClassifier** (sklearn.tree) — дерево решений: разбиения по признакам, в листьях — предсказанный класс. Обе модели вызывают **fit(X_train, y_train)** и **predict(X_test)** так же, как в регрессии.

**Метрика качества** классификации — **accuracy** (точность): доля правильных ответов среди всех объектов. Функция **accuracy_score(y_true, y_pred)** из **sklearn.metrics** вычисляет её (число от 0 до 1 или в процентах при нормализации). Для сбалансированных классов accuracy информативна; при сильном дисбалансе полезны также precision, recall, F1 (они в книге не разбираются).

```python
from sklearn.linear_model import LogisticRegression
from sklearn.tree import DecisionTreeClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import accuracy_score

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)
model = LogisticRegression()  # или DecisionTreeClassifier()
model.fit(X_train, y_train)
y_pred = model.predict(X_test)
acc = accuracy_score(y_test, y_pred)
```

---

## 11.3.3. Сводка по метрикам

- **Регрессия:** **MSE** (mean_squared_error) — средний квадрат ошибки; **RMSE** — корень из MSE, в единицах целевой переменной.
- **Классификация:** **accuracy** (accuracy_score) — доля верных предсказаний класса.

Один запуск train_test_split и одна метрика дают лишь оценку качества на одной случайной разбивке. Для более устойчивой оценки используют кросс-валидацию ([§12.3](chapter-12-03.md)). Здесь важно закрепить цепочку: подготовка X, y → разбиение → fit → predict → метрика.

---

## Ключевое

- **Регрессия:** LinearRegression, fit/predict; **mean_squared_error(y_test, y_pred)** — MSE; RMSE = sqrt(MSE).
- **Классификация:** LogisticRegression или DecisionTreeClassifier, fit/predict; **accuracy_score(y_test, y_pred)** — доля правильных ответов.
- Цепочка: X, y → train_test_split → model.fit(X_train, y_train) → y_pred = model.predict(X_test) → метрика по y_test и y_pred.

В [§12.1](chapter-12-01.md) обсудим, что покрыто в серии книг по машинному обучению и куда двигаться дальше для углубления.
