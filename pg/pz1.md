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
- подключиться к базе `postgres` из DBeaver на Windows;
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
- [DBeaver Community](https://dbeaver.io/download/);
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

### 1.4. Установка DBeaver

Скачать DBeaver Community для Windows с официального сайта и установить его. Подключение к PostgreSQL настраивается после запуска сервера, в шаге 16.

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

## Шаг 7. Настройка входа по SSH для `admin` и `root`

В PowerShell проверить наличие стандартного открытого ключа:

```powershell
Test-Path "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

Если результат `False`, создать ключ:

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519" -C "master1"
```

При создании ключа можно оставить парольную фразу пустой, если вход должен выполняться без дополнительных запросов. Файл `id_ed25519` — закрытый ключ; его не передают на сервер.

Передать открытый ключ пользователю `admin` через NAT-подключение:

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub" | ssh -p 2222 admin@127.0.0.1 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Проверить вход без указания пути к ключу: SSH сам использует стандартный `id_ed25519`.

```powershell
ssh admin@127.0.0.1 -p 2222
```

В SSH-сеансе `admin` открыть отдельное правило `sudo`:

```bash
sudo visudo -f /etc/sudoers.d/90-admin-nopasswd
```

Добавить строку:

```text
admin ALL=(ALL) NOPASSWD: ALL
```

Проверить, что `sudo` больше не запрашивает пароль:

```bash
sudo -n true
```

Настроить вход `root` по тому же открытому ключу. Задать пароль учётной записи `root`, чтобы разблокировать её и сохранить возможность входа без ключа:

```bash
sudo passwd root
sudo install -d -o root -g root -m 700 /root/.ssh
sudo nano /root/.ssh/authorized_keys
```

`sudo passwd root` задаёт пароль и разблокирует учётную запись; отдельная команда `passwd -u root` после этого не нужна.

В PowerShell показать открытый ключ:

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

Скопировать полученную строку целиком в `/root/.ssh/authorized_keys`, сохранить файл и на ВМ выполнить:

```bash
sudo chown root:root /root/.ssh/authorized_keys
sudo chmod 600 /root/.ssh/authorized_keys
sudo nano /etc/ssh/sshd_config
```

Убедиться, что в конфигурации разрешены вход `root` и аутентификация по паролю:

```text
PermitRootLogin yes
PubkeyAuthentication yes
PasswordAuthentication yes
```

В обычной работе SSH использует открытый ключ из `authorized_keys` и не спрашивает пароль учётной записи. Без ключа войти тоже можно, но тогда потребуется пароль `root`: вход одновременно без ключа и без пароля не настроен. Проверить конфигурацию и перезапустить SSH:

```bash
sudo /usr/sbin/sshd -t
sudo systemctl restart ssh
sudo /usr/sbin/sshd -T | grep '^permitrootlogin '
sudo /usr/sbin/sshd -T | grep '^pubkeyauthentication '
sudo /usr/sbin/sshd -T | grep '^passwordauthentication '
```

Ожидаемые значения — `permitrootlogin yes`, `pubkeyauthentication yes` и `passwordauthentication yes`. В PowerShell проверить вход `root` через NAT без указания ключа и без запроса пароля учётной записи:

```powershell
ssh root@127.0.0.1 -p 2222
```

Отдельно проверить запасной вход без ключа: следующая команда должна запросить пароль `root`.

```powershell
ssh -o PubkeyAuthentication=no root@127.0.0.1 -p 2222
```

Открыть локальный файл `%USERPROFILE%\.ssh\config` и добавить:

```sshconfig
Host master1
    HostName 192.168.0.33
    User admin
    Port 22
    ServerAliveInterval 60
    ServerAliveCountMax 3

Host master1-admin
    HostName 127.0.0.1
    User admin
    Port 2222
    ServerAliveInterval 60
    ServerAliveCountMax 3

Host master1-root
    HostName 127.0.0.1
    User root
    Port 2222
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

`localhost` вместо `127.0.0.1` также подходит, если проброс VirtualBox доступен по этому имени. Проверить три подключения:

```powershell
ssh master1
ssh master1-admin
ssh master1-root
```

В VS Code выполнить **Remote-SSH: Connect to Host...** и выбрать `master1`. Если мост недоступен, использовать `master1-admin`.

### Передача файлов через FileZilla от `root`

Для прямой записи в `/usr/local/src` создать в FileZilla подключение с протоколом **SFTP — SSH File Transfer Protocol**:

```text
Host: 127.0.0.1
Port: 2222
User: root
Logon type: Ask for password
```

При подключении ввести пароль учётной записи `root`, заданный командой `sudo passwd root`. Ключ в `/root/.ssh/authorized_keys` нужен для входа через `ssh master1-root` без пароля и не указывается в профиле FileZilla. Пользователь `root` может записывать в `/usr/local/src` без промежуточной передачи в `/home/admin`. Порядок передачи архива приведён в шаге 9.

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

Обновить список доступных пакетов:

```bash
sudo apt update
```

Полное обновление Ubuntu — отдельный необязательный этап подготовки. Если оно требуется, выполнить:

```bash
sudo apt full-upgrade
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

Для следующих практик с созданием Excel- и PDF-файлов из PL/Python также установить библиотеки для системного Python:

```bash
sudo apt install -y python3-openpyxl python3-reportlab
```

Это Python-библиотеки, а не расширения PostgreSQL. Их устанавливают в окружение Python, используемое сервером.

## Шаг 9. Получение исходного кода PostgreSQL 18.6

Основной способ — скачать архив непосредственно на `master1`:

```bash
cd /usr/local/src
sudo curl -fLO https://ftp.postgresql.org/pub/source/v18.6/postgresql-18.6.tar.gz
```

Если загрузка внутри ВМ недоступна или идёт слишком медленно, использовать FileZilla:

1. На Windows скачать [архив PostgreSQL 18.6](https://ftp.postgresql.org/pub/source/v18.6/postgresql-18.6.tar.gz) через браузер.
2. Открыть SFTP-подключение `root` в FileZilla с параметрами из шага 7.
3. В левой панели FileZilla найти скачанный `postgresql-18.6.tar.gz`, в правой открыть `/usr/local/src`. Перетащить архив в правую панель и дождаться завершения передачи.

После получения архива любым из двух способов распаковать его в `/usr/local/src`:

```bash
cd /usr/local/src
sudo tar -xzf postgresql-18.6.tar.gz
sudo chown -R admin:admin /usr/local/src/postgresql-18.6
cd /usr/local/src/postgresql-18.6
```

Исходники остаются в `/usr/local/src/postgresql-18.6` для последующей установки модулей.

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

В PostgreSQL 18 ICU и zlib включены по умолчанию, поэтому явно задавать `--with-icu` и `--with-zlib` не требуется. Пакеты `libicu-dev` и `zlib1g-dev` всё равно должны быть установлены.

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

Цели `world-bin` и `install-world-bin` уже включают модули `contrib`. Отдельная сборка `pageinspect`, `pg_buffercache`, `pg_stat_statements` и `dblink` нужна только при их отсутствии после установки. `plpython3u` понадобится в следующих работах курса; он находится в `src/pl/plpython`, а не в `contrib`, включён параметром `--with-python` и устанавливается вместе с сервером.

Проверить установку:

```bash
/opt/postgresql/bin/postgres --version
/opt/postgresql/bin/psql --version
ls /opt/postgresql/share/extension/{pageinspect,pg_buffercache,pg_stat_statements,dblink,plpython3u}.control
```

Ожидаемая версия:

```text
postgres (PostgreSQL) 18.6
```

### Что сделали сборка и установка

- `make world-bin` скомпилировал сервер, выбранный при `configure` язык PL/Python и двоичные модули `contrib`;
- `make install-world-bin` установил их в `/opt/postgresql`;
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

Параметр `--auth-local=peer` настраивает вход через локальный Unix-сокет по имени пользователя Linux. К существующей базе `postgres` можно подключиться под одноимённой ролью PostgreSQL без пароля командой `sudo -u postgres /opt/postgresql/bin/psql -d postgres`. Здесь `sudo` запускает только процесс `psql` от имени системного пользователя `postgres`; для `admin` с правилом `NOPASSWD` из шага 7 пароль Ubuntu также не запрашивается. Подключение с `-h 127.0.0.1` использует TCP и проверяется отдельно по паролю PostgreSQL.

## Шаг 14. Настройка PostgreSQL

Открыть основной файл конфигурации:

```bash
sudo -u postgres nano /data/postgresql/18/main/postgresql.conf
```

Добавить параметры в конец файла. Если `shared_preload_libraries` уже задан, дополнить существующий список значением `pg_stat_statements`; если оно там уже есть, повторно не добавлять:

```conf
listen_addresses = '*'
password_encryption = 'scram-sha-256'
shared_preload_libraries = 'pg_stat_statements'

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

`pg_stat_statements` требует загрузки при старте сервера. При первоначальном запуске это обеспечит указанная настройка. Если её добавили уже работающему серверу, выполнить `sudo systemctl restart postgresql-18` перед использованием расширения.

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

## Шаг 16. Установка расширений в базу `postgres` и итоговая проверка

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
SELECT datlocprovider, datlocale
FROM pg_database WHERE datname = current_database();
SHOW shared_preload_libraries;

CREATE EXTENSION pageinspect;
CREATE EXTENSION pg_buffercache;
CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION dblink;
CREATE EXTENSION plpython3u;
SELECT count(*) FROM pg_stat_statements;
DO $$
from openpyxl import Workbook
from reportlab.pdfgen import canvas
$$ LANGUAGE plpython3u;
\password postgres
\dx
\q
```

База `postgres` и роль `postgres` уже созданы командой `initdb`. Для ICU ожидаются `datlocprovider = i` и `datlocale = ru-RU`; значение `lc_collate` может отличаться, поскольку показывает локаль libc. Запрос к `pg_stat_statements` проверяет, что расширение загружено при старте сервера, а блок `DO` — что `openpyxl` и `reportlab` доступны внутри PL/Python.

Локальная команда `sudo -u postgres /opt/postgresql/bin/psql -d postgres` выше подключается через Unix-сокет без пароля PostgreSQL: правило `peer` сопоставляет системного пользователя `postgres` с одноимённой ролью. Команда `\password postgres` задаёт пароль только для TCP-подключений и не сохраняет его в истории оболочки. В этой учебной конфигурации использовать пароль `admin`.

Отдельно проверить подключение по TCP внутри ВМ:

```bash
psql -h 127.0.0.1 -U postgres -d postgres -W
```

Внутри `psql` выполнить:

```sql
SELECT current_user, current_database(), inet_server_addr(), inet_server_port();
\q
```

### Подключение через DBeaver с Windows

Все действия в этом разделе выполняются в DBeaver на хосте Windows. Выбрать **Database → New Database Connection → PostgreSQL**. Для прямого подключения через сетевой мост указать:

| Параметр | Значение |
| --- | --- |
| Host | `192.168.0.33` — адрес моста `master1` в примере |
| Port | `5432` |
| Database | `postgres` |
| Username | `postgres` |
| Password | пароль, заданный командой `\password postgres` |

В поле **Host** указать фактический адрес сетевого моста виртуальной машины; `192.168.0.33` — только пример из этой инструкции. Адрес `127.0.0.1` без SSH-туннеля здесь не подходит: он указывает на сам Windows-хост.

Если на ВМ настроен UFW, правило для локальной сети из шага 15 должно разрешать порт `5432`. Нажать **Test Connection**, затем **Finish**. Если DBeaver предложит загрузить драйвер PostgreSQL, подтвердить загрузку.

Если сетевой мост недоступен, использовать существующее SSH-подключение через NAT. В основных параметрах соединения DBeaver указать `Host: localhost`, `Port: 5432`, базу `postgres`, пользователя `postgres` и тот же пароль. В настройках **SSH** добавить туннель:

| Параметр SSH | Значение |
| --- | --- |
| Host/IP | `127.0.0.1` |
| Port | `2222` |
| User name | `admin` — пользователь Ubuntu |
| Authentication | `Public Key` |
| Private key | `C:\Users\<имя_пользователя>\.ssh\id_ed25519` |

Проверить туннель кнопкой **Test tunnel configuration**, затем нажать **Test Connection** и **Finish**. Для туннеля открывать порт `5432` в UFW не требуется.

Открыть SQL-редактор созданного соединения и выполнить:

```sql
SELECT current_user, current_database();
```

Ожидаются значения `postgres` и `postgres`.

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
make world-bin: компиляция сервера и модулей contrib
        ↓
make install-world-bin: установка собранных файлов
        ↓
initdb: создание кластера данных
        ↓
конфигурация PostgreSQL и запуск через systemd
        ↓
CREATE EXTENSION в нужной базе
```

`configure` не скачивает и не устанавливает OpenSSL, Python, ICU и другие компоненты. Он только проверяет, доступны ли их ранее установленные библиотеки, и подготавливает сборку.

# Если после установки нужна дополнительная возможность

## Не была установлена системная библиотека или пропущен параметр `configure`

Установить недостающий пакет, вернуться в каталог исходников и выполнить полную пересборку с прежними и новыми параметрами:

```bash
cd /usr/local/src/postgresql-18.6
make distclean
./configure <все необходимые параметры>
make -j"$(nproc)" world-bin
sudo make install-world-bin
```

После установки перезапустить службу:

```bash
sudo systemctl restart postgresql-18
```

`make distclean` нужен, потому что изменилось окружение, которое проверяет `configure`.

Повторять `initdb` не нужно: программа находится в `/opt/postgresql`, а кластер и пользовательские данные — в `/data/postgresql/18/main`.

## Требуется только расширение из `contrib`

Если после сборки только сервера без `contrib` команда `CREATE EXTENSION` сообщает `extension "..." is not available`, проверить наличие файла управления расширением. Например, для `dblink`:

```bash
ls /opt/postgresql/share/extension/dblink.control
```

Если файла нет, собрать и установить соответствующий модуль из `/usr/local/src/postgresql-18.6/contrib`:

```bash
cd /usr/local/src/postgresql-18.6/contrib/dblink
make USE_PGXS=1 PG_CONFIG=/opt/postgresql/bin/pg_config
sudo make USE_PGXS=1 PG_CONFIG=/opt/postgresql/bin/pg_config install
```

Для `pageinspect`, `pg_buffercache` и `pg_stat_statements` вместо `dblink` указать имя нужного каталога. Если файл уже есть, повторная сборка не требуется. Затем подключиться к нужной базе и зарегистрировать расширение, например:

```bash
sudo -u postgres /opt/postgresql/bin/psql -d postgres
```

```sql
CREATE EXTENSION dblink;
\dx
```

Установка файлов расширения и команда `CREATE EXTENSION` — разные этапы: первая помещает файлы в систему, вторая регистрирует расширение в выбранной базе.

Для `pg_stat_statements` дополнительно нужна строка `shared_preload_libraries = 'pg_stat_statements'` в `postgresql.conf` и перезапуск уже работающего сервера. `plpython3u` устанавливается при сборке PostgreSQL с `--with-python`; это процедурный язык из `src/pl/plpython`, а не модуль `contrib`.

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

## Не работает `ssh master1-admin` или `ssh master1-root`

Проверить:

- ВМ запущена;
- адаптер 1 работает в режиме NAT;
- правило проброса использует `127.0.0.1`, порт хоста `2222` и порт гостя `22`;
- служба SSH активна: `systemctl status ssh`;
- порт слушается: `ss -lntp | grep ':22'`.

## Не работает `ssh master1`

Сначала проверить `ssh master1-admin`. Если NAT работает, проблема находится в мосте, статическом адресе или локальной сети.

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
- [ ] Работают подключения `ssh master1`, `ssh master1-admin` и `ssh master1-root`.
- [ ] Открытый ключ `id_ed25519.pub` находится в `authorized_keys` пользователей Ubuntu `admin` и `root`; стандартный закрытый ключ `id_ed25519` остаётся на Windows.
- [ ] Правило `admin ALL=(ALL) NOPASSWD: ALL` действует; SSH использует ключ без запроса пароля при обычном входе.
- [ ] FileZilla подключается по SFTP от `root` с паролем и может передать архив в `/usr/local/src`.
- [ ] PostgreSQL 18.6 собран с указанными параметрами.
- [ ] Исходники находятся в `/usr/local/src/postgresql-18.6`.
- [ ] Программа установлена в `/opt/postgresql`.
- [ ] Установлены файлы `pageinspect`, `pg_buffercache`, `pg_stat_statements`, `dblink` и `plpython3u`.
- [ ] Кластер находится в `/data/postgresql/18/main`.
- [ ] Служба `postgresql-18` активна и включена в автозапуск.
- [ ] В базе `postgres` созданы все пять расширений; `pg_stat_statements` загружен через `shared_preload_libraries`.
- [ ] Команда `sudo -u postgres /opt/postgresql/bin/psql -d postgres` подключается через Unix-сокет без пароля PostgreSQL.
- [ ] DBeaver на Windows подключается к базе `postgres` через мост или SSH-туннель.
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
