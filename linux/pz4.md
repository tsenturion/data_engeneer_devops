# Практическое задание

## Цель

Развернуть на сервере Ubuntu учебную сетевую службу, настроить её управление через `systemd`, организовать журналирование, проверить сетевую доступность и DNS, настроить безопасное подключение по SSH, выполнить передачу файлов между двумя узлами, создать резервные копии данных и автоматизировать их запуск через `cron` и `at`.

В результате работы должна получиться небольшая администрируемая система. На основном сервере будет работать пользовательская служба, которая публикует тестовую веб-страницу. Второй сервер или клиентская машина будет использоваться для проверки сети, подключения по SSH, копирования файлов и хранения резервной копии.

Для выполнения задания желательно использовать две виртуальные машины Ubuntu:

```text
server1 — основной сервер
worker1 — удалённый сервер для резервных копий
```

Если доступна только одна машина, часть команд можно выполнить через `localhost`, но работу с `scp`, `rsync` и SSH-ключами лучше проверить между двумя отдельными виртуальными машинами.

## Задание

### Шаг 1. Подготовьте рабочую директорию и данные службы

На основном сервере создайте структуру проекта:

```bash
sudo mkdir -p /opt/demo-service/site
sudo mkdir -p /var/log/demo-service
```

Создайте тестовую веб-страницу:

```bash
echo "<h1>Учебная служба работает</h1>" | sudo tee /opt/demo-service/site/index.html
```

Добавьте на страницу имя сервера и текущую дату:

```bash
{
    echo "<h1>Учебная служба работает</h1>"
    echo "<p>Сервер: $(hostname)</p>"
    echo "<p>Страница создана: $(date)</p>"
} | sudo tee /opt/demo-service/site/index.html
```

Проверьте содержимое файла:

```bash
cat /opt/demo-service/site/index.html
```

Определите полный путь к Python:

```bash
which python3
```

Обычно результат будет таким:

```text
/usr/bin/python3
```

### Шаг 2. Создайте собственную службу systemd

Создайте unit-файл:

```bash
sudo nano /etc/systemd/system/demo-web.service
```

Добавьте в него следующую конфигурацию:

```ini
[Unit]
Description=Учебная веб-служба
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/demo-service/site
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 0.0.0.0
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Сохраните файл и сообщите `systemd`, что появилась новая конфигурация:

```bash
sudo systemctl daemon-reload
```

Проверьте синтаксис unit-файла:

```bash
systemd-analyze verify /etc/systemd/system/demo-web.service
```

Запустите службу:

```bash
sudo systemctl start demo-web.service
```

Посмотрите её состояние:

```bash
systemctl status demo-web.service
```

Проверьте краткое состояние:

```bash
systemctl is-active demo-web.service
```

Остановите службу:

```bash
sudo systemctl stop demo-web.service
```

Убедитесь, что она остановлена:

```bash
systemctl is-active demo-web.service
```

Снова запустите службу и выполните её перезапуск:

```bash
sudo systemctl start demo-web.service
sudo systemctl restart demo-web.service
```

Измените текст в файле `index.html`, после чего ещё раз перезапустите службу:

```bash
echo "<p>Служба была перезапущена</p>" | sudo tee -a /opt/demo-service/site/index.html
sudo systemctl restart demo-web.service
```

Настройте автоматический запуск службы при загрузке Ubuntu:

```bash
sudo systemctl enable demo-web.service
```

Проверьте, включена ли автозагрузка:

```bash
systemctl is-enabled demo-web.service
```

Можно одновременно включить автозагрузку и запустить службу:

```bash
sudo systemctl enable --now demo-web.service
```

### Шаг 3. Изучите журналы службы

Просмотрите последние сообщения службы:

```bash
journalctl -u demo-web.service
```

Выведите только последние двадцать строк:

```bash
journalctl -u demo-web.service -n 20
```

Посмотрите журнал с подробными пояснениями:

```bash
journalctl -xeu demo-web.service
```

Откройте просмотр новых сообщений в реальном времени:

```bash
journalctl -u demo-web.service -f
```

В другом терминале обратитесь к веб-службе:

```bash
curl http://127.0.0.1:8080
```

После проверки остановите просмотр журнала сочетанием:

```text
Ctrl+C
```

Посмотрите сообщения службы только за текущую загрузку системы:

```bash
journalctl -u demo-web.service -b
```

Посмотрите журнал за последние десять минут:

```bash
journalctl -u demo-web.service --since "10 minutes ago"
```

Создайте тестовую ошибку. Временно измените строку `ExecStart` в unit-файле, указав неправильный путь:

```ini
ExecStart=/usr/bin/python999 -m http.server 8080
```

После этого выполните:

```bash
sudo systemctl daemon-reload
sudo systemctl restart demo-web.service
systemctl status demo-web.service
journalctl -xeu demo-web.service
```

Зафиксируйте, какое сообщение объясняет причину отказа. Затем верните правильный путь:

```ini
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 0.0.0.0
```

Примените исправление:

```bash
sudo systemctl daemon-reload
sudo systemctl restart demo-web.service
```

Проверьте общие журналы Ubuntu:

```bash
ls -lah /var/log
```

Посмотрите последние строки системного журнала, если файл существует:

```bash
sudo tail -n 30 /var/log/syslog
```

Посмотрите журнал аутентификации:

```bash
sudo tail -n 30 /var/log/auth.log
```

Найдите сообщения, связанные с SSH:

```bash
sudo grep -i ssh /var/log/auth.log | tail -n 20
```

Посмотрите журнал пакетного менеджера:

```bash
sudo tail -n 30 /var/log/apt/history.log
```

Сравните обычные файлы в `/var/log` с журналом `journalctl`.

### Шаг 4. Проверьте сетевую конфигурацию сервера

Посмотрите все сетевые интерфейсы:

```bash
ip link show
```

Получите краткий вывод:

```bash
ip -br link
```

Посмотрите назначенные IP-адреса:

```bash
ip addr show
```

Используйте сокращённый формат:

```bash
ip -br addr
```

Определите IPv4-адрес основного интерфейса. Не используйте адрес `127.0.0.1`, если проверяете подключение с другой машины.

Посмотрите таблицу маршрутизации:

```bash
ip route
```

Определите шлюз по умолчанию. Проверьте, каким маршрутом пойдёт соединение к внешнему адресу:

```bash
ip route get 8.8.8.8
```

Проверьте локальный сетевой стек:

```bash
ping -c 4 127.0.0.1
```

Проверьте собственное имя сервера:

```bash
ping -c 4 "$(hostname)"
```

Проверьте шлюз. Его адрес возьмите из вывода `ip route`:

```bash
ping -c 4 АДРЕС_ШЛЮЗА
```

Проверьте доступность внешнего IP-адреса:

```bash
ping -c 4 8.8.8.8
```

Проверьте разрешение доменного имени:

```bash
ping -c 4 example.com
```

Если `traceroute` ещё не установлен, установите его:

```bash
sudo apt update
sudo apt install traceroute
```

Посмотрите маршрут до внешнего узла:

```bash
traceroute example.com
```

Получите вывод без преобразования IP-адресов в имена:

```bash
traceroute -n 8.8.8.8
```

### Шаг 5. Проверьте DNS

Посмотрите содержимое DNS-конфигурации:

```bash
cat /etc/resolv.conf
```

Определите, является ли файл символической ссылкой:

```bash
ls -l /etc/resolv.conf
```

Посмотрите реальные DNS-настройки интерфейсов:

```bash
resolvectl status
```

Выполните DNS-запрос:

```bash
resolvectl query example.com
```

Проверьте разрешение имени через системный механизм:

```bash
getent hosts example.com
```

При наличии пакета `dnsutils` выполните:

```bash
dig example.com
dig +short example.com
```

Если команда `dig` отсутствует, установите её:

```bash
sudo apt install dnsutils
```

Уточните, какой DNS-сервер используется системой и является ли адрес `127.0.0.53` реальным внешним DNS-сервером или локальным посредником `systemd-resolved`.

Не редактируйте `/etc/resolv.conf` вручную, если он управляется `systemd-resolved`.

### Шаг 6. Проверьте открытые порты

Убедитесь, что учебная служба запущена:

```bash
sudo systemctl start demo-web.service
```

Посмотрите все слушающие TCP- и UDP-порты:

```bash
sudo ss -tulpn
```

Найдите порт `8080`:

```bash
sudo ss -ltnp | grep ':8080'
```

Проверьте локальное подключение:

```bash
curl http://127.0.0.1:8080
```

Проверьте подключение через IP-адрес основного интерфейса:

```bash
curl http://IP_СЕРВЕРА:8080
```

Если установлен `netcat`, выполните:

```bash
nc -zv 127.0.0.1 8080
```

При отсутствии команды установите пакет:

```bash
sudo apt install netcat-openbsd
```

Остановите службу и повторите проверку порта:

```bash
sudo systemctl stop demo-web.service
sudo ss -ltnp | grep ':8080'
nc -zv 127.0.0.1 8080
```

После проверки снова запустите службу:

```bash
sudo systemctl start demo-web.service
```

### Шаг 7. Установите и настройте SSH-сервер

Проверьте наличие SSH-сервера:

```bash
dpkg -l | grep openssh-server
```

Если сервер не установлен, выполните:

```bash
sudo apt update
sudo apt install openssh-server
```

Запустите SSH-службу и включите её автозагрузку:

```bash
sudo systemctl enable --now ssh
```

Проверьте статус:

```bash
systemctl status ssh
```

Убедитесь, что SSH слушает порт `22`:

```bash
sudo ss -ltnp | grep ':22'
```

С удалённой машины проверьте доступность сервера:

```bash
ping -c 4 IP_СЕРВЕРА
```

Подключитесь по SSH:

```bash
ssh ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_СЕРВЕРА
```

При первом подключении проверьте отпечаток ключа сервера и подтвердите его добавление в `known_hosts`.

После подключения выполните:

```bash
hostname
whoami
ip -br addr
```

Выйдите из удалённого сеанса:

```bash
exit
```

### Шаг 8. Настройте вход по SSH-ключу

На клиентской машине создайте ключ:

```bash
ssh-keygen -t ed25519 -C "backup-access"
```

Для учебного задания используйте стандартный путь:

```text
~/.ssh/id_ed25519
```

Для безопасной настройки рекомендуется установить парольную фразу на закрытый ключ.

Скопируйте открытый ключ на сервер:

```bash
ssh-copy-id ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_СЕРВЕРА
```

Проверьте подключение:

```bash
ssh ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_СЕРВЕРА
```

На сервере проверьте права доступа:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

При необходимости установите правильные права:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Не передавайте никому закрытый ключ:

```text
~/.ssh/id_ed25519
```

Для передачи используется только открытая часть:

```text
~/.ssh/id_ed25519.pub
```

### Шаг 9. Выполните безопасную настройку SSH

Перед изменением конфигурации создайте резервную копию:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```

Откройте конфигурацию:

```bash
sudo nano /etc/ssh/sshd_config
```

Проверьте или добавьте параметры:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
MaxAuthTries 3
```

На этом этапе не отключайте парольный вход, пока не убедитесь, что вход по ключу работает в отдельном терминале.

Проверьте синтаксис конфигурации:

```bash
sudo sshd -t
```

Если команда не вывела ошибок, примените изменения:

```bash
sudo systemctl reload ssh
```

Не закрывая текущий SSH-сеанс, откройте второй терминал и проверьте вход по ключу:

```bash
ssh ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_СЕРВЕРА
```

Только после успешной проверки можно изменить:

```text
PasswordAuthentication no
```

Снова выполните:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

Проверьте активную конфигурацию:

```bash
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries'
```

Если сервер учебный и доступ к консоли отсутствует, отключение парольного входа выполняйте только после подтверждения работы ключей.

### Шаг 10. Передайте файлы через scp

На основном сервере создайте файл о состоянии:

```bash
{
    echo "Дата: $(date)"
    echo "Сервер: $(hostname)"
    echo
    echo "Состояние службы:"
    systemctl is-active demo-web.service
    echo
    echo "IP-адреса:"
    ip -br addr
    echo
    echo "Открытые порты:"
    ss -ltn
} > ~/server-state.txt
```

Скопируйте файл на удалённый сервер:

```bash
scp ~/server-state.txt ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/
```

Скопируйте весь каталог сайта:

```bash
scp -r /opt/demo-service/site \
ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/demo-site-copy
```

На удалённом сервере проверьте полученные данные:

```bash
ls -lah ~
cat ~/server-state.txt
find ~/demo-site-copy -type f
```

### Шаг 11. Выполните синхронизацию через rsync

Установите `rsync` на обеих машинах, если он отсутствует:

```bash
sudo apt install rsync
```

На удалённом сервере создайте каталог:

```bash
mkdir -p ~/remote-backups/site
```

С основного сервера сначала выполните пробную синхронизацию:

```bash
rsync -av --dry-run /opt/demo-service/site/ \
ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/remote-backups/site/
```

Если вывод корректный, выполните настоящую синхронизацию:

```bash
rsync -av /opt/demo-service/site/ \
ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/remote-backups/site/
```

Измените страницу:

```bash
echo "<p>Добавлено перед повторной синхронизацией</p>" | \
sudo tee -a /opt/demo-service/site/index.html
```

Повторите `rsync` и обратите внимание, что передаётся только изменённый файл:

```bash
rsync -av /opt/demo-service/site/ \
ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/remote-backups/site/
```

### Шаг 12. Создайте архивы

Подготовьте каталог резервных копий:

```bash
sudo mkdir -p /var/backups/demo-service
```

Создайте обычный архив `tar`:

```bash
sudo tar -cvf /var/backups/demo-service/site.tar \
-C /opt/demo-service site
```

Посмотрите содержимое:

```bash
tar -tvf /var/backups/demo-service/site.tar
```

Создайте сжатый архив `tar.gz`:

```bash
sudo tar -czvf /var/backups/demo-service/site.tar.gz \
-C /opt/demo-service site
```

Посмотрите его содержимое:

```bash
tar -tzvf /var/backups/demo-service/site.tar.gz
```

Создайте резервную копию с датой:

```bash
sudo tar -czf \
"/var/backups/demo-service/site-$(date +%Y-%m-%d_%H-%M-%S).tar.gz" \
-C /opt/demo-service site
```

Посмотрите список копий:

```bash
ls -lh /var/backups/demo-service
```

Проверьте восстановление:

```bash
mkdir -p /tmp/demo-restore
sudo tar -xzf /var/backups/demo-service/site.tar.gz \
-C /tmp/demo-restore
find /tmp/demo-restore -type f
```

Установите `zip`, если он отсутствует:

```bash
sudo apt install zip unzip
```

Создайте ZIP-архив:

```bash
cd /opt/demo-service
sudo zip -r /var/backups/demo-service/site.zip site
```

Посмотрите содержимое:

```bash
unzip -l /var/backups/demo-service/site.zip
```

Распакуйте архив для проверки:

```bash
mkdir -p /tmp/demo-zip-restore
unzip /var/backups/demo-service/site.zip \
-d /tmp/demo-zip-restore
```

### Шаг 13. Передайте архив на удалённый сервер

Скопируйте архив через `scp`:

```bash
scp /var/backups/demo-service/site.tar.gz \
ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/remote-backups/
```

Выполните синхронизацию всего каталога резервных копий:

```bash
rsync -av --progress /var/backups/demo-service/ \
ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:~/remote-backups/demo-service/
```

На удалённом сервере проверьте архив:

```bash
tar -tzf ~/remote-backups/demo-service/site.tar.gz
```

### Шаг 14. Настройте пользовательскую cron-задачу

Создайте пользовательский каталог журналов:

```bash
mkdir -p "$HOME/logs"
```

Откройте пользовательский `crontab`:

```bash
crontab -e
```

Добавьте задачу, которая каждую минуту записывает состояние учебной службы:

```cron
* * * * * /usr/bin/systemctl is-active demo-web.service >> "$HOME/logs/demo-service-status.log" 2>&1
```

Подождите две минуты и проверьте результат:

```bash
cat ~/logs/demo-service-status.log
```

После проверки измените расписание, чтобы задача выполнялась раз в десять минут:

```cron
*/10 * * * * /usr/bin/systemctl is-active demo-web.service >> "$HOME/logs/demo-service-status.log" 2>&1
```

Посмотрите пользовательские задания:

```bash
crontab -l
```

### Шаг 15. Настройте системную cron-задачу резервного копирования

Создайте системный cron-файл:

```bash
sudo nano /etc/cron.d/demo-backup
```

Для первоначальной проверки добавьте задачу, запускаемую каждую минуту:

```cron
* * * * * root /usr/bin/tar -czf /var/backups/demo-service/cron-site-$(/usr/bin/date +\%Y-\%m-\%d_\%H-\%M).tar.gz -C /opt/demo-service site >> /var/log/demo-service-backup.log 2>&1
```

Установите права на файл:

```bash
sudo chmod 644 /etc/cron.d/demo-backup
```

Подождите одну или две минуты и проверьте результат:

```bash
ls -lh /var/backups/demo-service
sudo cat /var/log/demo-service-backup.log
```

Посмотрите журнал `cron`:

```bash
journalctl -u cron -n 50
```

Также можно проверить текстовый журнал:

```bash
sudo grep CRON /var/log/syslog | tail -n 20
```

После успешной проверки измените расписание на ежедневный запуск в 02:00:

```cron
0 2 * * * root /usr/bin/tar -czf /var/backups/demo-service/cron-site-$(/usr/bin/date +\%Y-\%m-\%d).tar.gz -C /opt/demo-service site >> /var/log/demo-service-backup.log 2>&1
```

### Шаг 16. Настройте удалённое резервное копирование через cron

Убедитесь, что пользователь, от имени которого работает задача, может подключаться к `worker1` по SSH-ключу без интерактивного ввода пароля:

```bash
ssh ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1 hostname
```

Добавьте в пользовательский `crontab` задачу синхронизации:

```cron
30 2 * * * /usr/bin/rsync -av "$HOME/backups/" ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1:/home/ИМЯ_ПОЛЬЗОВАТЕЛЯ/remote-backups/ >> "$HOME/logs/rsync-backup.log" 2>&1
```

Перед добавлением в `cron` выполните эту же команду вручную и убедитесь, что она не запрашивает подтверждение ключа сервера или пароль.

Если архивы находятся в `/var/backups/demo-service`, настройте права так, чтобы выбранный пользователь мог их читать, либо выполняйте системную задачу от `root` с отдельным SSH-ключом. Не используйте закрытый пользовательский ключ бездумно для системных задач.

### Шаг 17. Создайте одноразовую задачу через at

Установите пакет:

```bash
sudo apt update
sudo apt install at
```

Включите службу:

```bash
sudo systemctl enable --now atd
```

Проверьте её состояние:

```bash
systemctl status atd
```

Создайте одноразовую задачу, которая через две минуты сохранит состояние службы:

```bash
echo '/usr/bin/systemctl status demo-web.service --no-pager > /tmp/demo-web-at-status.txt 2>&1' | at now + 2 minutes
```

Посмотрите очередь:

```bash
atq
```

После выполнения проверьте файл:

```bash
cat /tmp/demo-web-at-status.txt
```

Создайте ещё одну задачу, которая через пять минут создаст архив:

```bash
echo '/usr/bin/tar -czf /tmp/demo-at-backup.tar.gz -C /opt/demo-service site' | at now + 5 minutes
```

Посмотрите очередь:

```bash
atq
```

Удалите задание до его выполнения:

```bash
atrm НОМЕР_ЗАДАНИЯ
```

Снова проверьте очередь:

```bash
atq
```

### Шаг 18. Выполните итоговую проверку системы

Перезагрузите основной сервер:

```bash
sudo reboot
```

После загрузки снова подключитесь по SSH и проверьте:

```bash
systemctl is-enabled demo-web.service
systemctl is-active demo-web.service
systemctl status demo-web.service
```

Убедитесь, что порт `8080` открыт:

```bash
sudo ss -ltnp | grep ':8080'
```

Проверьте веб-службу:

```bash
curl http://127.0.0.1:8080
```

С удалённой машины выполните:

```bash
curl http://IP_СЕРВЕРА:8080
```

Посмотрите журналы текущей загрузки:

```bash
journalctl -u demo-web.service -b
```

Проверьте пользовательские и системные cron-задачи:

```bash
crontab -l
sudo cat /etc/cron.d/demo-backup
```

Проверьте резервные копии:

```bash
ls -lh /var/backups/demo-service
```

Проверьте данные на удалённом сервере:

```bash
ssh ИМЯ_ПОЛЬЗОВАТЕЛЯ@IP_WORKER1 \
'find ~/remote-backups -maxdepth 3 -type f -ls'
```

## Подсказки по ключевым частям

При работе с `systemd` после каждого изменения unit-файла выполняйте:

```bash
sudo systemctl daemon-reload
```

Если служба не запускается, сначала используйте:

```bash
systemctl status demo-web.service
journalctl -xeu demo-web.service
```

Если порт `8080` занят, найдите процесс:

```bash
sudo ss -ltnp | grep ':8080'
```

При диагностике сети двигайтесь последовательно. Сначала проверяйте интерфейс:

```bash
ip -br link
```

Затем адрес:

```bash
ip -br addr
```

После этого маршрут:

```bash
ip route
```

Далее проверяйте шлюз, внешний IP-адрес и DNS-имя:

```bash
ping -c 4 АДРЕС_ШЛЮЗА
ping -c 4 8.8.8.8
ping -c 4 example.com
```

Если внешний IP доступен, а имя не разрешается, исследуйте DNS:

```bash
cat /etc/resolv.conf
resolvectl status
getent hosts example.com
```

Для SSH сначала проверяйте конфигурацию и только затем перезагружайте службу:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

При изменении SSH не закрывайте действующее подключение, пока новый сеанс не был успешно открыт.

В заданиях `cron` используйте абсолютные пути:

```text
/usr/bin/tar
/usr/bin/date
/usr/bin/rsync
/usr/bin/systemctl
```

Не рассчитывайте, что окружение `cron` будет таким же, как в интерактивном терминале.

Внутри `crontab` экранируйте символ `%`:

```text
\%Y-\%m-\%d
```

Если `apt` сообщает о занятой блокировке:

```text
Could not get lock /var/lib/dpkg/lock-frontend
```

проверьте процесс:

```bash
ps aux | grep -E 'apt|dpkg|unattended' | grep -v grep
```

Не удаляйте файлы блокировки вручную, пока реальный процесс `apt` или `unattended-upgrade` продолжает работать.

При работе с `rsync --delete` сначала используйте:

```bash
rsync -av --delete --dry-run ИСТОЧНИК НАЗНАЧЕНИЕ
```

Это позволяет увидеть предполагаемые удаления без изменения файлов.

## Что проверить перед отправкой — чек-лист

- [ ] Создана собственная служба `demo-web.service`.
- [ ] Unit-файл проходит проверку `systemd-analyze verify`.
- [ ] Служба успешно запускается, останавливается и перезапускается.
- [ ] Для службы настроена автозагрузка.
- [ ] После перезагрузки служба запускается автоматически.
- [ ] Команда `systemctl status` показывает состояние `active (running)`.
- [ ] Журналы службы доступны через `journalctl`.
- [ ] Просмотрены журналы в `/var/log`.
- [ ] Определены сетевой интерфейс, IP-адрес и шлюз.
- [ ] Проверены локальный адрес, шлюз и внешний адрес через `ping`.
- [ ] Выполнен `traceroute`.
- [ ] Проверена конфигурация DNS.
- [ ] Объяснено назначение `/etc/resolv.conf`.
- [ ] Порт `8080` отображается через `ss`.
- [ ] Веб-служба доступна через `curl`.
- [ ] Установлен и запущен SSH-сервер.
- [ ] SSH-служба включена в автозагрузку.
- [ ] Выполнено подключение к удалённой машине.
- [ ] Настроена аутентификация по SSH-ключу.
- [ ] Вход по ключу проверен до отключения парольной аутентификации.
- [ ] Запрещён прямой SSH-вход для `root`.
- [ ] Конфигурация SSH проверена через `sshd -t`.
- [ ] Файл передан через `scp`.
- [ ] Каталог синхронизирован через `rsync`.
- [ ] Созданы архивы `.tar`, `.tar.gz` и `.zip`.
- [ ] Проверено содержимое каждого архива.
- [ ] Выполнено тестовое восстановление данных.
- [ ] Резервная копия передана на удалённый сервер.
- [ ] Создана пользовательская cron-задача.
- [ ] Создана системная cron-задача.
- [ ] Объяснена разница форматов пользовательского и системного `cron`.
- [ ] Работа `cron` подтверждена журналами.
- [ ] Создана и выполнена одноразовая задача через `at`.
- [ ] Выполнено удаление ещё не запущенной задачи через `atrm`.

## Советы по улучшению работы

Не ограничивайтесь снимками успешных команд. Добавьте один или два примера ошибок и реализуйте последовательность их диагностики через `systemctl status`, `journalctl`, `ss`, `ping` или SSH-журналы.

Настройте запуск службы не от имени `root`, а от отдельного системного пользователя. Например, создайте пользователя:

```bash
sudo useradd --system --home /opt/demo-service \
--shell /usr/sbin/nologin demo-service
```

Передайте ему рабочие файлы:

```bash
sudo chown -R demo-service:demo-service /opt/demo-service
```

После этого добавьте в раздел `[Service]`:

```ini
User=demo-service
Group=demo-service
```

Ограничьте возможности службы дополнительными параметрами:

```ini
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadOnlyPaths=/opt/demo-service/site
```

Перед использованием этих ограничений изучите их влияние и снова проверьте запуск службы.

Добавьте ротацию резервных копий. Например, удаляйте только учебные архивы старше семи дней:

```bash
find /var/backups/demo-service \
-type f \
-name 'cron-site-*.tar.gz' \
-mtime +7 \
-print
```

Сначала выполните команду только с `-print`. После проверки можно добавить удаление:

```bash
find /var/backups/demo-service \
-type f \
-name 'cron-site-*.tar.gz' \
-mtime +7 \
-delete
```

Проверяйте резервную копию после создания:

```bash
tar -tzf /var/backups/demo-service/site.tar.gz > /dev/null
echo $?
```

Код возврата `0` означает, что команда не обнаружила ошибку чтения архива.

Добавьте контрольную сумму:

```bash
sha256sum /var/backups/demo-service/site.tar.gz \
> /var/backups/demo-service/site.tar.gz.sha256
```

Проверьте её:

```bash
cd /var/backups/demo-service
sha256sum -c site.tar.gz.sha256
```

Для SSH используйте отдельный ключ только для резервного копирования. Ограничьте этот ключ на удалённом сервере и предоставьте владельцу каталога резервных копий минимально необходимые права.
