# §1.2 Установка

В [§1.1](chapter-01-01.md) мы обсудили, зачем использовать Scala для работы с данными. Теперь нужно подготовить среду: установить JDK (Java Development Kit), выбрать способ установки Scala — через Coursier, sbt или scala-cli — и убедиться, что команды `scala` и `scalac` доступны в командной строке. В этом разделе приведены пошаговые инструкции для основных ОС; в [§1.3](chapter-01-03.md) мы перейдём к инструментам разработки и REPL.

---

## 1.2.1. JDK — prerequisite

Scala компилируется в байт-код **JVM** и выполняется поверх Java. Поэтому перед установкой Scala нужна **JDK** (Java Development Kit): набор инструментов для компиляции и запуска Java-программ, включающий компилятор `javac`, среду выполнения (JRE) и утилиты. См. [Глоссарий](glossary.md).

Рекомендуется использовать **JDK 17** или новее (LTS-версии: 17, 21). Spark и большинство современных библиотек Scala поддерживают эти версии. Для обучения и прохождения книги подойдёт любая JDK 11 и выше; для работы со Spark 3.x обычно требуется JDK 11 или 17.

**Проверка наличия JDK**

В терминале выполните:

```bash
java -version
javac -version
```

Если отображается версия Java и компилятора, JDK установлен. Пример вывода:

```
openjdk version "17.0.9" 2023-10-17
OpenJDK Runtime Environment ...
OpenJDK 64-Bit Server VM ...
```

**Установка JDK при отсутствии**

- **Linux (Debian/Ubuntu):** `sudo apt install openjdk-17-jdk` или `openjdk-21-jdk`
- **Linux (Fedora/RHEL):** `sudo dnf install java-17-openjdk-devel` или `java-21-openjdk-devel`
- **macOS:** через Homebrew — `brew install openjdk@17`; на Apple Silicon можно использовать Azul Zulu или Temurin
- **Windows:** скачать установщик с [Adoptium](https://adoptium.net/) (Eclipse Temurin) или [Oracle JDK](https://www.oracle.com/java/technologies/downloads/)

После установки убедитесь, что `JAVA_HOME` указывает на каталог JDK (для sbt и некоторых инструментов это может быть важно). На Linux/macOS обычно добавляют в `~/.bashrc` или `~/.zshrc`:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64   # путь может отличаться
export PATH=$JAVA_HOME/bin:$PATH
```

---

## 1.2.2. Способы установки Scala

Есть несколько вариантов, как получить Scala на машине:

| Способ       | Что даёт                     | Когда использовать                          |
|--------------|------------------------------|---------------------------------------------|
| **cs setup** | `scala`, `scalac`, `sbt`, `scala-cli`, JDK | Универсальный старт; рекомендуется официально |
| **sbt**      | Сборка проектов, REPL, управление зависимостями | Полноценные проекты, Spark jobs             |
| **scala-cli**| Скрипты, быстрые эксперименты, REPL | Один файл, прототипы, обучение              |

**cs setup** (Coursier) — установщик, рекомендуемый на [scala-lang.org](https://www.scala-lang.org/download/). Он ставит JDK (если нужно), `scala`, `scalac`, `sbt` и `scala-cli` за один проход. Удобно для быстрого старта.

**sbt** (Scala Build Tool) — стандартный инструмент сборки для Scala-проектов. Управляет зависимостями, компилирует код, запускает тесты; используется в продакшене для Spark jobs и библиотек. См. [Глоссарий](glossary.md).

**scala-cli** — утилита для запуска одного файла или скрипта без настройки проекта. Подходит для быстрых экспериментов, REPL и обучения. См. [Глоссарий](glossary.md).

Для прохождения книги достаточно любого из способов; для работы со Spark ([Глава 8](chapter-08-01.md)) потребуется sbt. Ниже — установка через `cs setup` (рекомендуется) и отдельная установка sbt/scala-cli, если вы предпочитаете их устанавливать по отдельности.

---

## 1.2.3. Установка через cs setup (рекомендуется)

**Coursier** — менеджер артефактов и установщик для экосистемы JVM. Команда `cs setup` устанавливает последнюю версию Scala, sbt, scala-cli и при необходимости JDK, добавляя их в `PATH`.

**Linux (x86_64):**

```bash
curl -fL https://github.com/coursier/coursier/releases/latest/download/cs-x86_64-pc-linux.gz | gzip -d > cs
chmod +x cs
./cs setup
```

**Linux (ARM64, например Raspberry Pi):**

```bash
curl -fL https://github.com/VirtusLab/coursier-m1/releases/latest/download/cs-aarch64-pc-linux.gz | gzip -d > cs
chmod +x cs
./cs setup
```

**macOS (через Homebrew):**

```bash
brew install coursier/formulas/coursier
cs setup
```

**macOS (без Homebrew, Apple Silicon):**

```bash
curl -fL https://github.com/coursier/coursier/releases/latest/download/cs-aarch64-apple-darwin.gz | gzip -d > cs
chmod +x cs
(xattr -d com.apple.quarantine cs 2>/dev/null || true)
./cs setup
```

**Windows:** скачайте архив [cs-x86_64-pc-win32.zip](https://github.com/coursier/coursier/releases/latest/download/cs-x86_64-pc-win32.zip), распакуйте и запустите `cs.exe` (или `cs.bat`); следуйте инструкциям в консоли.

Во время `cs setup` утилита спросит, что установить (JDK, scala-cli, sbt и т.д.). Можно принять значения по умолчанию. После завершения **закройте и снова откройте терминал**, чтобы обновился `PATH`.

---

## 1.2.4. Установка sbt отдельно

Если вы не используете `cs setup` и хотите установить только sbt (например, для работы с существующим проектом или Spark):

**Linux (Debian/Ubuntu):**

```bash
echo "deb https://repo.scala-sbt.org/scalasbt/debian all main" | sudo tee /etc/apt/sources.list.d/sbt.list
curl -sL "https://keyserver.ubuntu.com/pks/lookup?op=get&search=0x2EE0EA64E40A89B84B2DF73499E82A75642AC823" | sudo apt-key add
sudo apt update
sudo apt install sbt
```

**macOS (Homebrew):**

```bash
brew install sbt
```

**Универсальный способ (все ОС):** через Coursier, но только sbt:

```bash
curl -fL https://github.com/coursier/coursier/releases/latest/download/cs-x86_64-pc-linux.gz | gzip -d > cs
chmod +x cs
./cs install sbt
```

(Для macOS и Windows — соответствующие бинарники Coursier с [releases](https://github.com/coursier/coursier/releases).)

sbt при первом запуске скачает Scala и зависимости; первый `sbt` может занять несколько минут.

---

## 1.2.5. Установка scala-cli отдельно

**scala-cli** удобен для скриптов и быстрых экспериментов без создания проекта.

**Linux (установка в /usr/local/bin):**

```bash
curl -fL https://github.com/VirtusLab/scala-cli/releases/latest/download/scala-cli-x86_64-pc-linux.gz | gzip -d > scala-cli
chmod +x scala-cli
sudo mv scala-cli /usr/local/bin/scala-cli
```

**macOS (Homebrew):**

```bash
brew install VirtusLab/scala-cli/scala-cli
```

**Универсально через Coursier:**

```bash
cs install scala-cli
```

Проверка: `scala-cli version` должен вывести версию утилиты.

---

## 1.2.6. Проверка scala и scalac

После установки нужно убедиться, что доступны интерпретатор и компилятор Scala.

**scala** — запускает REPL (интерактивную оболочку) или выполняет скомпилированный класс/скрипт. См. [Глоссарий](glossary.md).

**scalac** — компилятор Scala: превращает исходный код (.scala) в байт-код JVM (.class).

**Проверка версии:**

```bash
scala -version
```

Ожидаемый вывод (версия может отличаться):

```
Scala code runner version 3.3.7 -- Copyright 2002-2024, LAMP/EPFL
```

или для Scala 2:

```
Scala code runner version 2.13.12 -- Copyright 2002-2023, LAMP/EPFL
```

**Проверка компилятора:**

```bash
scalac -version
```

**Краткая интерактивная проверка REPL**

Запустите `scala` без аргументов. Должен открыться REPL с приглашением `scala>` (или `scala> `). Введите:

```scala
1 + 1
```

и нажмите Enter. Должно вывестись `val res0: Int = 2`. Выход из REPL — команда `:quit` или сочетание Ctrl+D (Linux/macOS) / Ctrl+Z и Enter (Windows).

Если `scala` или `scalac` не находятся, закройте и снова откройте терминал (или выполните `source ~/.bashrc` / `source ~/.zshrc`), чтобы обновить `PATH`. При установке через `cs setup` путь добавляется автоматически; при ручной установке убедитесь, что каталог с `scala` и `scalac` присутствует в `PATH`.

**Типичные ошибки**

- **`java: command not found`** — JDK не установлен или не в `PATH`; установите JDK и проверьте `java -version`.
- **`scala: command not found`** после установки — обновите сессию терминала; при `cs setup` путь добавляется в конфиг оболочки.
- Разные версии Scala у `scala` и sbt — sbt использует версию, указанную в `build.sbt`; глобальная `scala` может быть другой. Для обучения это обычно не критично.

---

## Ключевое

- Scala выполняется на JVM; нужен JDK 11 или выше (рекомендуется 17 или 21). Проверка: `java -version`, `javac -version`.
- Установить Scala можно через `cs setup` (рекомендуется; ставит JDK, scala, sbt, scala-cli), отдельно sbt или scala-cli.
- `scala` — интерпретатор и запуск REPL; `scalac` — компилятор. Проверка: `scala -version`, `scalac -version`; интерактивно — запуск `scala` и ввод `1 + 1`.
- После установки может потребоваться перезапуск терминала, чтобы обновился `PATH`.

В [§1.3](chapter-01-03.md) мы разберём инструменты разработки: sbt для сборки и зависимостей, IDE (IntelliJ IDEA, VS Code) и использование REPL для экспериментов.
