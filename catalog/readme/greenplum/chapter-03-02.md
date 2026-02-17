# §3.2 Установка Greenplum

В [§3.1](chapter-03-01.md) мы перечислили требования к окружению. Здесь — **установка** программного обеспечения Greenplum на все узлы кластера: откуда взять дистрибутив, как установить его (RPM или распаковка бинарников) и как настроить переменные окружения (GPHOME, PATH). Инициализация кластера (создание каталогов данных и запуск gpinitsystem) — в [§3.3](chapter-03-03.md). По [Tanzu Greenplum 7 — Installing the Greenplum Database Software](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/install_guide-install_gpdb.html).

---

## 3.2.1. Откуда взять дистрибутив

В зависимости от выбранного варианта использования Greenplum дистрибутив берут из разных источников:

- **VMware Tanzu Greenplum (Broadcom)** — готовые RPM-пакеты для RHEL/Oracle Linux/Rocky Linux доступны на портале поддержки Broadcom (ранее VMware). Требуется учётная запись и при необходимости подписка. См. [Greenplum by VMware](https://support.broadcom.com/group/ecx/productdownloads?subfamily=VMware%20Tanzu%20Greenplum%C2%AE).
- **Открытая сборка (open source)** — исходный код на GitHub (репозиторий greenplum-db/gpdb); сборка из исходников или готовые пакеты от сообщества (если есть для вашей ОС). Версии и наличие бинарников уточняйте на сайте greenplum.org и в репозиториях проекта.
- **Arenadata DB** — российский дистрибутив на базе Greenplum; дистрибутив и документация по установке — на сайте поставщика.

В этой книге примеры установки ориентированы на установку из **RPM** на RHEL/CentOS/Rocky/Oracle Linux; для других дистрибутивов или сборки из исходников шаги будут отличаться, но общая последовательность (установка на каждый хост, GPHOME, gpadmin, инициализация) сохраняется. См. [Глоссарий](glossary.md).

---

## 3.2.2. Установка на каждый хост

Программное обеспечение Greenplum нужно установить **на каждый хост** будущего кластера: на Master (coordinator), на Standby Master (если есть) и на **все хосты сегментов**. Бинарники на всех узлах должны быть одной и той же версии; иначе инициализация и работа кластера могут завершаться ошибками. По [Tanzu Greenplum 7 — Installing the Greenplum Database Software](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/install_guide-install_gpdb.html).

Типичный порядок:

1. Скачать пакет (RPM или архив) и скопировать его в домашний каталог пользователя gpadmin на **каждом** хосте (или на общий каталог, доступный с каждого узла).
2. На каждом хосте выполнить установку (от root или через sudo).
3. Назначить владельца установленных файлов — пользователь **gpadmin** (и группа gpadmin), чтобы процессы Greenplum могли читать и выполнять файлы без прав root.

Детали команд для RPM — в следующих подразделах.

---

## 3.2.3. Установка из RPM (RHEL, CentOS, Rocky, Oracle Linux)

На системах с **yum** (или dnf) установка из RPM выполняется одной командой. Имя пакета имеет вид `greenplum-db-<версия>-rhel<мажор>-x86_64.rpm` (для Oracle Linux и Rocky часто используют пакеты с меткой rhel8). Требуются права root или sudo.

**Установка в каталог по умолчанию:**

```bash
sudo yum install ./greenplum-db-<версия>-rhel8-x86_64.rpm
```

Команда `yum install` подтягивает зависимости и распаковывает пакет. По умолчанию файлы попадают в каталог **/usr/local/greenplum-db-<версия>** (например, `/usr/local/greenplum-db-7.0.0`), и создаётся символическая ссылка **/usr/local/greenplum-db**, указывающая на эту версию. При обновлении на новую минорную версию ссылка переключается на новый каталог. Среда: RHEL/Oracle Linux/Rocky, Greenplum 6/7. По документации Tanzu Greenplum.

**Назначение владельца:**

После установки на **каждом** хосте нужно передать права на каталоги установки пользователю gpadmin:

```bash
sudo chown -R gpadmin:gpadmin /usr/local/greenplum*
```

Без этого пользователь gpadmin не сможет корректно запускать утилиты и процессы Greenplum. Повторите установку и `chown` на Master, Standby и на всех хостах сегментов.

---

## 3.2.4. Установка в нестандартный каталог (опционально)

Если нужно установить Greenplum не в `/usr/local`, а в другой каталог (например, `/opt/greenplum`), можно использовать **rpm** с опцией **--prefix**:

```bash
sudo rpm --install ./greenplum-db-<версия>-rhel8-x86_64.rpm --prefix=/opt
```

В этом случае файлы окажутся в **/opt/greenplum-db-<версия>**, ссылка — **/opt/greenplum-db**. Зависимости пакетов при установке через `rpm` не ставятся автоматически; их нужно установить вручную (`yum install apr apr-util bash bzip2 ...` — полный список см. в документации). Далее на всех хостах:

```bash
sudo chown -R gpadmin:gpadmin /opt/greenplum*
```

Во всех последующих командах и в переменных окружения вместо `/usr/local` используют выбранный префикс (например, `/opt`). Документация и примеры в руководствах обычно предполагают установку в `/usr/local`; при нестандартном пути подставляйте свой каталог. По [Tanzu Greenplum 7 — Installing to a Non-Default Directory](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/install_guide-install_gpdb.html).

---

## 3.2.5. Переменные окружения: GPHOME и PATH

После установки нужно задать переменные окружения Greenplum. В каталоге установки лежит скрипт **greenplum_path.sh**, который выставляет **GPHOME** (путь к каталогу установки), **PATH** (чтобы в PATH попали bin и lib), а также при необходимости другие переменные (LD_LIBRARY_PATH и т.д.). См. [Глоссарий](glossary.md).

Под пользователем **gpadmin** выполните (для установки по умолчанию):

```bash
source /usr/local/greenplum-db/greenplum_path.sh
```

Если установка в нестандартный каталог (например, /opt):

```bash
source /opt/greenplum-db/greenplum_path.sh
```

После этого в текущей сессии будут доступны команды `gpssh`, `gpscp`, `gpinitsystem`, `psql` и др. Чтобы переменные подхватывались при каждом входе под gpadmin, добавьте вызов **source** в файл профиля оболочки (например, `~/.bashrc`):

```bash
echo 'source /usr/local/greenplum-db/greenplum_path.sh' >> ~/.bashrc
```

**GPHOME** — корневой каталог установки Greenplum (то, что задаётся в greenplum_path.sh). Утилиты и скрипты обращаются к нему для поиска бинарников и конфигураций. Переменная **MASTER_DATA_DIRECTORY** (каталог данных Master) задаётся позже, при инициализации кластера, и тоже добавляется в профиль gpadmin на хосте Master. См. [§3.3](chapter-03-03.md), [§3.4](chapter-03-04.md).

---

## 3.2.6. Установка из DEB и распаковка архива

Часть дистрибутивов (в том числе некоторые сборки для Debian/Ubuntu или от вендоров) поставляется в виде **DEB-**пакетов. Установка тогда выполняется через `dpkg` или `apt`:

```bash
sudo dpkg -i greenplum-db-<версия>-debian-amd64.deb
```

Путь установки и наличие скрипта greenplum_path.sh зависят от пакета; см. документацию к конкретному дистрибутиву.

При установке из **архива** (tar.gz) достаточно распаковать его в выбранный каталог на каждом хосте и выполнить `chown -R gpadmin:gpadmin <каталог>`. Путь к **greenplum_path.sh** будет внутри распакованного дерева (часто в корне или в подкаталоге bin). Далее так же подключайте его через `source` в сессии и в ~/.bashrc. См. инструкции сборки/установки для открытого Greenplum или Arenadata DB, если используете их.

---

## 3.2.7. Проверка установки на всех узлах

После установки на все хосты и настройки SSH (см. [§3.1](chapter-03-01.md)) проверьте, что на каждом узле установлена одна и та же версия и права принадлежат gpadmin. С хоста Master под пользователем gpadmin (после `source greenplum_path.sh`) выполните:

```bash
gpssh -f hostfile_exkeys -e 'ls -l /usr/local/greenplum-db'
```

В **hostfile_exkeys** перечислены все хосты кластера (Master, Standby, сегменты); при нестандартном пути замените `/usr/local/greenplum-db` на ваш GPHOME. Команда выполнится на всех узлах без запроса пароля (если SSH без пароля настроен). Убедитесь, что вывод на всех хостах показывает один и тот же каталог и владельца gpadmin. Если запрашивается пароль — повторите обмен ключами (gpssh-exkeys -f hostfile_exkeys). По [Tanzu Greenplum 7 — Confirming Your Installation](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/install_guide-install_gpdb.html).

---

## 3.2.8. Структура каталога установки

После установки в каталоге GPHOME (например, /usr/local/greenplum-db) находятся:

| Элемент | Назначение |
|--------|------------|
| **bin/** | Исполняемые файлы: psql, gpstart, gpstop, gpinitsystem, gpssh, gpscp и др. |
| **lib/** | Библиотеки Greenplum и PostgreSQL |
| **share/** | Общие файлы, шаблоны |
| **sbin/** | Вспомогательные скрипты |
| **docs/cli_help/gpconfigs/** | Примеры конфигурации для gpinitsystem и hostfile |
| **greenplum_path.sh** | Скрипт для установки GPHOME, PATH и других переменных |

Примеры конфигов из docs/cli_help/gpconfigs/ можно скопировать и отредактировать для [§3.3](chapter-03-03.md) (gpinitsystem). См. документацию «About Your Greenplum Database Installation».

---

## 3.2.9. Типичные ошибки

- **Установить только на Master:** Greenplum должен стоять на **всех** узлах (Master, Standby, каждый хост сегментов); иначе gpinitsystem или gpstart не смогут запустить процессы на удалённых хостах.
- **Забыть chown gpadmin:** без прав владельца gpadmin процессы не смогут читать конфиги и писать в каталоги данных; после установки всегда выполняйте chown на каждом хосте.
- **Не подключать greenplum_path.sh:** без source greenplum_path.sh в сессии (или в .bashrc) команды gpssh, gpinitsystem и т.д. не найдутся в PATH; при ошибке «command not found» проверьте, что GPHOME и PATH заданы.

---

## Ключевое

- **Дистрибутив** берут с портала Broadcom (Tanzu), из репозиториев открытого Greenplum или у поставщика (Arenadata DB и др.); для RHEL/CentOS/Rocky/Oracle — обычно RPM.
- Установка выполняется **на каждый хост** кластера (Master, Standby, все хосты сегментов); версия на всех узлах должна совпадать.
- **RPM:** `yum install ./greenplum-db-<версия>-rhel8-x86_64.rpm`; по умолчанию каталог **/usr/local/greenplum-db-<версия>** и ссылка **/usr/local/greenplum-db**; затем `chown -R gpadmin:gpadmin /usr/local/greenplum*`.
- **Переменные окружения:** скрипт **greenplum_path.sh** в каталоге установки задаёт **GPHOME** и **PATH**; выполнять `source .../greenplum_path.sh` в сессии и добавить в ~/.bashrc пользователя gpadmin.
- Проверка: **gpssh -f hostfile_exkeys -e 'ls -l ...'** по всем хостам; один и тот же каталог и владелец gpadmin.

В [§3.3](chapter-03-03.md) мы разберём конфигурацию и **gpinitsystem**: файл конфигурации, hostfile, создание каталогов данных и инициализацию кластера.
