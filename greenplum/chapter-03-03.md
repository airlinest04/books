# §3.3 Конфигурация и gpinitsystem

В [§3.2](chapter-03-02.md) мы установили Greenplum на все узлы и настроили GPHOME и PATH. Чтобы кластер заработал, нужно **инициализировать** систему: создать каталоги данных на Master и сегментах и запустить утилиту **gpinitsystem**, которая создаст экземпляры БД на каждом узле и объединит их в один кластер. В этом разделе — **файл конфигурации** для gpinitsystem, **hostfile** со списком хостов сегментов, создание каталогов данных и запуск **gpinitsystem**. Первый запуск кластера и проверка — в [§3.4](chapter-03-04.md). По [Tanzu Greenplum 7 — gpinitsystem](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/utility_guide-ref-gpinitsystem.html), [Tanzu Greenplum 6 — Initializing a Greenplum Database System](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/6/greenplum-database/install_guide-init_gpdb.html).

---

## 3.3.1. Что делает gpinitsystem

**gpinitsystem** — утилита однократной инициализации кластера Greenplum. Она:

- Проверяет доступ по SSH ко всем хостам из hostfile и к каталогам данных.
- Создаёт на Master каталог данных координатора (coordinator/master) и инициализирует в нём экземпляр PostgreSQL (initdb).
- На каждом хосте сегментов создаёт каталоги данных для primary-сегментов (и при включённом mirror — для mirror) и инициализирует в них экземпляры PostgreSQL.
- Заполняет системный каталог (топологию сегментов, порты, хосты) и настраивает взаимодействие узлов.
- В конце успешной инициализации **запускает** кластер (все инстансы поднимаются).

Запуск выполняется **с хоста Master** под пользователем **gpadmin**; перед этим нужно выполнить `source $GPHOME/greenplum_path.sh` и обменять ключи SSH (gpssh-exkeys). После инициализации в профиле gpadmin на Master задают переменную **MASTER_DATA_DIRECTORY** (или COORDINATOR_DATA_DIRECTORY в новых версиях), указывающую на каталог данных Master. См. [Глоссарий](glossary.md).

---

## 3.3.2. Файл конфигурации gpinitsystem

Утилита gpinitsystem читает параметры из **файла конфигурации**. Пример такого файла лежит в `$GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config`. Скопируйте его и отредактируйте под свой кластер. По [Tanzu Greenplum 7 — Initialization Configuration File Format](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/7/greenplum-database/utility_guide-ref-gpinitsystem.html).

Основные параметры (в GP6 часто используются имена MASTER_*, в GP7 — COORDINATOR_*; ниже приведены оба варианта, где различаются):

| Параметр | Назначение |
|----------|------------|
| **COORDINATOR_HOSTNAME** (или MASTER_HOSTNAME) | Имя хоста Master; должно совпадать с выводом `hostname` на этой машине. |
| **COORDINATOR_DIRECTORY** (или MASTER_DIRECTORY) | Каталог на хосте Master, в котором будет создан каталог данных координатора (например, /data/coordinator). |
| **COORDINATOR_PORT** (или MASTER_PORT) | Порт, на котором Master принимает подключения (по умолчанию 5432). |
| **SEG_PREFIX** | Префикс имён каталогов сегментов: gpseg0, gpseg1, …; координатор — gpseg-1. |
| **PORT_BASE** | Базовый порт для primary-сегментов; порты каждого следующего сегмента на хосте увеличиваются на 1. Не должны попадать в диапазон ip_local_port_range (см. sysctl). |
| **declare -a DATA_DIRECTORY=(...)** | Список каталогов на хостах сегментов для данных primary; количество элементов задаёт число primary-сегментов **на один хост** (при одном адресе хоста в hostfile). Можно указывать один путь несколько раз. |
| **TRUSTED_SHELL** | Оболочка для удалённого выполнения (значение **ssh**). |
| **ENCODING** | Кодировка БД (например, UNICODE). |
| **MACHINE_LIST_FILE** | (Опционально.) Полный путь к hostfile; можно не задавать, если передаёте hostfile через опцию -h. |
| **MIRROR_PORT_BASE** | (Опционально.) Базовый порт для mirror-сегментов. |
| **declare -a MIRROR_DATA_DIRECTORY=(...)** | (Опционально.) Каталоги для mirror; число элементов должно совпадать с числом в DATA_DIRECTORY. |

Каталоги в COORDINATOR_DIRECTORY и DATA_DIRECTORY должны существовать на соответствующих хостах и быть доступны на запись пользователю gpadmin; gpinitsystem создаёт внутри них подкаталоги (gpseg-1, gpseg0, …). См. подраздел 3.3.4.

Пример фрагмента конфига (GP6-стиль имён):

```bash
SEG_PREFIX=gpseg
PORT_BASE=6000
declare -a DATA_DIRECTORY=(/data1/primary /data2/primary)
MASTER_HOSTNAME=mdw
MASTER_DIRECTORY=/data/master
MASTER_PORT=5432
TRUSTED_SHELL=ssh
ENCODING=UNICODE
```

Для mirror раскомментируйте и задайте MIRROR_PORT_BASE и MIRROR_DATA_DIRECTORY. Точный список параметров и имена (COORDINATOR_* vs MASTER_*) см. в документации вашей версии Greenplum.

---

## 3.3.3. Hostfile для gpinitsystem

**Hostfile** (файл со списком хостов) передаётся в gpinitsystem опцией **-h** или задаётся параметром **MACHINE_LIST_FILE** в конфиге. В нём перечисляются **только хосты сегментов** (не Master и не Standby): по одному имени или адресу на строку. См. [Глоссарий](glossary.md), [§3.1](chapter-03-01.md).

Правила:

- Строка — один хост или один сетевой интерфейс (если у одного физического хоста несколько интерфейсов и вы хотите распределить сегменты по ним). Без пустых строк и лишних пробелов.
- Количество primary-сегментов на хост определяется комбинацией числа **строк для этого хоста** в hostfile и числа **элементов в DATA_DIRECTORY**: сегменты распределяются по парам (хост, каталог). Например, два хоста по одной строке и два каталога в DATA_DIRECTORY дадут по два сегмента на каждом хосте (всего четыре primary-сегмента).
- Имена должны резолвиться с Master (через /etc/hosts или DNS); рекомендуется использовать имена и локальный /etc/hosts. По документации Tanzu Greenplum.

Пример hostfile для трёх хостов сегментов (по одному интерфейсу):

```text
sdw1
sdw2
sdw3
```

Файл сохраняют, например, как `~/gpconfigs/hostfile_gpinitsystem` и передают в команду: `gpinitsystem -c gpinitsystem_config -h hostfile_gpinitsystem`.

---

## 3.3.4. Создание каталогов данных

Перед запуском gpinitsystem на **каждом** хосте нужно подготовить каталоги, в которые будут записаны данные:

- **На Master:** каталог, указанный в COORDINATOR_DIRECTORY (или MASTER_DIRECTORY). Внутри него gpinitsystem создаст подкаталог с именем вида SEG_PREFIX + «-1» (например, gpseg-1).
- **На каждом хосте сегментов:** каталоги из списка DATA_DIRECTORY (например, /data1/primary, /data2/primary). Они должны существовать и принадлежать gpadmin (или быть доступны на запись). Внутри gpinitsystem создаст gpseg0, gpseg1, … .
- **При использовании mirror:** каталоги из MIRROR_DATA_DIRECTORY на соответствующих хостах (mirror другого content id должны быть на другом хосте).

Пример (выполнить на каждом хосте или через gpssh с Master):

```bash
# На Master
sudo mkdir -p /data/master
sudo chown gpadmin:gpadmin /data/master

# На каждом хосте сегментов (sdw1, sdw2, ...)
sudo mkdir -p /data1/primary /data2/primary
sudo chown gpadmin:gpadmin /data1/primary /data2/primary
```

При нескольких хостах удобно использовать **gpssh** с общим hostfile, чтобы выполнить mkdir и chown на всех узлах одной командой. По [Tanzu Greenplum 6 — Creating the Greenplum Database Configuration File](https://techdocs.broadcom.com/us/en/vmware-tanzu/data-solutions/tanzu-greenplum/6/greenplum-database/install_guide-init_gpdb.html).

---

## 3.3.5. Запуск gpinitsystem

Под пользователем **gpadmin** на хосте Master выполните:

```bash
source /usr/local/greenplum-db/greenplum_path.sh   # или ваш GPHOME
gpinitsystem -c gpconfigs/gpinitsystem_config -h gpconfigs/hostfile_gpinitsystem
```

Пути к конфигу и hostfile — те, куда вы их сохранили. Утилита проверит конфигурацию, доступ по SSH и к каталогам, выведет сводку и запросит подтверждение (**Continue with Greenplum creation? Yy/Nn**). После ответа `y` начнётся инициализация Master и всех сегментов (параллельно); в конце при успехе кластер будет **запущен** и появится сообщение вроде «Greenplum Database instance successfully created». Среда: RHEL/CentOS/Rocky, пользователь gpadmin. По документации Tanzu Greenplum.

Полезные опции:

- **-a** — не запрашивать подтверждение (для скриптов).
- **-s standby_hostname** — настроить Standby Master на указанном хосте; при необходимости **-S standby_datadir** и **-P standby_port**.
- **--mirror-mode=spread** (или **group**) — способ размещения mirror (spread — по разным хостам; см. документацию).
- **-e password** (или **--su_password=password**) — пароль суперпользователя БД (по умолчанию может быть задан в конфиге или использоваться значение по умолчанию; в продакшене пароль обязательно сменить).
- **-O output_file** — записать итоговую конфигурацию кластера в файл (для последующего использования с -I при пересоздании кластера).

Логи gpinitsystem пишет в **~/gpAdminLogs**. При сбое может быть создан скрипт отката (backout_gpinitsystem_...) для очистки частично созданных каталогов и процессов. См. документацию по troubleshooting.

---

## 3.3.6. Установка MASTER_DATA_DIRECTORY после инициализации

После успешного gpinitsystem каталог данных Master будет, например, **/data/master/gpseg-1** (если MASTER_DIRECTORY=/data/master и SEG_PREFIX=gpseg). Этот путь нужно прописать в окружении пользователя gpadmin на **хосте Master** (и при наличии Standby — на Standby), чтобы утилиты gpstart, gpstop, psql и др. знали, к какому кластеру обращаться.

Добавьте в **~/.bashrc** пользователя gpadmin на Master:

```bash
source /usr/local/greenplum-db/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/data/master/gpseg-1
```

Путь подставьте тот, что реально создан (он выводится в конце gpinitsystem). Затем выполните `source ~/.bashrc` или войдите в сессию заново. См. [§3.4](chapter-03-04.md).

---

## 3.3.7. Типичные ошибки

- **Включить в hostfile Master или Standby:** в hostfile только хосты **сегментов**; Master — хост, с которого запускается gpinitsystem, его имя задаётся в конфиге (COORDINATOR_HOSTNAME / MASTER_HOSTNAME).
- **Не создать каталоги или не выдать права gpadmin:** gpinitsystem завершится с ошибкой доступа; создайте каталоги и chown на всех узлах заранее.
- **Оставить пароль суперпользователя по умолчанию в продакшене:** задайте надёжный пароль через -e или сразу после инициализации смените его (ALTER ROLE).
- **Перепутать порты с ip_local_port_range:** PORT_BASE и MIRROR_PORT_BASE не должны попадать в диапазон временных портов ОС (проверьте sysctl net.ipv4.ip_local_port_range).

---

## Ключевое

- **gpinitsystem** инициализирует кластер: создаёт каталоги данных координатора и сегментов, инициализирует в них экземпляры PostgreSQL, настраивает топологию и в конце запускает кластер. Запускается с **Master** под **gpadmin** после source greenplum_path.sh и настройки SSH.
- **Файл конфигурации** задаёт COORDINATOR_* (или MASTER_*), DATA_DIRECTORY, PORT_BASE, SEG_PREFIX, ENCODING и при необходимости mirror; пример — в $GPHOME/docs/cli_help/gpconfigs/gpinitsystem_config.
- **Hostfile** (-h или MACHINE_LIST_FILE) содержит только хосты сегментов, по одному адресу/имени на строку; число сегментов на хост связано с числом строк для хоста и числом элементов в DATA_DIRECTORY.
- **Каталоги данных** на Master и на каждом хосте сегментов создают заранее (mkdir) и передают владение **gpadmin**; gpinitsystem создаёт внутри них подкаталоги (gpseg-1, gpseg0, …).
- После успешной инициализации в ~/.bashrc пользователя gpadmin на Master задают **MASTER_DATA_DIRECTORY** (путь к каталогу данных Master, например /data/master/gpseg-1).

В [§3.4](chapter-03-04.md) мы разберём первый запуск (gpstart), подключение через psql, проверку gp_segment_configuration и простой тестовый запрос.
