# §2.2 Папки и объекты

В [§2.1](chapter-02-01.md) мы разобрали Repository Server, домены и репозитории. Все объекты репозитория (Sources, Targets, Mappings, Workflows и др.) хранятся внутри **папок** (Folders). В этом разделе мы рассмотрим организацию объектов в папках, свойства папок, типы объектов по приложениям (Designer, Workflow Manager, Repository Manager), shared folders и shortcuts, а также права доступа (Read, Write, Execute). Вы научитесь создавать папки, назначать права и понимать, какие объекты где создаются. Подробнее Connections и переменные — в [§2.3](chapter-02-03.md). См. [Глоссарий](glossary.md).

---

## 2.2.1. Папки: организация метаданных

**Папка** (Folder) — контейнер для организации метаданных в репозитории. Все объекты (Sources, Targets, Mappings, Workflows, Sessions, Connections) принадлежат папке; без папки объект создать нельзя. Папки помогают логически структурировать репозиторий: по проектам, окружениям, доменам данных. Источник: [Folders Overview](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/folders/folders-overview.html).

Типичная структура:

```
Repository
  ├── DEV
  │   ├── Sources
  │   ├── Targets
  │   ├── Mappings
  │   └── Workflows
  ├── TEST
  │   └── ...
  └── PROD
      └── ...
```

Или по проектам:

```
Repository
  ├── Project_Sales
  ├── Project_HR
  └── Project_Finance
```

Папки создаются в **Repository Manager**; при создании объекта в Designer или Workflow Manager разработчик выбирает папку, в которую сохранить объект. Содержимое папок можно сравнивать (compare folders), копировать между папками и репозиториями. См. [§2.4](chapter-02-04.md).

---

## 2.2.2. Свойства папки

При создании папки настраиваются свойства. Источник: [Managing Folder Properties](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/folders/managing-folder-properties.html).

| Свойство | Обязательное | Описание |
|----------|--------------|----------|
| **Name** | Да | Имя папки. Не использовать точку (.) в имени — может вызывать ошибки при выполнении сессий. |
| **Description** | Нет | Описание папки; отображается в Repository Manager. |
| **Owner** | — | Владелец папки; по умолчанию — пользователь, создавший папку. Только для чтения при создании; меняется на вкладке Permissions. |
| **OS Profile** | Нет | Имя профиля ОС; если Integration Service использует OS profiles, указывается профиль для выполнения сессий в этой папке. |
| **Allow Shortcut** | Нет | Разрешает создание shortcuts; делает папку shared (общей) для использования в других репозиториях Repository Domain. |
| **Status** | Условно | Статус объектов в папке; требуется для версионированных репозиториев. |

Папки бывают **shared** (общие) и **non-shared** (необщие). Shared folder с Allow Shortcut может быть доступна через shortcuts из других папок и репозиториев в рамках Repository Domain.

---

## 2.2.3. Объекты Designer: Sources, Targets, Mappings

Объекты, создаваемые в **Designer**, хранятся в папках. Источник: [Objects Created in the Designer](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata/objects-created-in-the-designer.html).

| Объект | Инструмент | Описание |
|--------|------------|----------|
| **Source definitions** | Source Analyzer | Описание структуры источника: таблицы БД, представления, синонимы, плоские файлы, XML, COBOL. Имена колонок, типы, ограничения. |
| **Target definitions** | Target Designer | Описание структуры приёмника: таблицы БД, файлы, XML. |
| **Transformations** | Transformation Developer, Mapping Designer | Трансформации: порты, выражения, условия. Встроенные (Expression, Filter, Lookup) и пользовательские. |
| **Reusable transformations** | Transformation Developer | Переиспользуемые трансформации для нескольких маппингов в папке, репозитории или Repository Domain. |
| **Mappings** | Mapping Designer | Маппинг: источники, трансформации, приёмники; связи между портами. |
| **Mapplets** | Mapplet Designer | Набор трансформаций для переиспользования в нескольких маппингах. |
| **User-defined functions** | — | Пользовательские функции на языке трансформаций Informatica. |
| **Shortcuts** | Designer | Ссылки на объекты из shared folders (локальные или глобальные в Repository Domain). |

Все эти объекты при сохранении помещаются в выбранную папку. Mapping ссылается на Sources и Targets в той же папке (или через shortcuts). См. [главы 3–4](chapter-03-01.md), [главы 5–8](chapter-05-01.md).

---

## 2.2.4. Объекты Workflow Manager: Sessions, Workflows, Connections

Объекты, создаваемые в **Workflow Manager**, также хранятся в папках. Источник: [Objects Created in the Workflow Manager](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/understanding-the-repository/understanding-metadata/objects-created-in-the-workflow-manager.html).

| Объект | Описание |
|--------|----------|
| **Database connections** | Параметры подключения к БД источников и приёмников; используется Integration Service при выполнении сессий. |
| **Sessions** | Задачи workflow; содержат ссылку на Mapping и настройки (connections, параметры, партиционирование). |
| **Workflows** | Набор инструкций (задач): Session, Command, Decision, Timer, Email и др.; порядок выполнения. |
| **Workflow tasks** | Отдельные задачи: Command (выполнение команды ОС), Decision (условная ветвление), Timer (задержка), Email (уведомление). |
| **Worklets** | Переиспользуемая группа задач workflow; вкладываются в workflow и другие worklets. |

Session связывается с Mapping из той же папки (или из другой через ссылку). Workflow содержит Session и другие задачи. Connections могут быть глобальными (доступны во всех папках репозитория) или локальными для папки. Подробнее Connections — в [§2.3](chapter-02-03.md), Workflows — в [главе 9](chapter-09-01.md).

---

## 2.2.5. Объекты Repository Manager: папки и организация

В **Repository Manager** создают и управляют папками, перемещают объекты между папками, настраивают права, выполняют экспорт и импорт. Repository Manager не создаёт Sources, Mappings или Workflows — только структуру (папки) и права.

Дополнительные объекты, которые могут создаваться или управляться через Repository Manager:

- **Object queries** — сохранённые запросы к объектам репозитория.
- **Deployment groups** — группы объектов для развёртывания.
- **Labels** — метки версий (при version control).

Эти объекты относятся к **глобальным** (global objects): они не привязаны к одной папке, но на них распространяются права доступа. См. [§2.4](chapter-02-04.md).

---

## 2.2.6. Shared folders и shortcuts

**Shared folder** — папка с включённым Allow Shortcut; её объекты можно использовать в других папках и репозиториях через **shortcuts** (ярлыки).

**Shortcut** — ссылка на объект из shared folder. Различают:

- **Local shortcut** — ссылка на объект из shared folder в том же репозитории.
- **Global shortcut** — ссылка на объект из shared folder в global repository в рамках Repository Domain.

Shortcuts позволяют переиспользовать Sources, Targets, Mapplets, трансформации без копирования: изменение в исходном объекте отражается во всех маппингах, использующих shortcut. Это упрощает поддержку при изменении структуры источника или общей логики.

---

## 2.2.7. Права доступа: Read, Write, Execute

**Permissions** (права доступа) определяют уровень доступа пользователя или группы к объекту (папке или глобальному объекту). Источник: [Managing Object Permissions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/managing-object-permissions/managing-object-permissions-overview.html).

| Право | Для папки | Для глобальных объектов |
|-------|-----------|--------------------------|
| **Read** | Просмотр папки и её объектов | Просмотр object queries, labels, deployment groups, connection objects |
| **Write** | Создание и редактирование объектов в папке | Поддержка object queries, labels; добавление/удаление в deployment groups |
| **Execute** | Запуск и планирование workflow в папке | Выполнение object queries, применение labels, копирование deployment groups |

Права работают совместно с **privileges** (привилегиями) — действиями, которые пользователь может выполнять в приложениях PowerCenter. Привилегия даёт право на действие в принципе; permission — право на конкретный объект. Например, привилегия «create folder» позволяет создавать папки; но создать объект в папке можно только при наличии Write permission на эту папку.

---

## 2.2.8. Владелец и группа по умолчанию

При создании папки или глобального объекта автоматически назначаются:

- **Owner** (владелец) — пользователь, создавший объект. Владелец получает Read, Write и Execute; эти права **нельзя изменить**. Владельца можно передать другому пользователю.
- **Default group** («Others») — группа по умолчанию; отображается как «Others». По умолчанию имеет только Read. Write и Execute можно добавить. Права default group задают **минимальный уровень** для всех пользователей и групп, добавляемых к объекту.

При добавлении пользователя или группы к списку прав объекта им назначаются права не ниже, чем у default group. Настройка прав выполняется на вкладке Permissions в свойствах папки или глобального объекта. Источник: [Managing Object Permissions Overview](https://docs.informatica.com/data-integration/powercenter/10-5/repository-guide/managing-object-permissions/managing-object-permissions-overview.html).

---

## 2.2.9. Глобальные объекты и их права

**Глобальные объекты** (global objects) не привязаны к одной папке:

- Object queries
- Deployment groups
- Labels
- Connection objects (при определении как глобальные)

На них назначаются права так же, как на папки: Read, Write, Execute. Connection objects часто делают глобальными, чтобы один connection использовался в сессиях из разных папок. См. [§2.3](chapter-02-03.md).

---

## 2.2.10. Создание папки: пошагово

1. Открыть **Repository Manager**, подключиться к репозиторию.
2. В дереве репозитория выбрать корень или родительскую папку.
3. Меню **Folder → Create** (или контекстное меню).
4. Заполнить свойства: **Name** (обязательно), Description, при необходимости Allow Shortcut, OS Profile.
5. На вкладке **Permissions** проверить владельца и права для групп/пользователей.
6. Подтвердить создание.

После создания папка отображается в дереве; в неё можно сохранять объекты из Designer и Workflow Manager. Задание из TOC: создать папку — выполняется через эти шаги.

---

## 2.2.11. Типичные ошибки при работе с папками

- **Использовать точку в имени папки:** может вызывать ошибки при выполнении сессий; избегать.
- **Путать папку и репозиторий:** папка — контейнер внутри репозитория; репозиторий — вся БД метаданных.
- **Забывать про права:** пользователь без Write не сможет создавать и редактировать объекты; без Execute — запускать workflow. Проверять права при проблемах доступа.
- **Создавать объекты не в той папке:** при сохранении в Designer/Workflow Manager выбирать корректную папку; перемещение возможно через Repository Manager, но может потребовать обновления ссылок.
- **Игнорировать shared folders при переиспользовании:** если Sources или Mapplets нужны в нескольких проектах — использовать shared folder и shortcuts вместо копирования.

---

## Ключевое

- **Папка** — контейнер для объектов репозитория; все Sources, Targets, Mappings, Workflows, Sessions хранятся в папках. Создаётся в Repository Manager.
- **Свойства папки:** Name (без точки), Description, Owner, OS Profile, Allow Shortcut (для shared), Status (для versioned).
- **Объекты Designer:** Source, Target, Transformation, Mapping, Mapplet, shortcuts. **Объекты Workflow Manager:** Connection, Session, Workflow, Worklet.
- **Shared folder + shortcuts** — переиспользование объектов между папками и репозиториями без копирования.
- **Права:** Read (просмотр), Write (создание/редактирование), Execute (запуск workflow). Owner имеет все права без возможности изменения. Default group («Others») задаёт минимальный уровень.
- **Глобальные объекты:** object queries, deployment groups, labels, connection objects; права назначаются отдельно.

В [§2.3](chapter-02-03.md) мы разберём Connections (подключения к БД и файлам) и переменные сессии (параметры, переменные).
