# Практическое задание

# Установка PostgreSQL 18.6 из исходного кода в Ubuntu и настройка запуска сервера

## Цель

Освоить полный процесс подготовки Unix-системы для установки PostgreSQL из исходного кода.

<https://www.postgresql.org/docs/18/installation.html>

В ходе выполнения задания необходимо научиться подготавливать сервер Ubuntu, устанавливать зависимости для компиляции, получать исходный код PostgreSQL, выполнять настройку сборки, компилировать сервер, устанавливать PostgreSQL в отдельный каталог, создавать пользователя операционной системы для работы базы данных, инициализировать кластер базы данных и управлять процессом PostgreSQL.

В рамках задания необходимо использовать PostgreSQL версии **18.6**.

Выполнение производится на Ubuntu-сервере через SSH. Все действия выполняются командами в терминале. Создание отдельных `.sh`-скриптов не требуется.

При сборке PostgreSQL из исходного кода необходимо учитывать, что процесс конфигурации использует дополнительные библиотеки, включая ICU. Поэтому в зависимостях должны присутствовать `libicu-dev` и `pkg-config`, которые необходимы для корректного обнаружения библиотек при выполнении `configure`. ([pgEdge Документация][1])

# Задание

## Шаг 1. Подготовка операционной системы

Подключиться к Ubuntu-серверу по SSH.

Проверить информацию о системе:

```bash
uname -a
```

Проверить версию Ubuntu:

```bash
lsb_release -a
```

Обновить информацию о доступных пакетах:

```bash
sudo apt update
```

Установить обновления:

```bash
sudo apt upgrade -y
```

# Шаг 2. Установка всех необходимых зависимостей

Установить необходимые пакеты одной командой:

```bash
sudo apt update && sudo apt install -y \
    build-essential \
    autoconf \
    pkg-config \
    flex \
    bison \
    perl \
    libperl-dev \
    python3 \
    python3-dev \
    tcl \
    tcl-dev \
    libreadline-dev \
    libedit-dev \
    zlib1g-dev \
    libicu-dev \
    libssl-dev \
    libxml2-dev \
    libxslt1-dev \
    gettext \
    libkrb5-dev \
    libldap-dev \
    libpam0g-dev \
    libsystemd-dev \
    libcurl4-openssl-dev \
    liblz4-dev \
    libzstd-dev \
    uuid-dev \
    libossp-uuid-dev \
    libselinux1-dev \
    libnuma-dev \
    liburing-dev \
    llvm-dev \
    clang \
    meson \
    ninja-build \
    libipc-run-perl \
    wget \
    curl \
    tar \
    gzip \
    bzip2 \
    ca-certificates
```

После установки проверить наличие компилятора:

```bash
gcc --version
```

Проверить наличие make:

```bash
make --version
```

Проверить наличие pkg-config:

```bash
pkg-config --version
```

# Шаг 3. Создание пользователя PostgreSQL

Создать отдельного системного пользователя для PostgreSQL.

Создать пользователя:

```bash
sudo useradd -m postgres
```

Проверить наличие пользователя:

```bash
cat /etc/passwd | grep postgres
```

Переключиться на пользователя PostgreSQL:

```bash
sudo -i -u postgres
```

Проверить текущего пользователя:

```bash
whoami
```

Ожидаемый результат:

```text
postgres
```

# Шаг 4. Подготовка каталога установки

Создать каталог, куда будет установлен PostgreSQL:

```bash
sudo mkdir -p /opt/postgresql
```

Создать каталог для хранения данных:

```bash
sudo mkdir -p /opt/postgresql/data
```

Назначить владельцем пользователя PostgreSQL:

```bash
sudo chown -R postgres:postgres /opt/postgresql
```

Проверить права:

```bash
ls -la /opt
```

# Шаг 5. Получение исходного кода PostgreSQL 18.6

Перейти в каталог временных файлов:

```bash
cd /tmp
```

Скачать исходный код PostgreSQL 18.6:

```bash
wget https://ftp.postgresql.org/pub/source/v18.6/postgresql-18.6.tar.gz
```

Проверить наличие архива:

```bash
ls -lh postgresql-18.6.tar.gz
```

# Шаг 6. Распаковка исходного кода

Распаковать архив:

```bash
tar -xzf postgresql-18.6.tar.gz
```

Перейти в каталог PostgreSQL:

```bash
cd postgresql-18.6
```

Проверить содержимое:

```bash
ls
```

В каталоге должны находиться:

```text
configure
src
doc
contrib
README
```

# Шаг 7. Настройка сборки PostgreSQL

Выполнить подготовку конфигурации:

```bash
./configure --prefix=/opt/postgresql
```

В процессе выполнения PostgreSQL проверит:

наличие компилятора;

наличие библиотек;

поддержку SSL;

наличие ICU;

доступность необходимых инструментов.

При успешном завершении должна появиться информация о создании файлов конфигурации сборки.

Проверить отсутствие ошибок:

```bash
echo $?
```

Результат:

```text
0
```

# Шаг 8. Компиляция PostgreSQL

Запустить сборку:

```bash
make -j$(nproc)
```

Команда:

```bash
$(nproc)
```

автоматически определяет количество доступных ядер процессора.

После завершения проверить результат:

```bash
echo $?
```

Ожидаемый результат:

```text
0
```

# Шаг 9. Установка PostgreSQL

Установить собранный сервер:

```bash
sudo make install
```

Проверить содержимое каталога:

```bash
ls /opt/postgresql
```

Должны появиться каталоги:

```text
bin
include
lib
share
```

# Шаг 10. Проверка установленных программ PostgreSQL

Проверить сервер PostgreSQL:

```bash
/opt/postgresql/bin/postgres --version
```

Ожидаемый результат:

```text
postgres (PostgreSQL) 18.6
```

Проверить клиент:

```bash
/opt/postgresql/bin/psql --version
```

# Шаг 11. Инициализация кластера PostgreSQL

Переключиться на пользователя PostgreSQL:

```bash
sudo -i -u postgres
```

Создать кластер базы данных:

```bash
/opt/postgresql/bin/initdb -D /opt/postgresql/data
```

После выполнения проверить содержимое каталога:

```bash
ls /opt/postgresql/data
```

Должны появиться:

```text
base
global
pg_wal
postgresql.conf
pg_hba.conf
```

# Шаг 12. Запуск сервера PostgreSQL

Запустить сервер:

```bash
/opt/postgresql/bin/pg_ctl \
-D /opt/postgresql/data \
-l /opt/postgresql/logfile \
start
```

Проверить состояние:

```bash
/opt/postgresql/bin/pg_ctl \
-D /opt/postgresql/data status
```

# Шаг 13. Подключение к PostgreSQL

Открыть консоль PostgreSQL:

```bash
/opt/postgresql/bin/psql
```

Проверить версию сервера:

```sql
SELECT version();
```

Выйти из консоли:

```sql
\q
```

# Шаг 14. Управление процессом PostgreSQL

Выполнить остановку сервера:

```bash
/opt/postgresql/bin/pg_ctl \
-D /opt/postgresql/data stop
```

Проверить состояние:

```bash
/opt/postgresql/bin/pg_ctl \
-D /opt/postgresql/data status
```

Запустить сервер повторно:

```bash
/opt/postgresql/bin/pg_ctl \
-D /opt/postgresql/data \
-l /opt/postgresql/logfile \
start
```

# Подсказки по ключевым частям

## Если ошибка при выполнении configure

В первую очередь проверить наличие зависимостей:

```bash
pkg-config --version
```

Проверить ICU:

```bash
pkg-config --modversion icu-uc
```

Если библиотека не найдена, значит отсутствует пакет `libicu-dev` или `pkg-config`.

## Если ошибка доступа к каталогам

Проверить владельца:

```bash
ls -la /opt
```

Каталог PostgreSQL должен принадлежать:

```text
postgres postgres
```

Исправление:

```bash
sudo chown -R postgres:postgres /opt/postgresql
```

## Если команда postgres не найдена

Использовать полный путь:

```bash
/opt/postgresql/bin/postgres
```

или добавить путь временно:

```bash
export PATH=/opt/postgresql/bin:$PATH
```

Проверка:

```bash
postgres --version
```

## Если сервер не запускается

Посмотреть журнал:

```bash
cat /opt/postgresql/logfile
```

Проверить статус:

```bash
/opt/postgresql/bin/pg_ctl \
-D /opt/postgresql/data status
```

# Что проверить перед отправкой

## Подготовка системы

□ Ubuntu обновлена.

□ Все зависимости установлены одной командой.

□ Установлены:

```text
build-essential
libreadline-dev
zlib1g-dev
flex
bison
libssl-dev
libxml2-dev
libicu-dev
pkg-config
python3-dev
perl
wget
tar
```

## Пользователь PostgreSQL

□ Создан системный пользователь:

```text
postgres
```

□ Команда:

```bash
whoami
```

возвращает:

```text
postgres
```

## Установка PostgreSQL

□ Исходный код PostgreSQL 18.6 скачан.

□ Архив успешно распакован.

□ Выполнена команда:

```bash
./configure --prefix=/opt/postgresql
```

□ Компиляция завершилась без ошибок.

□ PostgreSQL установлен в:

```text
/opt/postgresql
```

## Сервер базы данных

□ Создан каталог данных:

```text
/opt/postgresql/data
```

□ Выполнена инициализация:

```bash
initdb
```

□ Сервер успешно запускается через:

```bash
pg_ctl start
```

□ Выполняется подключение через:

```bash
psql
```

□ Команда:

```sql
SELECT version();
```

показывает PostgreSQL 18.6.

# Советы по улучшению работы

После выполнения базовой установки рекомендуется дополнительно изучить структуру созданного каталога данных PostgreSQL.

Обратить внимание на:

```text
postgresql.conf
```

Основной файл настроек сервера.

```text
pg_hba.conf
```

Файл управления правилами подключения пользователей.

```text
pg_wal
```

Каталог журнала упреждающей записи PostgreSQL.

Также рекомендуется научиться выполнять установку в отдельный каталог с версией:

```text
/opt/postgresql-18.6
```

а затем создавать символическую ссылку:

```bash
sudo ln -s /opt/postgresql-18.6 /opt/postgresql
```

Такой подход часто используется на серверах, где необходимо иметь возможность быстро переключаться между версиями PostgreSQL.

Для дополнительного контроля можно изучить вывод процессов:

```bash
ps aux | grep postgres
```

и определить назначение каждого процесса PostgreSQL после запуска.

[1]: https://docs.pgedge.com/postgresql/v16/server-administration/installation-from-source-code/requirements/?utm_source=chatgpt.com "Requirements"
