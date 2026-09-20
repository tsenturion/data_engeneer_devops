# Практическая работа № 1

## Развёртывание `master1` и установка PostgreSQL 18.6 из исходного кода

## Цель работы

Развернуть в Oracle VirtualBox один Ubuntu-сервер `master1`, настроить для него два сетевых подключения, организовать безопасный доступ по SSH и установить PostgreSQL 18.6 из исходного кода.

В результате выполнения работы вы будете уметь:

- создать и установить серверную виртуальную машину;
- объяснить назначение NAT и сетевого моста;
- настроить проброс порта VirtualBox и статический адрес через Netplan;
- подключаться к `master1` по SSH из PowerShell и VS Code;
- установить зависимости для сборки PostgreSQL;
- объяснить различия между этапами `apt`, `configure`, `make`, `make install` и `initdb`;
- собрать PostgreSQL 18.6 с дополнительными возможностями;
- создать кластер базы данных и службу `systemd`;
- проверить работу сервера и изучить его журналы.

Работа выполняется только на одной виртуальной машине — `master1`.

Docker для сборки PostgreSQL из исходного кода не нужен и в этой работе не устанавливается. Передача файлов выполняется через SSH/SFTP; отдельный FTP-сервер также не требуется.

## Используемые версии

На момент актуализации задания используются:

- Oracle VirtualBox;
- Ubuntu Server;
- PostgreSQL 18.6;
- DBeaver;
- PowerShell;
- Visual Studio Code.

Официальные страницы:

- [Visual Studio Code](https://code.visualstudio.com/download);
- [Oracle VirtualBox](https://www.oracle.com/virtualization/technologies/vm/downloads/virtualbox-downloads.html);
- [Ubuntu Server](https://ubuntu.com/download/server);
- [установка PostgreSQL 18 из исходного кода](https://www.postgresql.org/docs/18/installation.html);
- [исходный код PostgreSQL 18.6](https://www.postgresql.org/ftp/source/v18.6/).

## Итоговая схема

| Компонент                    | Значение                                                    |
| ---------------------------- | ----------------------------------------------------------- |
| Основная ОС                  | Windows 10 или Windows 11                                   |
| Виртуальная машина           | `master1`                                                   |
| Пользователь Ubuntu          | `admin`                                                     |
| Сетевой адаптер 1            | NAT, адрес выдаётся по DHCP                                 |
| Резервный SSH через NAT      | `127.0.0.1:2222` → `master1:22`                             |
| Сетевой адаптер 2            | Сетевой мост, статический адрес в домашней или учебной сети |
| Адрес моста в примере        | `192.168.0.33/24`                                           |
| Каталог программы PostgreSQL | `/opt/postgresql`                                           |
| Каталог данных               | `/data/postgresql/18/main`                                  |
| Служба                       | `postgresql-18.service`                                     |

Адрес `192.168.0.33/24` приведён только как пример. Перед настройкой нужно определить параметры своей сети и выбрать свободный адрес вне диапазона DHCP либо зарезервировать его на маршрутизаторе.

## Почему используются два сетевых адаптера

NAT предоставляет виртуальной машине доступ в Интернет и не зависит от адреса домашней сети. Проброс `127.0.0.1:2222 → 22` позволяет подключиться к SSH даже при недоступном сетевом мосте.

Сетевой мост делает `master1` отдельным узлом локальной сети. К нему можно обращаться напрямую, например по адресу `192.168.0.33`.

В этой схеме маршрут по умолчанию и DNS получает только NAT-интерфейс. Для интерфейса моста задаётся лишь статический адрес локальной сети. Это предотвращает появление двух конкурирующих маршрутов по умолчанию.

# Выполнение работы

## Шаг 1. Подготовка Windows

### 1.1. Определение параметров локальной сети

Открыть PowerShell и выполнить:

```powershell
ipconfig
```

Для активного адаптера найти:

- IPv4-адрес компьютера;
- маску подсети;
- основной шлюз.

Пример:

```text
IPv4-адрес:      192.168.0.10
Маска:           255.255.255.0
Основной шлюз:   192.168.0.1
```

Маска `255.255.255.0` соответствует префиксу `/24`. В такой сети адрес `master1` должен находиться в диапазоне `192.168.0.1–192.168.0.254`, не совпадать со шлюзом и другими устройствами.

Перед выбором адреса нужно проверить диапазон DHCP в настройках маршрутизатора. Для примера далее используется `192.168.0.33/24`.

### 1.2. Установка Visual Studio Code

Скачать стабильный Windows User Installer с официального сайта и запустить установку. На странице дополнительных задач включить все пять доступных флажков:

- создание значка на рабочем столе;
- действие «Открыть с помощью Code» для файлов;
- действие «Открыть с помощью Code» для каталогов;
- регистрация VS Code как редактора поддерживаемых типов файлов;
- добавление VS Code в `PATH`.

После установки открыть раздел Extensions и установить расширение:

```text
ms-vscode-remote.remote-ssh
```

Расширения Python и Jupyter для этой работы не обязательны.

### 1.3. Установка VirtualBox

Установить актуальный Oracle VirtualBox. Extension Pack для этой работы не требуется.

Если VirtualBox предлагает установить сетевые драйверы, подтвердить установку. Кратковременный разрыв сетевого соединения в этот момент является нормальным.

## Шаг 2. Создание виртуальной машины `master1`

Скачать ISO-образ Ubuntu Server 26.04.1 LTS AMD64 и создать новую виртуальную машину со следующими параметрами:

| Параметр | Рекомендуемое значение                  |
| -------- | --------------------------------------- |
| Name     | `master1`                               |
| Type     | Linux                                   |
| Version  | Ubuntu (64-bit)                         |
| RAM      | не менее 4096 МБ, рекомендуется 8192 МБ |
| CPU      | не менее 2, рекомендуется 4             |
| Диск     | 50 ГБ, VDI, динамический                |

Для пошаговой установки рекомендуется включить **Skip Unattended Installation**.

На время установки оставить сетевой адаптер 1 в режиме NAT. Запустить ВМ и установить Ubuntu Server.

В установщике указать:

```text
Hostname: master1
Username: admin
```

На этапе SSH Setup включить **Install OpenSSH server**. Импорт ключей из внешних сервисов не требуется. Дополнительные Featured Server Snaps не устанавливать.

После завершения установки перезагрузить ВМ и извлечь ISO из виртуального привода.

Войти в консоль под пользователем `admin` и проверить имя узла:

```bash
hostnamectl
```

Если имя отличается:

```bash
sudo hostnamectl set-hostname master1
```

## Шаг 3. Настройка NAT и сетевого моста VirtualBox

Полностью выключить виртуальную машину:

```bash
sudo poweroff
```

В VirtualBox открыть **Settings → Network**.

### 3.1. Адаптер 1 — NAT

Установить параметры:

```text
Enable Network Adapter: включено
Attached to: NAT
Cable Connected: включено
```

Открыть **Advanced → Port Forwarding** и создать правило:

| Name        | Protocol | Host IP     | Host Port | Guest IP        | Guest Port |
| ----------- | -------- | ----------- | --------: | --------------- | ---------: |
| SSH-master1 | TCP      | `127.0.0.1` |    `2222` | оставить пустым |       `22` |

Привязка к `127.0.0.1` не публикует этот порт на других интерфейсах Windows.

### 3.2. Адаптер 2 — сетевой мост

Установить параметры:

```text
Enable Network Adapter: включено
Attached to: Bridged Adapter
Name: активный физический Ethernet- или Wi-Fi-адаптер Windows
Promiscuous Mode: Deny
Cable Connected: включено
```

Не выбирать VPN, виртуальный адаптер Hyper-V или отключённый интерфейс. Если мост через Wi-Fi запрещён политикой точки доступа, прямой адрес локальной сети может не работать; доступ через NAT и порт `2222` при этом сохранится.

Запустить `master1`.

## Шаг 4. Определение имён интерфейсов Ubuntu

В консоли ВМ выполнить:

```bash
ip -br address
ip route
```

Обычно VirtualBox создаёт:

```text
enp0s3 — NAT
enp0s8 — сетевой мост
```

Имена могут отличаться. Во всех следующих командах нужно использовать имена, полученные именно на своей ВМ.

Проверить доступ в Интернет через NAT:

```bash
ping -c 4 1.1.1.1
getent hosts ubuntu.com
```

## Шаг 5. Настройка статического адреса через Netplan

Этот этап сначала выполняется из консоли VirtualBox, а не по SSH: ошибка в YAML может временно отключить сеть.

Посмотреть существующие файлы:

```bash
ls -la /etc/netplan
```

Создать резервную копию текущей конфигурации. В следующей команде вместо `50-cloud-init.yaml` указать имя существующего YAML-файла:

```bash
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak
```

Открыть этот же YAML-файл:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Пример конфигурации:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: false
    enp0s8:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.0.33/24
```

Важные особенности:

- NAT-интерфейс получает адрес, маршрут по умолчанию и DNS по DHCP;
- мост получает только статический адрес локальной сети;
- для `enp0s8` не задаются `routes: default` и `nameservers`;
- табуляция в YAML запрещена, используются пробелы.

Установить безопасные права, проверить синтаксис и применить конфигурацию с возможностью автоматического отката:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan generate
sudo netplan try --timeout 120
```

Если сеть работает, подтвердить конфигурацию клавишей Enter. Затем выполнить:

```bash
sudo netplan apply
```

Проверить результат:

```bash
ip -br address
ip route
resolvectl status
```

Ожидается:

- адрес NAT на первом интерфейсе;
- `192.168.0.33/24` на интерфейсе моста;
- один маршрут `default`, созданный NAT-интерфейсом;
- маршрут к локальной сети через интерфейс моста.

Если `netplan try` откатил настройки, восстановить резервную копию:

```bash
sudo cp /etc/netplan/50-cloud-init.yaml.bak /etc/netplan/50-cloud-init.yaml
sudo netplan apply
```

## Шаг 6. Настройка и проверка SSH

Если OpenSSH не был выбран в установщике Ubuntu, установить его из консоли ВМ:

```bash
sudo apt update
sudo apt install -y openssh-server
```

Включить службу и проверить её состояние:

```bash
sudo systemctl enable --now ssh
systemctl status ssh --no-pager
ss -lntp | grep ':22'
```

Если используется UFW, открыть SSH:

```bash
sudo ufw allow OpenSSH
sudo ufw status
```

### 6.1. Проверка NAT-подключения

В PowerShell на Windows выполнить:

```powershell
ssh admin@127.0.0.1 -p 2222
```

### 6.2. Проверка подключения через мост

```powershell
ssh admin@192.168.0.33
```

При переустановке ВМ старая запись ключа узла может мешать подключению. Удалить только соответствующую запись:

```powershell
ssh-keygen -R "[127.0.0.1]:2222"
ssh-keygen -R "192.168.0.33"
```

## Шаг 7. Настройка входа по отдельному SSH-ключу

В PowerShell создать отдельный ключ для `master1`:

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519-master1" -C "master1-admin"
```

Передать открытый ключ через стабильное NAT-подключение:

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519-master1.pub" | ssh -p 2222 admin@127.0.0.1 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Проверить вход по ключу:

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519-master1" admin@127.0.0.1 -p 2222
```

Открыть локальный файл `%USERPROFILE%\.ssh\config` и добавить:

```sshconfig
Host master1
    HostName 192.168.0.33
    User admin
    Port 22
    IdentityFile ~/.ssh/id_ed25519-master1
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3

Host master1-nat
    HostName 127.0.0.1
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519-master1
    IdentitiesOnly yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

Проверить оба псевдонима:

```powershell
ssh master1
ssh master1-nat
```

В VS Code выполнить **Remote-SSH: Connect to Host...** и выбрать `master1`. Если мост недоступен, использовать `master1-nat`.

### Дополнительно: передача файлов через FileZilla

FileZilla необязательна: VS Code Remote SSH уже умеет работать с файлами ВМ. Если требуется графическая передача файлов, создать в FileZilla подключение с протоколом **SFTP — SSH File Transfer Protocol**:

```text
Host: 127.0.0.1
Port: 2222
User: admin
Key file: C:\Users\<имя_пользователя>\.ssh\id_ed25519-master1
```

Для подключения через мост можно использовать адрес `192.168.0.33` и порт `22`.

Прямой вход по SSH под `root` и правило `admin ALL=(ALL) NOPASSWD: ALL` в этой работе не настраиваются. Для административных команд используется `sudo` с паролем пользователя `admin`.

После успешной настройки рекомендуется выключить ВМ и создать снимок VirtualBox с именем `base-network-ssh`.

## Шаг 8. Подготовка Ubuntu к сборке PostgreSQL

Подключиться по SSH:

```powershell
ssh master1
```

Проверить систему:

```bash
hostnamectl
lsb_release -a
uname -a
ip -br address
```

Обновить пакеты:

```bash
sudo apt update
sudo apt full-upgrade -y
```

Если после обновления установлен новый kernel, перезагрузить ВМ и подключиться снова:

```bash
sudo reboot
```

Установить компилятор, инструменты и библиотеки разработки:

```bash
sudo apt install -y \
    build-essential \
    pkg-config \
    flex \
    bison \
    gettext \
    perl \
    libperl-dev \
    python3 \
    python3-dev \
    tcl \
    tcl-dev \
    libreadline-dev \
    zlib1g-dev \
    libicu-dev \
    libssl-dev \
    libxml2-dev \
    libxslt1-dev \
    libkrb5-dev \
    libldap-dev \
    libpam0g-dev \
    libsystemd-dev \
    libcurl4-openssl-dev \
    liblz4-dev \
    libzstd-dev \
    uuid-dev \
    libnuma-dev \
    liburing-dev \
    ca-certificates \
    curl \
    tar
```

Проверить основные инструменты и библиотеки:

```bash
gcc --version
make --version
pkg-config --version
pkg-config --modversion icu-uc
pkg-config --modversion openssl
python3 --version
```

### Что сделал `apt`

`apt` установил инструменты и системные библиотеки до начала сборки PostgreSQL:

- `build-essential`, `flex`, `bison` — инструменты компиляции;
- пакеты `*-dev` — заголовочные файлы и библиотеки для компоновки;
- `libssl-dev` — поддержка TLS;
- `libicu-dev` — ICU-локали и правила сортировки;
- `python3-dev`, `libperl-dev`, `tcl-dev` — серверные процедурные языки;
- `libldap-dev`, `libpam0g-dev`, `libkrb5-dev` — дополнительные способы аутентификации;
- `liblz4-dev`, `libzstd-dev` — дополнительные алгоритмы сжатия;
- `libsystemd-dev` — уведомления о готовности службы;
- `liburing-dev` — поддержка `io_uring` в PostgreSQL 18.

Установка библиотек сама по себе не добавляет эти возможности в уже собранный PostgreSQL. Их наличие будет проверено на следующем этапе.

## Шаг 9. Получение исходного кода PostgreSQL 18.6

Создать постоянный каталог исходников:

```bash
sudo install -d -o admin -g admin /usr/local/src/postgresql
cd /usr/local/src/postgresql
```

Скачать и распаковать официальный архив:

```bash
curl -fLO https://ftp.postgresql.org/pub/source/v18.6/postgresql-18.6.tar.gz
tar -xzf postgresql-18.6.tar.gz
cd postgresql-18.6
```

Проверить содержимое:

```bash
ls
```

В каталоге должны присутствовать `configure`, `src`, `contrib`, `doc` и `README`.

## Шаг 10. Конфигурация сборки

Выполнить:

```bash
./configure \
    --prefix=/opt/postgresql \
    --enable-nls \
    --with-perl \
    --with-python \
    --with-tcl \
    --with-ssl=openssl \
    --with-gssapi \
    --with-ldap \
    --with-pam \
    --with-systemd \
    --with-uuid=e2fs \
    --with-libcurl \
    --with-libnuma \
    --with-liburing \
    --with-libxml \
    --with-libxslt \
    --with-lz4 \
    --with-zstd
```

В PostgreSQL 18 ICU и zlib включены по умолчанию, поэтому отдельных параметров `--with-icu` и `--with-zlib` нет. Пакеты `libicu-dev` и `zlib1g-dev` всё равно должны быть установлены.

Проверить код завершения:

```bash
echo $?
```

Ожидаемый результат — `0`.

### Что сделал `configure`

`configure` ничего не устанавливает. Он:

- проверяет компилятор и ОС;
- ищет ранее установленные библиотеки;
- проверяет запрошенные параметры;
- формирует файлы сборки, включая `Makefile`.

Параметр `--with-ssl=openssl`, например, означает: использовать поддержку OpenSSL при сборке, если необходимые заголовки и библиотеки уже установлены.

## Шаг 11. Компиляция и установка

Собрать сервер и двоичные модули `contrib` без документации:

```bash
make -j"$(nproc)" world-bin
```

Проверить результат:

```bash
echo $?
```

Ожидаемый результат — `0`.

Установить собранные файлы:

```bash
sudo make install-world-bin
```

Проверить установку:

```bash
/opt/postgresql/bin/postgres --version
/opt/postgresql/bin/psql --version
ls -la /opt/postgresql
```

Ожидаемая версия:

```text
postgres (PostgreSQL) 18.6
```

### Что сделали `make` и `make install`

- `make` скомпилировал исходный код с возможностями, обнаруженными `configure`;
- `make install-world-bin` скопировал сервер, клиентские программы, библиотеки и модули `contrib` в `/opt/postgresql`;
- кластер базы данных на этих этапах ещё не создан.

## Шаг 12. Создание системного пользователя и каталога данных

Проверить, существует ли системный пользователь `postgres`:

```bash
id postgres
```

Если пользователь отсутствует, создать его:

```bash
sudo adduser --system --group --home /var/lib/postgresql --shell /bin/bash postgres
```

Создать каталог данных с закрытыми правами:

```bash
sudo install -d -o postgres -g postgres -m 700 /data/postgresql/18/main
```

Каталог `/opt/postgresql` содержит программу и остаётся под управлением `root`. Каталог `/data/postgresql/18/main` содержит изменяемые данные и принадлежит `postgres`.

Добавить программы PostgreSQL в системный `PATH`:

```bash
sudo nano /etc/profile.d/postgresql.sh
```

Содержимое файла:

```bash
export PATH=/opt/postgresql/bin:$PATH
export MANPATH=/opt/postgresql/share/man:${MANPATH:-}
```

Применить в текущем сеансе:

```bash
source /etc/profile.d/postgresql.sh
```

## Шаг 13. Инициализация кластера

Создать кластер с контрольными суммами, UTF-8 и ICU-локалью:

```bash
sudo -u postgres /opt/postgresql/bin/initdb \
    -D /data/postgresql/18/main \
    --encoding=UTF8 \
    --locale-provider=icu \
    --icu-locale=ru-RU \
    --data-checksums \
    --auth-local=peer \
    --auth-host=scram-sha-256
```

Выбор локали делается до начала работы с данными. Если по заданию нужна другая сортировка, заменить `ru-RU` до выполнения `initdb`.

Проверить структуру кластера:

```bash
sudo -u postgres ls -la /data/postgresql/18/main
```

Должны появиться `base`, `global`, `pg_wal`, `postgresql.conf` и `pg_hba.conf`.

`initdb` создаёт новый кластер базы данных. Этот шаг не является частью компиляции и не должен повторяться при обычной пересборке бинарных файлов.

## Шаг 14. Настройка PostgreSQL

Открыть основной файл конфигурации:

```bash
sudo -u postgres nano /data/postgresql/18/main/postgresql.conf
```

Добавить в конец файла:

```conf
listen_addresses = '*'
password_encryption = 'scram-sha-256'

logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_truncate_on_rotation = on
log_line_prefix = '%m [%p] %q%u@%d '
```

Разрешить парольные подключения из локальной сети. Открыть:

```bash
sudo -u postgres nano /data/postgresql/18/main/pg_hba.conf
```

Добавить в конец:

```conf
host    all    all    192.168.0.0/24    scram-sha-256
```

Если фактическая сеть отличается, заменить адрес сети и префикс.

Значение `listen_addresses = '*'` позволяет службе запускаться даже при временно недоступном мосте. Фактический удалённый доступ ограничивают правила `pg_hba.conf` и, при включённом UFW, правило межсетевого экрана для локальной подсети.

Создать каталог журналов и настроить очистку файлов старше 30 дней:

```bash
sudo install -d -o postgres -g postgres -m 700 /data/postgresql/18/main/log
sudo nano /etc/tmpfiles.d/postgresql-18.conf
```

Содержимое:

```text
d /data/postgresql/18/main/log 0700 postgres postgres 30d -
```

Применить правило создания каталога:

```bash
sudo systemd-tmpfiles --create /etc/tmpfiles.d/postgresql-18.conf
```

## Шаг 15. Создание службы `systemd`

Создать unit-файл:

```bash
sudo nano /etc/systemd/system/postgresql-18.service
```

Содержимое:

```ini
[Unit]
Description=PostgreSQL 18.6 database server
Documentation=https://www.postgresql.org/docs/18/
After=network.target

[Service]
Type=notify
User=postgres
Group=postgres
ExecStart=/opt/postgresql/bin/postgres -D /data/postgresql/18/main
ExecReload=/bin/kill -HUP $MAINPID
KillMode=mixed
KillSignal=SIGINT
TimeoutStopSec=120

[Install]
WantedBy=multi-user.target
```

Перечитать конфигурацию `systemd`, включить автозапуск и запустить сервер:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now postgresql-18
```

Проверить:

```bash
systemctl status postgresql-18 --no-pager
sudo journalctl -u postgresql-18 -n 50 --no-pager
ss -lntp | grep ':5432'
```

Если используется UFW и подключение к базе с Windows требуется напрямую, разрешить порт 5432 только из своей локальной сети:

```bash
sudo ufw allow from 192.168.0.0/24 to any port 5432 proto tcp
sudo ufw status
```

Не открывать PostgreSQL для всех адресов правилом `sudo ufw allow 5432`.

## Шаг 16. Создание роли, базы и итоговая проверка

Подключиться локально под системным пользователем `postgres`:

```bash
sudo -u postgres /opt/postgresql/bin/psql -d postgres
```

Выполнить:

```sql
SELECT version();
SHOW data_directory;
SHOW server_encoding;
SHOW lc_collate;

CREATE ROLE admin LOGIN;
\password admin
CREATE DATABASE lab OWNER admin;
\q
```

Команда `\password admin` запросит пароль и не сохранит его в истории оболочки.

Проверить подключение по TCP внутри ВМ:

```bash
psql -h 127.0.0.1 -U admin -d lab -W
```

Внутри `psql` выполнить:

```sql
SELECT current_user, current_database(), inet_server_addr(), inet_server_port();
\q
```

Проверить управление службой:

```bash
sudo systemctl restart postgresql-18
systemctl is-active postgresql-18
systemctl is-enabled postgresql-18
```

Проверить файловый журнал:

```bash
sudo -u postgres ls -lh /data/postgresql/18/main/log
sudo -u postgres tail -n 30 /data/postgresql/18/main/log/postgresql-$(date +%F).log
```

# Смысл последовательности установки

Правильная логическая цепочка выглядит так:

```text
VirtualBox и Ubuntu
        ↓
NAT, мост, Netplan и SSH
        ↓
apt: компилятор и системные библиотеки
        ↓
configure: проверка окружения и формирование Makefile
        ↓
make: компиляция исходного кода
        ↓
make install: установка готовых файлов программы
        ↓
initdb: создание кластера данных
        ↓
конфигурация PostgreSQL и запуск через systemd
```

`configure` не скачивает и не устанавливает OpenSSL, Python, ICU и другие компоненты. Он только проверяет, доступны ли их ранее установленные библиотеки, и подготавливает сборку.

# Если после установки нужна дополнительная возможность

## Не была установлена системная библиотека или пропущен параметр `configure`

Установить недостающий пакет, вернуться в каталог исходников и выполнить полную пересборку с прежними и новыми параметрами:

```bash
cd /usr/local/src/postgresql/postgresql-18.6
make distclean
./configure <все необходимые параметры>
make -j"$(nproc)" world-bin
sudo make install-world-bin
sudo systemctl restart postgresql-18
```

`make distclean` нужен, потому что изменилось окружение, которое проверяет `configure`.

Повторять `initdb` не нужно: программа находится в `/opt/postgresql`, а кластер и пользовательские данные — в `/data/postgresql/18/main`.

## Требуется только расширение из `contrib`

В этой работе выполнены `make world-bin` и `make install-world-bin`, поэтому двоичные модули `contrib` уже установлены. Нужное расширение активируется отдельно в конкретной базе, например:

```bash
sudo -u postgres psql -d lab
```

```sql
CREATE EXTENSION dblink;
CREATE EXTENSION "uuid-ossp";
\dx
```

Установка файлов расширения и команда `CREATE EXTENSION` — разные этапы: первая помещает файлы в систему, вторая регистрирует расширение в выбранной базе.

# Чем эта схема отличается от установки через `apt`

Официальные бинарные пакеты обычно предпочтительнее для обычного рабочего сервера. Сборка из исходников используется здесь в учебных целях и когда необходим контроль параметров сборки.

| Этап                      | Из исходного кода                                    | Через пакетный менеджер                              |
| ------------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| Компиляция                | выполняется вручную                                  | выполнена создателем пакета                          |
| Каталоги                  | выбирает администратор                               | определяет пакет                                     |
| Пользователь `postgres`   | создаётся вручную                                    | создаётся пакетом                                    |
| Кластер                   | создаётся через `initdb`                             | обычно создаётся пакетными скриптами                 |
| Служба `systemd`          | создаётся вручную                                    | поставляется пакетом                                 |
| Дополнительные компоненты | параметры `configure` и пересборка                   | готовые пакеты, если они предусмотрены дистрибутивом |
| Обновления                | новая сборка и контролируемая замена бинарных файлов | обновление пакетов                                   |

Обновление внутри одной основной ветки, например `18.4 → 18.6`, обычно не требует повторного `initdb`. Переход на новую основную версию, например `18 → 19`, требует отдельной миграции через `pg_upgrade` либо выгрузку и восстановление данных — пакетный менеджер не отменяет это требование.

# Диагностика типовых ошибок

## Не работает `ssh master1-nat`

Проверить:

- ВМ запущена;
- адаптер 1 работает в режиме NAT;
- правило проброса использует `127.0.0.1`, порт хоста `2222` и порт гостя `22`;
- служба SSH активна: `systemctl status ssh`;
- порт слушается: `ss -lntp | grep ':22'`.

## Не работает `ssh master1`

Сначала проверить `ssh master1-nat`. Если NAT работает, проблема находится в мосте, статическом адресе или локальной сети.

На ВМ проверить:

```bash
ip -br address
ip route
sudo netplan generate
```

В VirtualBox убедиться, что адаптер 2 подключён к активному физическому интерфейсу Windows и установлен флажок **Cable Connected**.

## Ошибка Netplan

Проверить отступы и имена интерфейсов:

```bash
sudo netplan --debug generate
```

Для безопасной проверки использовать:

```bash
sudo netplan try --timeout 120
```

## Ошибка `configure`

Прочитать последние строки вывода и файл `config.log`:

```bash
tail -n 50 config.log
```

Проверить конкретную библиотеку через `pkg-config`, например:

```bash
pkg-config --modversion icu-uc
pkg-config --modversion openssl
pkg-config --modversion libxml-2.0
```

После установки отсутствующей зависимости выполнить `make distclean` и повторить `configure` со всеми параметрами.

## PostgreSQL не запускается

Проверить состояние и оба источника журналов:

```bash
systemctl status postgresql-18 --no-pager
sudo journalctl -u postgresql-18 -n 100 --no-pager
sudo -u postgres find /data/postgresql/18/main/log -maxdepth 1 -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort
```

Проверить владельца и права каталога данных:

```bash
namei -l /data/postgresql/18/main
```

## Порт 5432 уже занят

```bash
sudo ss -lntp | grep ':5432'
```

Причиной может быть PostgreSQL, ранее установленный через `apt`. Не запускать одновременно два сервера на одном адресе и порту.

# Контрольный список

- [ ] Создана только одна ВМ `master1`.
- [ ] На адаптере 1 настроены NAT и проброс `127.0.0.1:2222 → 22`.
- [ ] На адаптере 2 настроен сетевой мост.
- [ ] Netplan содержит NAT с DHCP и мост со статическим адресом без второго маршрута по умолчанию.
- [ ] Работают подключения `ssh master1` и `ssh master1-nat`.
- [ ] Вход выполняется по отдельному ключу `id_ed25519-master1`.
- [ ] PostgreSQL 18.6 собран с указанными параметрами.
- [ ] Программа установлена в `/opt/postgresql`.
- [ ] Кластер находится в `/data/postgresql/18/main`.
- [ ] Служба `postgresql-18` активна и включена в автозапуск.
- [ ] Созданы роль `admin` и база `lab`.
- [ ] Журналы PostgreSQL создаются и очищаются не позднее чем через 30 дней.
- [ ] `SELECT version();` показывает PostgreSQL 18.6.

# Вопросы для самопроверки

1. Чем режим NAT отличается от сетевого моста?
2. Зачем при наличии моста сохраняется проброс SSH через NAT?
3. Почему в данной схеме нельзя задавать второй маршрут по умолчанию на интерфейсе моста?
4. Что устанавливает `apt`, а что делает `configure`?
5. Чем отличаются `make`, `make install` и `initdb`?
6. Почему каталог программы и каталог данных принадлежат разным пользователям?
7. В каком случае требуется повторная сборка PostgreSQL, а в каком достаточно `CREATE EXTENSION`?
8. Нужно ли повторять `initdb` после пересборки PostgreSQL 18.6 и почему?
