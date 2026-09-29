# Task 7 — Ansible Server Automation

Автоматизация конфигурации Linux-сервера с помощью Ansible.
Вся конфигурация сервера описана кодом и воспроизводится одной командой.

---

## Содержание

- [Архитектура](#архитектура)
- [Тестовое окружение](#тестовое-окружение)
- [Структура репозитория](#структура-репозитория)
- [Ansible: основные понятия](#ansible-основные-понятия)
- [Приложение](#приложение)
- [Сбор информации о сервере](#сбор-информации-о-сервере)
- [Идемпотентность](#идемпотентность)
- [Восстановление сломанного сервера](#восстановление-сломанного-сервера)
- [Rebuild test](#rebuild-test)
- [Безопасность](#безопасность)
- [Troubleshooting](#troubleshooting)
- [Воспроизведение с нуля](#воспроизведение-с-нуля)
- [AI Assistance](#ai-assistance)

---

## Архитектура

```
Control Node (ноутбук, Omarchy Linux)
        |
        | SSH (ключ ed25519)
        v
Managed Node (Ubuntu 24.04, 192.168.122.17)
```

**Control Node** — машина, на которой установлен Ansible и с которой запускаются
команды. В этом проекте это мой рабочий ноутбук.

**Managed Node** — сервер, который настраивается. Ansible на нём **не установлен**.

Это ключевое свойство Ansible — **push-модель без агента**. Ansible подключается
по обычному SSH, копирует на сервер временные Python-скрипты (модули), выполняет
их, забирает результат и удаляет. На управляемой машине нужен только SSH-доступ
и Python, никакого постоянно работающего агента.

Версии:

```
ansible [core 2.21.3]
  config file = None
  configured module search path = ['/home/tiko/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.14/site-packages/ansible
  ansible collection location = /home/tiko/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.14.7
  jinja version = 3.1.6
  pyyaml version = 6.0.3
```

`config file = None` означает, что `ansible.cfg` в проекте не создавался, и все
настройки берутся из значений по умолчанию. Поэтому инвентарь указывается явно
флагом `-i` при каждом запуске.

На сервере установлены `ansible-core` (движок и базовые модули) и пакет `ansible`
(дополнительные коллекции модулей). Для этого проекта достаточно `ansible-core`.

---

## Тестовое окружение

Учебный сервер, выданный в предыдущих заданиях, был отозван, поэтому managed node
был поднят локально как виртуальная машина.

Стенд: **QEMU/KVM + libvirt**, Ubuntu 24.04 cloud image, настройка при первой
загрузке через **cloud-init**.

```bash
# на ноутбуке
sudo pacman -S --needed qemu-full libvirt virt-install dnsmasq edk2-ovmf
sudo systemctl enable --now libvirtd

qemu-img resize noble-server-cloudimg-amd64.img +8G

virt-install \
  --connect qemu:///system \
  --name training-server \
  --memory 2048 \
  --vcpus 1 \
  --disk /var/lib/libvirt/images/noble-server-cloudimg-amd64.img,format=qcow2 \
  --import \
  --os-variant ubuntu24.04 \
  --network network=default \
  --cloud-init user-data=/var/lib/libvirt/images/user-data.yaml \
  --graphics none \
  --noautoconsole
```

Публичный SSH-ключ внедряется в VM через cloud-init (`user-data.yaml`), то есть
тем же механизмом, которым это делают облачные провайдеры.

Полезные команды управления стендом:

```bash
sudo virsh list --all                                  # список VM
sudo virsh start training-server                       # запуск
sudo virsh shutdown training-server                    # корректное выключение
sudo virsh domifaddr training-server --source lease    # IP из DHCP
sudo virsh console training-server                     # серийная консоль (выход Ctrl+])
```

---

## Подготовка SSH-доступа

Для Ansible создана отдельная пара ключей, не используемая больше нигде:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_training -C "ansible-training"
```

Публичная часть (`ansible_training.pub`) попадает на сервер в
`~/.ssh/authorized_keys` пользователя `ubuntu`. Приватная часть остаётся на
control node в `~/.ssh/` — **вне репозитория**, поэтому физически не может
попасть в git.

Ручная проверка доступа перед использованием Ansible:

```bash
ssh -i ~/.ssh/ansible_training ubuntu@192.168.122.17
```

Аутентификация только по ключу, парольный вход не используется.
Privilege escalation — через `sudo` без пароля (настроено cloud-init).

---

## Структура репозитория

```
ansible-training/
├── inventory/
│   └── hosts.yml                    # описание сервера и параметров подключения
├── playbooks/
│   └── server.yml                   # основной плейбук
├── templates/
│   ├── nginx.conf.j2                # шаблон конфига nginx
│   └── training-app.service.j2      # шаблон systemd-юнита
├── files/
│   └── training-app.sh              # приложение (копируется как есть)
├── vars/
│   └── main.yml                     # переменные
├── README.md
└── .gitignore
```

Разделение `templates/` и `files/`:

- `templates/` — файлы, которые проходят обработку Jinja2 (подстановка переменных)
- `files/` — файлы, которые копируются байт в байт без изменений

---

## Ansible: основные понятия

### Inventory

Файл, описывающий управляемые машины и параметры подключения к ним.

`inventory/hosts.yml`:

```yaml
training:
  hosts:
    server:
      ansible_host: 192.168.122.17
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ~/.ssh/ansible_training
      ansible_python_interpreter: /usr/bin/python3
```

- `training` — имя группы, к ней обращается плейбук
- `server` — имя хоста внутри Ansible (ярлык, к DNS отношения не имеет)
- `ansible_host` — реальный адрес подключения
- `ansible_python_interpreter` — явно зафиксированный интерпретатор

Последний параметр добавлен, чтобы убрать предупреждение:

```
[WARNING]: Host 'server' is using the discovered Python interpreter at
'/usr/bin/python3.12', but future installation of another Python interpreter
could cause a different interpreter to be discovered.
```

Без явного указания Ansible каждый раз сам ищет Python на сервере, и результат
поиска может измениться после обновления системы. Для воспроизводимости
интерпретатор зафиксирован на `/usr/bin/python3` (симлинк на системный Python,
а не на конкретную минорную версию).

Проверка разбора инвентаря:

```bash
ansible-inventory -i inventory/hosts.yml --list
```

```json
{
    "_meta": {
        "hostvars": {
            "server": {
                "ansible_host": "192.168.122.17",
                "ansible_ssh_private_key_file": "~/.ssh/ansible_training",
                "ansible_user": "ubuntu"
            }
        }
    },
    "all": {
        "children": ["ungrouped", "training"]
    },
    "training": {
        "hosts": ["server"]
    }
}
```

Группы `all` и `ungrouped` создаются Ansible автоматически: в `all` попадают все
хосты инвентаря, в `ungrouped` — не входящие ни в одну группу.

### Проверка связи

```bash
ansible training -i inventory/hosts.yml -m ping
```

```
server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Модуль `ping` в Ansible — это **не** ICMP-пинг. Он подключается по SSH,
запускает на сервере Python и получает ответ. То есть проверяет весь стек сразу:
сеть, SSH-аутентификацию и работоспособность Python на удалённой машине.

### Playbook

YAML-файл с описанием **желаемого состояния** сервера. Не список команд, а
описание того, как должно быть. Структура:

```
Playbook (файл)
 └── Play (к каким хостам применяем)
      └── Tasks (список задач, выполняются сверху вниз)
           └── Module (что конкретно делает задача)
```

### Task и Module

**Task** — одна единица работы. **Module** — код, который эту работу выполняет.

```yaml
- name: Create application directory      # task
  ansible.builtin.file:                   # module
    path: /opt/training-app               # параметры модуля
    state: directory
    owner: training-app
    group: training-app
    mode: '0755'
```

`ansible.builtin.file` — полное имя модуля (FQCN). Можно писать коротко `file`,
но полная форма явно показывает, из какой коллекции взят модуль.

**Модуль vs shell/command.** В плейбуке нет ни одной задачи с `shell` или
`command`. Причина в идемпотентности: Ansible не знает, что делает произвольная
команда, поэтому любая её задача всегда рапортует `changed`. Модули же знают
своё предметное состояние и сообщают об изменении только когда оно реально было.

Использованные модули:

| Модуль | Назначение |
|---|---|
| `ansible.builtin.user` | создание системного пользователя |
| `ansible.builtin.file` | директории, симлинки, права, удаление |
| `ansible.builtin.apt` | установка пакетов |
| `ansible.builtin.copy` | копирование файлов без изменений |
| `ansible.builtin.template` | генерация файлов из Jinja2-шаблонов |
| `ansible.builtin.systemd_service` | управление службами systemd |

### Variable

Переменные выносят конфигурационные значения из логики задач.

`vars/main.yml`:

```yaml
---
application_name: training-app
app_user: training-app
app_directory: /opt/training-app
listen_port: 80
server_name: training-server
app_port: 9000
app_message: Hello from training-app
```

Подключение в плейбуке:

```yaml
  vars_files:
    - ../vars/main.yml
```

Цель — сделать плейбук переиспользуемым без изменения логики задач. Смена порта
требует правки **одной строки** в `vars/main.yml`, задачи и шаблоны не трогаются.

### Template

Файл с плейсхолдерами Jinja2, который Ansible превращает в готовый конфиг:

```
Jinja2 template + Ansible variables → Generated configuration → Nginx
```

`templates/nginx.conf.j2`:

```jinja
server {
    listen {{ listen_port }};
    server_name {{ server_name }};

    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/{{ application_name }}_access.log;
    error_log /var/log/nginx/{{ application_name }}_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Результат на сервере (`/etc/nginx/sites-available/training-app`):

```nginx
server {
    listen 80;
    server_name training-server;

    root /var/www/html;
    index index.html;

    access_log /var/log/nginx/training-app_access.log;
    error_log /var/log/nginx/training-app_error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

`{{ ... }}` заменились значениями переменных, а `$uri` остался нетронутым — это
переменная nginx, Jinja2 её не трогает.

### Handler

Задача, выполняемая **только по уведомлению** от другой задачи, вернувшей `changed`.

```
Конфигурация изменилась (changed)
        ↓
    notify
        ↓
Handler выполняется в конце плея
```

```
Конфигурация не менялась (ok)
        ↓
    нет notify
        ↓
Handler не запускается — лишнего перезапуска нет
```

```yaml
  handlers:
    - name: Reload nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded

    - name: Restart training-app
      ansible.builtin.systemd_service:
        name: "{{ application_name }}"
        state: restarted
        daemon_reload: true
```

Два важных свойства хендлеров:

1. Выполняются **в конце плея**, а не сразу после уведомившей задачи.
2. Выполняются **один раз**, даже если их уведомили несколько задач.

Второе видно в логах приложения: две задачи (`Deploy application script` и
`Deploy systemd unit file`) уведомляют один хендлер, но перезапуск происходит
однократно.

`reloaded` vs `restarted`:

- nginx умеет перечитывать конфигурацию без остановки (`reloaded`) — соединения
  не рвутся
- bash-скрипт приложения этого не умеет, единственный способ применить
  изменения — полный перезапуск (`restarted`)

`daemon_reload: true` нужен, потому что systemd кеширует юниты в памяти. Без
перечитывания конфигурации изменённый `.service` файл не будет учтён.

### Idempotency

Свойство, при котором повторный запуск не приводит к повторным изменениям.
Ansible сравнивает текущее состояние с желаемым и меняет только то, что отличается.

| Статус | Значение |
|---|---|
| `ok` | состояние уже правильное, ничего не делал |
| `changed` | состояние отличалось, привёл к нужному |
| `failed` | не смог выполнить |

Это работает благодаря декларативному `state`:
`present`, `absent`, `directory`, `link`, `started`.
Задача говорит «пусть будет так», а не «сделай это».

### Become

`become: true` — выполнение задач с повышением привилегий (через `sudo`).

В этом проекте `become` вынесен на уровень плея, а не проставлен точечно.
Обоснование: **все** задачи плейбука требуют root-прав:

| Задача | Почему нужен root |
|---|---|
| создание пользователя | запись в `/etc/passwd`, `/etc/shadow` |
| создание `/opt/training-app` | запись в `/opt` |
| установка пакетов | apt требует root |
| запись в `/var/www/html` | владелец root |
| запись в `/etc/nginx/` | владелец root |
| запись в `/etc/systemd/system/` | владелец root |
| управление службами | systemctl требует root |

Ни одна задача не выполняется от обычного пользователя, поэтому вынос на уровень
плея не расширяет привилегии сверх необходимого.

Подключение к серверу при этом происходит **не** от root, а от обычного
пользователя `ubuntu`, который повышает права через `sudo` только для выполнения
задач.

---

## Приложение

### Пользователь приложения

Приложение работает от выделенного системного пользователя `training-app`,
не от root.

```yaml
    - name: Create application user
      ansible.builtin.user:
        name: training-app
        system: true
        shell: /usr/sbin/nologin
        home: /opt/training-app
        create_home: false
        state: present
```

- `system: true` — системный пользователь (UID < 1000), не для входа людей
- `shell: /usr/sbin/nologin` — интерактивный вход запрещён
- `create_home: false` — директорию создаёт отдельная задача с нужными правами

Проверка на сервере:

```
$ id training-app
uid=999(training-app) gid=988(training-app) groups=988(training-app)

$ getent passwd training-app
training-app:x:999:988::/opt/training-app:/usr/sbin/nologin

$ ls -ld /opt/training-app
drwxr-xr-x 2 training-app training-app 4096 /opt/training-app
```

UID 999 подтверждает, что пользователь системный.

Попытка войти под этим пользователем:

```
$ sudo su - training-app
This account is currently not available.
```

Сообщение выдаёт `/usr/sbin/nologin`. При этом systemd **может** запускать
процессы от имени этого пользователя — это разные механизмы.

### Директория приложения

`/opt/training-app`, владелец `training-app:training-app`, права `0755`.
Права `777` не используются нигде в проекте.

### systemd-сервис

`templates/training-app.service.j2`:

```ini
[Unit]
Description={{ application_name }} service
After=network.target

[Service]
Type=simple
User={{ app_user }}
Group={{ app_user }}
WorkingDirectory={{ app_directory }}
Environment="APP_PORT={{ app_port }}"
Environment="APP_MESSAGE={{ app_message }}"
ExecStart={{ app_directory }}/training-app.sh
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Разбор секций:

**`[Unit]`** — метаданные и зависимости.
`After=network.target` — не запускать раньше поднятия сети, приложение слушает порт.

**`[Service]`** — как запускать и управлять.

| Директива | Назначение |
|---|---|
| `Type=simple` | процесс из `ExecStart` и есть служба, не форкается в фон |
| `User` / `Group` | запуск от непривилегированного пользователя |
| `Environment` | конфигурация передаётся в приложение через переменные окружения |
| `Restart=always` | перезапуск при любом завершении процесса |
| `RestartSec=5` | пауза перед перезапуском, чтобы не уйти в rate limit |
| `StandardOutput=journal` | stdout приложения пишется в системный журнал |

**`[Install]`** — что делать при `systemctl enable`.
`WantedBy=multi-user.target` — автозапуск при загрузке системы.

### Приложение

`files/training-app.sh` — bash-скрипт, слушающий TCP-порт через `nc`:

```bash
#!/bin/bash

PORT="${APP_PORT:-9000}"
MESSAGE="${APP_MESSAGE:-Hello from training-app}"

echo "Starting training-app on port ${PORT}"

while true; do
    RESPONSE="HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\nConnection: close\r\n\r\n${MESSAGE}\n"
    echo -e "${RESPONSE}" | nc -l -p "${PORT}" -q 1
    echo "$(date '+%Y-%m-%d %H:%M:%S') - request served"
done
```

`"${APP_PORT:-9000}"` — подстановка с значением по умолчанию. Конфигурация
приходит извне (из systemd-юнита), скрипт остаётся универсальным.

Скрипт копируется модулем `copy` (а не `template`), потому что шаблонизация ему
не нужна — конфигурацию он получает через окружение.

Состояние службы:

```
● training-app.service - training-app service
     Loaded: loaded (/etc/systemd/system/training-app.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-09-28 14:40:19 UTC
   Main PID: 1958 (training-app.sh)
      Tasks: 2 (limit: 2316)
     Memory: 616.0K (peak: 968.0K)
     CGroup: /system.slice/training-app.service
             ├─1958 /bin/bash /opt/training-app/training-app.sh
             └─1980 nc -l -p 9000 -q 1
```

В CGroup видно два процесса: сам скрипт (Main PID) и порождённый им `nc`,
который слушает порт.

Логи (`StandardOutput=journal` работает):

```
$ sudo journalctl -u training-app -n 20 --no-pager
Sep 28 14:40:19 training-server systemd[1]: Started training-app.service - training-app service.
Sep 28 14:40:19 training-server training-app.sh[1958]: Starting training-app on port 9000
Sep 28 14:45:12 training-server training-app.sh[1960]: GET / HTTP/1.1
Sep 28 14:45:12 training-server training-app.sh[1960]: Host: 192.168.122.17:9000
Sep 28 14:45:12 training-server training-app.sh[1960]: User-Agent: curl/8.22.0
Sep 28 14:45:13 training-server training-app.sh[1958]: 2026-09-28 14:45:13 - request served
```

Проверка работы:

```
$ curl http://192.168.122.17:9000
Hello from training-app
```

### Nginx

Nginx устанавливается, настраивается и запускается через Ansible.
Конфигурация **никогда** не редактируется вручную на сервере — она приходит из
шаблона.

Задачи:

1. установка пакета
2. генерация конфига из Jinja2-шаблона в `sites-available/`
3. создание симлинка в `sites-enabled/` (включение сайта)
4. удаление симлинка сайта по умолчанию
5. проверка, что служба запущена и добавлена в автозапуск

Разделение `sites-available` / `sites-enabled` — соглашение Debian/Ubuntu:
в первой лежат все конфиги, во второй — симлинки на включённые. Nginx читает
только `sites-enabled/`.

Дефолтный сайт отключается удалением симлинка, сам файл в `sites-available/`
остаётся — конфигурация не уничтожается, сайт просто выключается.

Проверка:

```
$ curl http://192.168.122.17
<html>
<head><title>Training App</title></head>
<body>
<h1>Deploy by Ansible</h1>
</body>
</html>
```

---

## Сбор информации о сервере

Перед настройкой сервер исследован средствами Ansible (модуль `setup`).

```bash
ansible training -i inventory/hosts.yml -m setup -a "filter=ansible_distribution*"
```

```json
"ansible_distribution": "Ubuntu",
"ansible_distribution_file_variety": "Debian",
"ansible_distribution_major_version": "24",
"ansible_distribution_release": "noble",
"ansible_distribution_version": "24.04"
```

```bash
ansible training -i inventory/hosts.yml -m setup -a "filter=ansible_mem*"
```

```json
"ansible_memtotal_mb": 1967,
"ansible_memfree_mb": 1509,
"ansible_memory_mb": {
    "real": {"free": 1509, "total": 1967, "used": 458},
    "swap": {"free": 0, "total": 0, "used": 0}
}
```

```bash
ansible training -i inventory/hosts.yml -m setup -a "filter=ansible_processor*"
```

```json
"ansible_processor": ["0", "GenuineIntel", "11th Gen Intel(R) Core(TM) i7-11850H @ 2.50GHz"],
"ansible_processor_cores": 1,
"ansible_processor_vcpus": 1
```

```bash
ansible training -i inventory/hosts.yml -m setup -a "filter=ansible_default_ipv4"
```

```json
"ansible_default_ipv4": {
    "address": "192.168.122.17",
    "gateway": "192.168.122.1",
    "interface": "enp1s0",
    "macaddress": "52:54:00:94:d7:9e",
    "netmask": "255.255.255.0",
    "mtu": 1500
}
```

```bash
ansible training -i inventory/hosts.yml -m setup -a "filter=ansible_uptime_seconds"
```

```json
"ansible_uptime_seconds": 3886
```

### Сравнение с ручным исследованием

| Задача | Вручную | Через Ansible |
|---|---|---|
| ОС | `hostnamectl`, `cat /etc/os-release` | `ansible_distribution` |
| память | `free -h` | `ansible_memtotal_mb` |
| CPU | `lscpu`, `nproc` | `ansible_processor_vcpus` |
| сеть | `ip a`, `ip route` | `ansible_default_ipv4` |
| диск | `df -h`, `lsblk` | `ansible_mounts`, `ansible_devices` |
| uptime | `uptime` | `ansible_uptime_seconds` |

**Почему автоматизация выигрывает при большом парке серверов:**

Дело не только в скорости набора команд. Ручные утилиты возвращают **текст,
отформатированный для человека**, причём каждая по-своему, и формат отличается
между дистрибутивами. Чтобы использовать это в скрипте, вывод пришлось бы
парсить регулярными выражениями, отдельно под каждый случай.

Ansible возвращает **структурированный JSON с едиными именами полей**. Факт
`ansible_distribution` называется одинаково на Ubuntu, Debian, CentOS и RHEL.
Благодаря этому плейбук может принимать решения на основе фактов — например,
использовать `apt` на Debian-семействе и `dnf` на RedHat-семействе, не меняя
логику задачи.

Плюс один запуск команды опрашивает сразу всю группу хостов, а не один сервер.

---

## Идемпотентность

Ключевая проверка задания: первый запуск изменяет состояние, второй — нет.

**Первый запуск (после полной очистки сервера):**

```
TASK [Create application user] ************************ changed
TASK [Create application directory] ******************* changed
TASK [Install nginx] ********************************** changed
TASK [Create website content] ************************* changed
TASK [Deploy nginx site configuration] **************** changed
TASK [Enable nginx site] ****************************** changed
TASK [Disable default nginx site] ********************* changed
TASK [Deploy application script] ********************** changed
TASK [Deploy systemd unit file] *********************** changed
TASK [Ensure training-app service is running] ********* changed
RUNNING HANDLER [Reload nginx] ************************ changed
RUNNING HANDLER [Restart training-app] **************** changed

PLAY RECAP
server : ok=15  changed=12  unreachable=0  failed=0
```

**Второй запуск, сразу следом:**

```
TASK [Gathering Facts] ******************************** ok
TASK [Create application user] ************************ ok
TASK [Create application directory] ******************* ok
TASK [Install required packages] ********************** ok
TASK [Install nginx] ********************************** ok
TASK [Create website content] ************************* ok
TASK [Ensure nginx is running and enabled] ************ ok
TASK [Deploy nginx site configuration] **************** ok
TASK [Enable nginx site] ****************************** ok
TASK [Disable default nginx site] ********************* ok
TASK [Deploy application script] ********************** ok
TASK [Deploy systemd unit file] *********************** ok
TASK [Ensure training-app service is running] ********* ok

PLAY RECAP
server : ok=13  changed=0  unreachable=0  failed=0
```

`changed=0` — цель достигнута.

Обрати внимание на разницу счётчиков: `ok=15` против `ok=13`. Разница в двух
хендлерах — во втором запуске их никто не уведомил, поэтому они не выполнялись
и не попали в счётчик. Это и есть требование задания:

```
Нет изменений конфигурации → нет уведомления → нет лишнего перезапуска
```

**Почему идемпотентность получилась без дополнительной настройки:**

Все задачи используют модули с декларативным `state`. Если бы те же действия
выполнялись через `command` или `shell` (`mkdir`, `useradd`, `systemctl start`),
каждая такая задача рапортовала бы `changed` при каждом запуске — Ansible не
знает, что сделала произвольная команда, и считает изменением сам факт её
выполнения.

---

## Восстановление сломанного сервера

Проверка того, что плейбук возвращает сервер в желаемое состояние после ручного
вмешательства.

### Что было сломано вручную

```bash
sudo rm -rf /opt/training-app                       # удалена директория приложения
sudo systemctl stop nginx                           # остановлен nginx
sudo rm /etc/nginx/sites-enabled/training-app       # удалён симлинк сайта
sudo chmod 777 /var/www/html/index.html             # испорчены права
```

### Симптомы

```
$ curl http://localhost
curl: (7) Failed to connect to localhost port 80 after 1 ms: Couldn't connect to server

$ ls -l /var/www/html/index.html
-rwxrwxrwx 1 www-data www-data 98 /var/www/html/index.html
```

**Неожиданное наблюдение:** служба `training-app` продолжала показывать
`active (running)` и отвечать на запросы, хотя её исполняемый файл был удалён:

```
$ ls -la /opt/
total 8
drwxr-xr-x  2 root root 4096 .
drwxr-xr-x 22 root root 4096 ..

$ curl http://localhost:9000
Hello from training-app
```

Причина: в Linux процесс удерживает **inode** файла, а не путь к нему. Удаление
файла убирает только имя в директории — процесс продолжает работать со своей
копией в памяти.

Проблема проявилась только при перезапуске:

```
$ sudo systemctl restart training-app
$ systemctl status training-app --no-pager
● training-app.service - training-app service
     Active: activating (auto-restart) (Result: exit-code)
    Process: 3196 ExecStart=/opt/training-app/training-app.sh (code=exited, status=203/EXEC)
   Main PID: 3196 (code=exited, status=203/EXEC)
```

`status=203/EXEC` — стандартный код systemd «не смог выполнить программу».
`activating (auto-restart)` — сработал `Restart=always`, systemd ждёт
`RestartSec=5` и пробует снова.

**Практический вывод:** `systemctl status` показывает состояние процесса,
а не целостность конфигурации. Мониторинг по `systemctl is-active` до
перезапуска показал бы, что всё в порядке.

### Восстановление

Запущен тот же плейбук, без единой ручной команды на сервере:

```
TASK [Create application directory] ******************* changed
TASK [Create website content] ************************* changed
TASK [Ensure nginx is running and enabled] ************ changed
TASK [Enable nginx site] ****************************** changed
TASK [Deploy application script] ********************** changed
RUNNING HANDLER [Reload nginx] ************************ changed
RUNNING HANDLER [Restart training-app] **************** changed

PLAY RECAP
server : ok=15  changed=7  unreachable=0  failed=0
```

Каждое изменение соответствует конкретной поломке:

| Задача | Что восстановила |
|---|---|
| Create application directory | вернула удалённую директорию с владельцем |
| Create website content | вернула права `0644` вместо `777` |
| Ensure nginx running | запустила остановленный nginx |
| Enable nginx site | пересоздала удалённый симлинк |
| Deploy application script | вернула удалённый скрипт |
| Handler Reload nginx | применил конфигурацию |
| Handler Restart training-app | поднял службу с восстановленным скриптом |

Задачи `Deploy systemd unit file` и `Deploy nginx site configuration` вернули
`ok` — эти файлы не были повреждены, и Ansible их не трогал. Исправляется
**только то, что отличается**.

### Проверка

```
$ curl http://192.168.122.17
<html>
<head><title>Training App</title></head>
<body>
<h1>Deploy by Ansible</h1>
</body>
</html>

$ curl http://192.168.122.17:9000
Hello from training-app

$ ls -l /var/www/html/index.html
-rw-r--r-- 1 www-data www-data 98 /var/www/html/index.html

$ ls -l /opt/training-app/
-rwxr-xr-x 1 training-app training-app 407 training-app.sh
```

Права вернулись с `777` к `0644`, всё работает.

Это демонстрирует разницу между **ручной настройкой сервера** и **конфигурацией
желаемого состояния**: при ручном подходе потребовалось бы вспомнить все
сделанные изменения и повторить их.

---

## Rebuild test

Полная имитация чистого сервера: удалено всё, что управляется плейбуком.

```bash
sudo systemctl stop training-app
sudo systemctl disable training-app
sudo rm /etc/systemd/system/training-app.service
sudo systemctl daemon-reload

sudo rm -rf /opt/training-app
sudo userdel training-app

sudo apt purge -y nginx nginx-common
sudo rm -rf /etc/nginx /var/www/html
```

Состояние после очистки:

```
$ systemctl status training-app --no-pager
Unit training-app.service could not be found.

$ id training-app
id: 'training-app': no such user

$ which nginx
(пусто)

$ ls /opt/
(пусто)
```

Восстановление одной командой:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/server.yml
```

```
PLAY RECAP
server : ok=15  changed=12  unreachable=0  failed=0
```

Цепочка выполнения совпала со схемой из задания:

```
Create application user       → Users
Install nginx                 → Packages
Create application directory  → Directories
Deploy nginx site config      → Configuration
Deploy application script     → Application
Deploy systemd unit           → Systemd
Enable nginx site             → Nginx
                              → Working server
```

Проверка после пересборки:

```
$ curl http://192.168.122.17
<html>
<head><title>Training App</title></head>
<body>
<h1>Deploy by Ansible</h1>
</body>
</html>

$ curl http://192.168.122.17:9000
Hello from training-app
```

И сразу повторный запуск плейбука:

```
PLAY RECAP
server : ok=13  changed=0  unreachable=0  failed=0
```

Конфигурация сервера полностью воспроизводима из кода и остаётся идемпотентной
после пересборки.

---

## Безопасность

| Требование | Как выполнено |
|---|---|
| Приватные ключи не в git | ключи в `~/.ssh/`, вне репозитория |
| Нет паролей и токенов | секретов в проекте нет |
| SSH не ослаблен | вход только по ключу, пароли не используются |
| Нет `chmod 777` | права `0755` для директорий и скрипта, `0644` для конфигов |
| Приложение не от root | `User=training-app` в systemd-юните |
| `become` обоснован | см. раздел [Become](#become) |

Пользователь приложения дополнительно ограничен `shell: /usr/sbin/nologin` —
интерактивный вход под ним невозможен.

`.gitignore`:

```
*.retry
*.log
.vault_pass
vars/secrets.yml
```

Отдельный ключ для Ansible (`ansible_training`) не используется больше нигде,
для GitLab создан свой.

---

## Troubleshooting

### Проблема 1 — виртуальная машина не получает IP-адрес

**Симптомы**

`virsh domifaddr` возвращал пустой результат, в том числе с `--source lease`:

```
$ sudo virsh domifaddr training-server --source lease
 Name   MAC address   Protocol   Address
------------------------------------------
```

Файл лизов DHCP `/var/lib/libvirt/dnsmasq/virbr0.status` был пустым, при этом VM
успешно загружалась до приглашения входа (проверено через `virsh console`).

**Исследование**

Проверено, что сеть активна:

```
$ sudo virsh net-list --all
 Name      State    Autostart   Persistent
 default   active   yes         yes
```

Проверено, что интерфейс подключён к домену:

```
$ sudo virsh domiflist training-server
 Interface   Type      Source    Model    MAC
 vnet0       network   default   virtio   52:54:00:94:d7:9e
```

То есть проблема не в конфигурации libvirt. Проверены правила фаервола:

```
$ sudo nft list ruleset | grep -iE "drop|virbr0|libvirt"
# Warning: table ip filter is managed by iptables-nft, do not touch!
		type filter hook input priority filter; policy drop;
		type filter hook forward priority filter; policy drop;
		counter packets 114 bytes 29593 drop
table ip libvirt_network {
		ip saddr 192.168.122.0/24 iif "virbr0" counter accept
```

Обнаружена таблица `ip filter` с политикой `drop` на цепочках INPUT и FORWARD
и ненулевым счётчиком отброшенных пакетов. В nftables несколько таблиц могут
висеть на одном хуке, поэтому разрешающие правила в таблице `libvirt_network`
не спасали — вторая таблица отбрасывала трафик своей политикой.

**Первая гипотеза оказалась неполной.** Изначально предполагалось, что правилами
управляет `iptables.service`, и временные правила `iptables -I` действительно
восстанавливали работу. Но после перезагрузки ноутбука проблема возвращалась.
Проверка сервисов показала настоящего виновника:

```
$ systemctl is-enabled iptables nftables firewalld ufw
disabled
disabled
not-found
enabled

$ systemctl is-active iptables nftables firewalld ufw
inactive
inactive
inactive
active
```

```
$ sudo ufw status verbose
Status: active
Default: deny (incoming), allow (outgoing), deny (routed)
```

**Корневая причина**

Фаерволом управляет **ufw**, который при каждом старте разворачивает свои
правила в iptables, затирая всё, что добавлено вручную. Две его политики по
умолчанию блокировали работу VM:

- `deny (incoming)` — отбрасывались DHCP-запросы от VM к dnsmasq на хосте
- `deny (routed)` — блокировался форвардинг, то есть выход VM в интернет

**Решение**

Правила добавлены средствами ufw, а не через `iptables -I` — только так они
переживают перезагрузку:

```bash
sudo ufw allow in on virbr0          # входящий трафик к хосту (DHCP, DNS)
sudo ufw route allow in on virbr0    # форвардинг из VM наружу
sudo ufw route allow out on virbr0   # ответный трафик к VM
```

Разница между `ufw allow` и `ufw route allow` соответствует разнице между
цепочками INPUT и FORWARD в iptables: первое про трафик **к самому хосту**,
второе — про трафик, идущий **через хост** транзитом.

**Проверка**

```
$ sudo virsh domifaddr training-server --source lease
 Name    MAC address         Protocol   Address
 vnet1   52:54:00:94:d7:9e   ipv4       192.168.122.17/24

$ ssh -i ~/.ssh/ansible_training ubuntu@192.168.122.17
Welcome to Ubuntu 24.04.5 LTS
```

Правила пережили последующие перезагрузки ноутбука — проблема больше не
повторялась.

---

### Проблема 2 — конфигурация nginx не применяется

**Симптомы**

Плейбук отрабатывал успешно, конфиг из шаблона появлялся на сервере, но сайт
продолжал работать по старой конфигурации. Логи, заданные в новом конфиге,
оставались пустыми.

**Исследование**

Первая проверка ввела в заблуждение:

```
$ sudo nginx -T | grep -A3 'server_name'
    server_name training-server;
    root /var/www/html;
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Выглядело так, будто конфигурация применена. **Эта гипотеза оказалась неверной:**
`nginx -T` читает конфигурацию **с диска** и проверяет её синтаксис, а не
показывает то, что работающий процесс держит в памяти.

Решающая проверка — по логам. Если новый конфиг активен, запросы должны
писаться в `training-app_access.log`:

```
$ ls -la /var/log/nginx/
-rw-r----- 1 www-data adm      89 access.log
-rw-r--r-- 1 root     root      0 training-app_access.log

$ curl http://192.168.122.17
$ ls -la /var/log/nginx/
-rw-r----- 1 www-data adm     178 access.log            ← вырос
-rw-r--r-- 1 root     root      0 training-app_access.log ← остался нулевым
```

Запрос записался в старый лог. Гипотеза подтвердилась.

Файлы `training-app_*.log` нулевого размера создал сам `nginx -T` при проверке
синтаксиса — отсюда и владелец `root:root` вместо `www-data:adm`.

**Корневая причина**

Nginx работает со снимком конфигурации, снятым при запуске процесса. Изменение
файлов на диске не влияет на работающий процесс, пока ему не сказано перечитать
конфигурацию. В плейбуке не было задачи, выполняющей reload после изменения
конфига.

**Решение**

Добавлен handler, срабатывающий только при реальном изменении конфигурации:

```yaml
  handlers:
    - name: Reload nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded
```

и `notify: Reload nginx` к трём задачам, которые меняют конфигурацию nginx.

Выбран `reloaded`, а не `restarted`: nginx умеет перечитывать конфигурацию без
остановки, не разрывая активные соединения.

**Проверка**

Изменено значение `listen_port` в `vars/main.yml` с `80` на `8080`:

```
TASK [Deploy nginx site configuration] **************** changed
RUNNING HANDLER [Reload nginx] ************************ changed

PLAY RECAP
server : ok=11  changed=2
```

```
$ curl http://192.168.122.17:8080
<html>
<head><title>Training App</title></head>
...

$ curl http://192.168.122.17
curl: (7) Failed to connect to 192.168.122.17:80 after 0 ms: Could not connect to server
```

Изменение применилось немедленно. Порт возвращён на `80`.

---

### Проблема 3 — предупреждение о Python-интерпретаторе

**Симптомы**

Каждая команда Ansible выводила:

```
[WARNING]: Host 'server' is using the discovered Python interpreter at
'/usr/bin/python3.12', but future installation of another Python interpreter
could cause a different interpreter to be discovered.
```

**Корневая причина**

Модули Ansible — это Python-скрипты, выполняемые на управляемой машине. Если
интерпретатор не указан явно, Ansible ищет его сам (interpreter discovery).
Результат поиска может измениться после установки другой версии Python,
что нарушает воспроизводимость.

**Решение**

Интерпретатор зафиксирован в инвентаре:

```yaml
      ansible_python_interpreter: /usr/bin/python3
```

Указан именно `/usr/bin/python3` (симлинк на системный Python), а не
`/usr/bin/python3.12` — жёсткая привязка к минорной версии сломалась бы при
обновлении дистрибутива.

**Проверка**

```
$ ansible training -i inventory/hosts.yml -m ping
server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Предупреждение исчезло.

---

### Проблема 4 — libvirt: session vs system

**Симптомы**

```
$ virt-install --name training-server ...
ERROR    Network not found: no network with matching name 'default'
```

Позже, при попытке обратиться к уже созданной VM:

```
$ virsh domifaddr training-server
error: failed to get domain 'training-server'
```

**Корневая причина**

У libvirt два независимых режима подключения:

- `qemu:///session` — работает от имени пользователя, не имеет преднастроенной
  сети `default`
- `qemu:///system` — работает через системный демон, содержит NAT-сеть `default`
  (192.168.122.0/24)

`virt-install` без явного `--connect` ушёл в session-режим, где сети `default`
не существует. А `virsh` без `sudo` смотрел в session и не видел домен,
созданный в system.

**Решение**

Явное указание подключения и перенос образов в системную директорию (процесс
QEMU в system-режиме работает не от пользователя и не может читать файлы в
домашней папке с правами `700`):

```bash
sudo mv ~/vms/noble-server-cloudimg-amd64.img /var/lib/libvirt/images/
sudo virt-install --connect qemu:///system --disk /var/lib/libvirt/images/... 
```

Для удобства можно задать подключение по умолчанию:

```bash
export LIBVIRT_DEFAULT_URI=qemu:///system
```

**Проверка**

```
$ sudo virsh list --all
 Id   Name              State
 1    training-server   running
```

---

## Воспроизведение с нуля

Как повторить конфигурацию на свежем Ubuntu-сервере.

### Требования

**Control node:**
- Ansible (`ansible-core` 2.15+)
- SSH-клиент

**Managed node:**
- Ubuntu 22.04 или 24.04
- SSH-доступ
- Python 3
- пользователь с правами sudo

### Шаги

**1. Подготовить SSH-доступ**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_training -C "ansible-training"
ssh-copy-id -i ~/.ssh/ansible_training.pub ubuntu@<IP_СЕРВЕРА>
```

Проверить подключение вручную:

```bash
ssh -i ~/.ssh/ansible_training ubuntu@<IP_СЕРВЕРА>
```

**2. Установить Ansible на control node**

```bash
# Arch
sudo pacman -S ansible

# Ubuntu / Debian
sudo apt install ansible

ansible --version
```

**3. Клонировать репозиторий**

```bash
git clone git@gitlab.com:tikokhlghatyan/ansible-training.git
cd ansible-training
```

**4. Указать свой сервер в инвентаре**

Отредактировать `inventory/hosts.yml`, заменив `ansible_host` на адрес своего
сервера, а при необходимости — `ansible_user` и путь к ключу.

**5. Проверить связь**

```bash
ansible training -i inventory/hosts.yml -m ping
```

Ожидаемый результат — `"ping": "pong"`.

**6. При необходимости изменить параметры**

Все настраиваемые значения находятся в `vars/main.yml`:

| Переменная | Что задаёт |
|---|---|
| `application_name` | имя приложения, systemd-юнита, файлов логов |
| `app_user` | пользователь, от которого работает приложение |
| `app_directory` | директория приложения |
| `listen_port` | порт nginx |
| `server_name` | `server_name` в конфиге nginx |
| `app_port` | порт приложения |
| `app_message` | текст, который отдаёт приложение |

**7. Запустить плейбук**

Предварительный просмотр без внесения изменений:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/server.yml --check
```

Реальный запуск:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/server.yml
```

**8. Проверить результат**

```bash
curl http://<IP_СЕРВЕРА>          # веб-страница nginx
curl http://<IP_СЕРВЕРА>:9000     # приложение
```

```bash
ssh -i ~/.ssh/ansible_training ubuntu@<IP_СЕРВЕРА> \
  "systemctl status training-app --no-pager"
```

**9. Убедиться в идемпотентности**

```bash
ansible-playbook -i inventory/hosts.yml playbooks/server.yml
```

Второй запуск должен дать `changed=0`.

---

## AI Assistance

> Раздел заполняется своими словами. Ниже — заготовка по темам, где помощь ИИ
> действительно повлияла на решение. Нужно дописать, что именно понял и как
> проверил.

### Вопрос 1 — почему VM не получает IP-адрес

**Что спрашивал:** VM загружается, но `virsh domifaddr` пустой, файл лизов
dnsmasq пустой. Где искать причину?

**Что предложил ИИ:** проверить последовательно — активна ли сеть `default`,
подключён ли интерфейс к домену, не блокирует ли трафик фаервол хоста. Для
последнего — посмотреть `nft list ruleset` и определить, какой сервис управляет
правилами.

**Что понял:**

...

**Как проверил:**

...

### Вопрос 2 — почему конфигурация nginx не применяется

**Что спрашивал:** плейбук успешно кладёт конфиг на сервер, `nginx -T` его
показывает, но запросы обслуживаются по-старому.

**Что предложил ИИ:** проверить не через `nginx -T` (он читает с диска), а по
логам — в какой файл реально пишутся запросы. Причина в том, что nginx работает
со снимком конфигурации, снятым при запуске.

**Что понял:**

...

**Как проверил:**

...

### Вопрос 3 — разница между модулями и shell-командами

**Что спрашивал:** почему задание требует использовать модули вместо
`shell`/`command`.

**Что предложил ИИ:** модули знают предметное состояние и сообщают `changed`
только при реальном изменении; произвольная команда всегда считается изменением,
потому что Ansible не знает её семантику.

**Что понял:**

...

**Как проверил:**

...

---

## Что изучено

1. Разница между control node и managed node, и почему Ansible не требует
   агента на управляемой машине.
2. Декларативный подход: описывается желаемое состояние (`state`), а не
   последовательность команд.
3. Идемпотентность и её практическая ценность — плейбук можно запускать
   многократно без побочных эффектов.
4. Разделение логики (задачи) и данных (переменные), шаблонизация конфигов
   через Jinja2.
5. Handlers как механизм реакции на изменения без лишних перезапусков служб.
6. Диагностика по методу «симптом → измерение → гипотеза → проверка»: два раза
   первоначальная гипотеза оказалась неверной, и это выяснилось только
   измерением.
7. `systemctl status` показывает состояние процесса, а не целостность
   конфигурации — процесс может работать с удалённым файлом.
