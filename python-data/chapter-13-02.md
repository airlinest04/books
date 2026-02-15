# §13.2 Keras

В [§13.1](chapter-13-01.md) мы обсудили, когда уместны глубокие нейросети. **Keras** — высокоуровневый API для построения и обучения нейронных сетей; с версии 2.x он встроен в TensorFlow как **tf.keras**. В этом разделе — **последовательная модель** (Sequential), **плотные слои** (Dense), **компиляция** (compile) и **обучение** (fit). Один минимальный пример позволит запустить обучение простой сети. В [§13.3](chapter-13-03.md) кратко сравним TensorFlow и PyTorch.

---

## 13.2.1. Keras как высокоуровневый API

Keras скрывает детали реализации градиентного спуска и обратного распространения ошибки: пользователь задаёт **слои** (тип, число нейронов, функцию активации), **оптимизатор**, **функцию потерь** и **метрики**, затем вызывает **fit()** на данных. Бэкендом по умолчанию служит **TensorFlow**; установка: **pip install tensorflow** (включает Keras).

Импорт: **from tensorflow import keras** или **from tensorflow.keras import Sequential, layers**. Модель собирается из слоёв; для простого линейного стека слоёв удобна **Sequential**.

---

## 13.2.2. Последовательная модель и плотные слои

**Sequential** — контейнер слоёв, через которые данные проходят по порядку (сверху вниз). Слои добавляются методом **.add()**.

**Dense** (плотный, полносвязный слой): каждый нейрон слоя связан со всеми выходами предыдущего слоя. Параметры: **units** — число нейронов в слое, **activation** — функция активации ('relu', 'sigmoid', 'softmax' и др.), **input_shape** — форма входа (только у первого слоя: кортеж размерностей, например (n_features,) для вектора признаков).

Типичная схема для классификации по таблице признаков: первый Dense с relu и input_shape, один или несколько Dense с relu, последний Dense с units=число_классов и activation='softmax' (вероятности классов). Для регрессии последний слой — один нейрон без активации или с 'linear'.

```python
from tensorflow.keras import Sequential
from tensorflow.keras.layers import Dense

model = Sequential([
    Dense(64, activation="relu", input_shape=(n_features,)),
    Dense(32, activation="relu"),
    Dense(n_classes, activation="softmax"),
])
```

---

## 13.2.3. Компиляция и обучение

**model.compile(optimizer, loss, metrics)** настраивает процесс обучения: **optimizer** — алгоритм обновления весов ('adam', 'sgd' и др.), **loss** — функция потерь ('sparse_categorical_crossentropy' для целочисленных меток классов, 'mse' для регрессии), **metrics** — список метрик для вывода при обучении (например, ['accuracy']).

**model.fit(x, y, epochs=..., batch_size=..., validation_split=...)** запускает обучение: **x** — массив признаков (или тензор), **y** — целевые значения (метки классов или числа); **epochs** — число проходов по данным, **batch_size** — размер мини-батча. **validation_split** — доля данных, откладываемая для проверки качества после каждой эпохи (опционально). fit возвращает объект History с историей loss и метрик по эпохам.

Перед fit данные должны быть числовыми массивами (NumPy или тензоры); для классификации с softmax метки — целые (0, 1, …) при sparse_categorical_crossentropy. После обучения **model.predict(x_new)** возвращает предсказания.

```python
model.compile(optimizer="adam", loss="sparse_categorical_crossentropy", metrics=["accuracy"])
model.fit(X_train, y_train, epochs=10, batch_size=32, validation_split=0.1)
# Обучение на X_train, y_train; 10 эпох; 10% данных — валидация
loss, acc = model.evaluate(X_test, y_test)
y_pred = model.predict(X_test)
```

Этот пример — иллюстрация цикла «модель — compile — fit»; для реальных задач потребуются предобработка данных, подбор архитектуры и гиперпараметров, что выходит за рамки книги.

---

## Ключевое

- **Keras** (tf.keras) — высокоуровневый API для нейросетей; входит в TensorFlow; установка — pip install tensorflow.
- **Sequential** — модель из последовательности слоёв; **Dense(units, activation, input_shape)** — плотный слой; первый слой задаёт input_shape.
- **compile(optimizer, loss, metrics)** — выбор оптимизатора, потерь и метрик; **fit(x, y, epochs, batch_size)** — обучение; **predict(x)** — предсказание.

В [§13.3](chapter-13-03.md) кратко рассмотрим TensorFlow и PyTorch: роли, стиль API и когда что выбирать.
