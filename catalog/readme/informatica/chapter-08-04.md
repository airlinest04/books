# §8.4 Mapplets и повторное использование

В [§8.3](chapter-08-03.md) мы рассмотрели Normalizer, Sequence Generator, Stored Procedure и Update Strategy. **Mapplet** — переиспользуемый объект, содержащий группу трансформаций; создаётся в Mapplet Designer и добавляется в маппинги как единый блок. В этом разделе — назначение Mapplet, Input/Output трансформации, правила и ограничения, связь с переиспользуемыми трансформациями. См. [Глоссарий](glossary.md).

---

## 8.4.1. Mapplet: назначение

**Mapplet** — переиспользуемый объект, создаваемый в Mapplet Designer; содержит набор трансформаций и позволяет использовать одну и ту же логику в нескольких маппингах. Источник: [Mapplets Overview](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/mapplets-overview.html).

**Пример:** несколько fact-таблиц требуют серию Lookup для поиска ключей измерений. Вместо дублирования Lookup в каждом маппинге создают Mapplet с этой логикой и подключают его к каждому маппингу.

При добавлении Mapplet в маппинг создаётся **instance** (экземпляр). Изменения в Mapplet наследуются всеми instance. Источник: [Mapplets Overview](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/mapplets-overview.html).

---

## 8.4.2. Input и Output

Mapplet должен иметь вход и выход. Источник: [Understanding Mapplet Input and Output](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/understanding-mapplet-input-and-output.html).

| Компонент | Описание |
|-----------|----------|
| **Mapplet input** | Source definitions и/или **Input** трансформация. Input — пассивная трансформация, определяющая входные порты Mapplet; подключается к потоку данных в маппинге. |
| **Mapplet output** | Одна или несколько **Output** трансформаций. Output — пассивная; каждая Output — одна выходная группа Mapplet. |
| **Mapplet ports** | Отображаются в Mapping Designer: input — из Input, output — из Output. При использовании только Source definitions input-портов в маппинге нет. |

Input и Output — специальные трансформации, доступные только в Mapplet Designer. Источник: [Transformation Descriptions](https://docs.informatica.com/data-integration/powercenter/10-5/transformation-guide/working-with-transformations/transformations-overview/transformation-descriptions.html).

---

## 8.4.3. Варианты входа

**С источником внутри Mapplet:** Mapplet может включать Source definitions и Source Qualifier; данные поступают из источников, определённых в Mapplet. Источник: [Mapplets Overview](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/mapplets-overview.html).

**С Input из маппинга:** Mapplet получает данные из потока маппинга через Input; Input подключается к Source Qualifier или другой трансформации в маппинге. Источник: [Understanding Mapplet Input and Output](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/understanding-mapplet-input-and-output.html).

Можно комбинировать: часть данных — из Source в Mapplet, часть — через Input из маппинга.

---

## 8.4.4. Использование в маппинге

При перетаскивании Mapplet в маппинг Designer создаёт instance. Редактировать Mapplet можно только в Mapplet Designer; изменения наследуются всеми instance. Источник: [Using Mapplets in Mappings](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/using-mapplets-in-mappings.html).

**Шаги:**
1. Перетащить Mapplet в маппинг.
2. Подключить хотя бы один input-порт Mapplet к трансформации в маппинге (если есть input-порты).
3. Подключить хотя бы один output-порт к трансформации в маппинге.

Не все порты обязаны быть подключены; неподключённые порты допустимы. Источник: [Mapplets Overview](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/mapplets-overview.html).

---

## 8.4.5. Правила и ограничения

Источник: [Rules and Guidelines for Mapplets](https://docs.informatica.com/data-integration/powercenter/10-5/designer-guide/mapplets/rules-and-guidelines-for-mapplets.html).

**Обязательно:**
- Минимум одна Input трансформация или Source definition с подключённым портом.
- Минимум одна Output трансформация с подключённым портом.
- Input получает данные от одного активного источника.
- Один порт Input нельзя подключать к нескольким трансформациям в Mapplet.

**Запрещено в Mapplet:**
- Normalizer
- COBOL sources
- XML Source Qualifier, XML sources
- Target definitions
- Pre- и post-session stored procedures
- Другие Mapplets

**Ограничения:**
- Sequence Generator — только reusable.
- Stored Procedure — тип Normal.
- При изменении Mapplet (удаление портов, смена типа passive/active) маппинги могут стать Invalid.
- Не менять datatype, precision, scale портов при использовании Mapplet в маппингах.

---

## 8.4.6. Mapplet vs переиспользуемая трансформация

| Критерий | Mapplet | Reusable transformation |
|----------|---------|---------------------------|
| Содержимое | Группа трансформаций | Одна трансформация |
| Создание | Mapplet Designer | Transformation Developer |
| Вход/выход | Input, Output или Source | Порты трансформации |
| Переиспользование | Instance в маппинге | Instance в маппинге |

Mapplet — для переиспользования **логики** (цепочки трансформаций); reusable transformation — для переиспользования **одной** трансформации. См. [§5.1.5](chapter-05-01.md).

---

## 8.4.7. Типичные ошибки

- **Mapplet без Output:** Mapplet должен содержать хотя бы одну Output с подключённым портом.
- **Input от нескольких активных источников:** Input получает данные только от одного активного источника.
- **Normalizer в Mapplet:** Normalizer в Mapplet использовать нельзя.
- **Редактирование instance в Mapping Designer:** Mapplet редактируется только в Mapplet Designer; в маппинге — только подключение портов.

---

## Ключевое

- **Mapplet** — переиспользуемая группа трансформаций; создаётся в Mapplet Designer; instance в маппинге.
- **Input** — вход Mapplet из потока маппинга; **Output** — выход в маппинг; Input и Output — только в Mapplet.
- **Вход:** Source definitions внутри Mapplet и/или Input из маппинга.
- **Ограничения:** без Normalizer, COBOL, XML, Target; Sequence Generator — только reusable.
- **Mapplet vs reusable transformation:** Mapplet — группа трансформаций; reusable — одна трансформация.

В [§9.1](chapter-09-01.md) мы перейдём к Workflows и выполнению: определение Workflow, связь с маппингами и сессиями.
