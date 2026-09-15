# Troubleshooting

Шесть сбоев, встреченных при построении пайплайна. Каждый разобран по схеме задания: проблема, ожидаемое поведение, наблюдаемое, расследование, причина, исправление, проверка.

Все сбои настоящие — возникли в ходе работы, а не были инсценированы.

---

## 1. Postgres не стартует в тестах

**Problem.** Задача `run-tests` падает, все три теста не проходят.

**Expected behavior.** Сервис `postgres:16` поднимается, приложение подключается, тесты проходят.

**Observed behavior.**

```
requests.exceptions.ConnectionError: HTTPConnectionPool(host='localhost', port=9000):
Failed to establish a new connection: [Errno 111] Connection refused
```

Одинаково во всех трёх тестах.

**Investigation.**

`Connection refused` означает, что на порту никто не слушает. Значит сломаны не тесты — тестировать нечего, приложение не поднялось.

Причина не была видна: приложение запускалось командой `python3 src/app.py &`, и вывод фонового процесса терялся. Добавлено перенаправление:

```yaml
- python3 src/app.py > app.log 2>&1 &
- sleep 5
- cat app.log        # ДО запуска тестов
- pytest tests/ -v
```

Порядок важен: `cat` после `pytest` не выполнился бы при падении тестов — то есть ровно тогда, когда нужен.

**Evidence.**

```
psycopg2.OperationalError: connection to server at "postgres" (172.17.0.2),
port 5432 failed: Connection refused
```

Имя резолвится — адрес получен. Значит контейнер сервиса существует. Не отвечает порт.

Проверка переменной:

```yaml
- echo "password length: ${#DB_PASSWORD}"
```

Вернуло `0` — переменная пуста.

**Root cause.** Переменная `DB_PASSWORD` в настройках проекта помечена **Protected** и доступна только в защищённых ветках. Рабочая ветка `feature/gitlab-cicd` не защищена.

Цепочка: переменная пуста → `POSTGRES_PASSWORD` пустой → postgres отказывается инициализироваться → порт не отвечает → приложение падает на `init_db()` → тесты не находят сервер.

Пять ступеней между настройкой галки и симптомом.

**Fix.** Пароль тестовой базы задан прямо в пайплайне:

```yaml
variables:
  POSTGRES_PASSWORD: testpass
  DB_PASSWORD: testpass
```

Это не секрет: база живёт минуту в изолированной сети задачи, порт наружу не проброшен, после завершения уничтожается.

Настоящий пароль вынесен в отдельную переменную `DEPLOY_DB_PASSWORD` — чтобы имена не пересекались и переменная проекта не перекрывала тестовую.

**Verification.** Пайплайн зелёный, `3 passed`.

---

## 2. Приватный ключ вместо пути к файлу

**Problem.** Задача `deploy` падает на подготовке SSH-ключа.

**Expected behavior.** Ключ копируется из временного файла в `~/.ssh/id_ed25519`.

**Observed behavior.**

```
$ cp "$SSH_PRIVATE_KEY" ~/.ssh/id_ed25519
cp: unrecognized option: ---BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQ...
-----END OPENSSH PRIVATE KEY-----
BusyBox v1.37.0 multi-call binary.
Usage: cp [-arPLHpfinlsTu] SOURCE DEST
```

**Investigation.** `cp` получил не путь, а содержимое ключа целиком. Увидел строку, начинающуюся с дефисов, и принял её за флаг.

**Evidence.** Сам вывод ошибки: вместо пути напечатан текст ключа.

**Root cause.** Переменная `SSH_PRIVATE_KEY` создана с типом **Variable** вместо **File**.

С типом Variable переменная содержит само значение. С типом File GitLab записывает значение во временный файл и подставляет **путь** к нему — это и нужно для команд, ожидающих файл.

**Fix.** Тип переменной изменён на File.

**Дополнительное последствие.** Приватный ключ оказался в логе задачи и был скомпрометирован. Пришлось пересоздать пару и обновить `authorized_keys`.

Урок: секрет утекает не только через git, но и через сообщения об ошибках. Команда, получившая секрет вместо пути, напечатала его целиком.

**Verification.** `cp` отработал без ошибок.

---

## 3. Повреждённый ключ

**Problem.** SSH не может прочитать ключ.

**Expected behavior.** Подключение к серверу по ключу.

**Observed behavior.**

```
Load key "/root/.ssh/id_ed25519": error in libcrypto: unsupported
Permission denied, please try again.
Tigran@172.17.0.1: Permission denied (publickey,password).
```

**Investigation.** Ключ на месте, права `600` выставлены — но парсер SSH его не принимает. Значит содержимое повреждено.

**Evidence.** `error in libcrypto: unsupported` возникает при разборе ключа, до попытки аутентификации. Соединение с сервером при этом установилось — `Permission denied (publickey)` означает, что сервер отверг клиента, а не что он недоступен.

**Root cause.** Отсутствие завершающего перевода строки. Приватный ключ обязан заканчиваться переводом строки после `-----END OPENSSH PRIVATE KEY-----`; GitLab не гарантирует его при передаче значения переменной.

**Fix.**

```yaml
- cat "$SSH_PRIVATE_KEY" > ~/.ssh/id_ed25519
- echo "" >> ~/.ssh/id_ed25519
- chmod 600 ~/.ssh/id_ed25519
- ssh-keygen -y -f ~/.ssh/id_ed25519 > /dev/null && echo "key OK" || echo "key BROKEN"
```

Последняя строка — диагностика: `ssh-keygen -y` пытается вывести публичный ключ из приватного. Получилось — ключ читается. Вывод перенаправлен в `/dev/null`, чтобы сам ключ не попал в лог.

**Verification.** В логе появилось `key OK`, подключение прошло.

---

## 4. Пустой адрес сервера

**Problem.** SSH не может определить, куда подключаться.

**Expected behavior.** Подключение к `172.17.0.1`.

**Observed behavior.**

```
ssh: Could not resolve hostname : Name does not resolve
```

**Investigation.** Между словом `hostname` и двоеточием — **пустота**. SSH получил пустое имя хоста.

Команда была:

```yaml
- ssh ... Tigran@$DEPLOY_HOST "..."
```

Развернулась в `Tigran@` — адрес отсутствовал.

**Evidence.** Пустое место в сообщении об ошибке. Диагностический приём:

```yaml
- echo "DEPLOY_HOST=[$DEPLOY_HOST]"
```

Квадратные скобки показывают границы значения — пустая переменная выводится как `[]`.

**Root cause.** Та же причина, что в сбое 1: переменная `DEPLOY_HOST` помечена **Protected**, рабочая ветка не защищена.

**Fix.** Галка Protected снята.

**Verification.**

```
Warning: Permanently added '172.17.0.1' (ED25519) to the list of known hosts.
connected
intern-srv-l2
```

---

## 5. Приложение падает после развёртывания

**Problem.** Контейнер запускается и сразу останавливается.

**Expected behavior.** Приложение работает, отвечает на `/health`.

**Observed behavior.**

```
$ docker ps -a | grep training-app
registry.gitlab.com/.../traning-app-ci:89da2f6b   Exited (1)   training-app
```

Задача `deploy` при этом **прошла успешно** — `docker run -d` вернул ID контейнера и завершился с кодом 0.

**Investigation.**

```
$ docker logs training-app
psycopg2.OperationalError: could not translate host name "postgres" to address:
Name or service not known
```

**Evidence.** Формулировка ошибки отличается от сбоя 1: там было `Connection refused` (имя резолвилось, порт не отвечал), здесь `could not translate host name` — имя не разрешается вовсе.

Команда запуска:

```bash
docker run -d --name training-app -p 8081:9000 $IMAGE_NAME:$IMAGE_TAG
```

Ни одного флага `-e`, ни `--network`.

**Root cause.** Две причины сразу.

**Переменные окружения не переданы.** Приложение взяло значения по умолчанию из кода: `DB_HOST=postgres`, пустой пароль.

**Контейнер не в той сети.** Запущен без `--network`, попал в дефолтный `bridge`. База живёт в сети Compose `traning-app-ci_default`, где работает встроенный DNS. В дефолтной сети имя `postgres` не резолвится.

**Fix.**

```bash
docker run -d --name training-app \
  --network traning-app-ci_default \
  -p 8081:9000 \
  -e DB_HOST=postgres -e DB_NAME=appdb -e DB_USER=appuser \
  -e DB_PASSWORD=$DEPLOY_DB_PASSWORD -e APP_ENV=training \
  $IMAGE_NAME:$IMAGE_TAG
```

Имя сети определено командой:

```bash
docker inspect traning-app-ci-postgres-1 | grep -A5 Networks
```

**Verification.**

```
$ docker ps -a | grep training-app
registry.gitlab.com/.../traning-app-ci:87610780   Up About a minute   0.0.0.0:8081->9000/tcp

$ curl http://localhost:8081/health
Application: OK
Database: OK
```

**Отдельный вывод.** Этот сбой показал, зачем нужен пункт 15 задания: задача `deploy` считалась успешной при неработающем приложении. `docker run -d` возвращает 0, как только контейнер создан, — что происходит внутри дальше, ему безразлично.

Добавлена проверка:

```yaml
- sleep 5
- ssh ... "curl -f http://localhost:8081/health"
```

Теперь такой сбой роняет задачу.

---

## 6. Опечатка в команде

**Problem.** Задача `deploy` падает после успешного развёртывания.

**Expected behavior.** Пауза, затем проверка health.

**Observed behavior.**

```
registry.gitlab.com/tikokhlghatyan/traning-app-ci:22dc8f08
training-app
968cec07b20c8aa4e3a18a202069edf9591f36f9f51f5633a7891c2c4cd0ff42
/bin/sh: eval: line 203: sleep5: not found
$ sleep5
ERROR: Job failed: exit code 127
```

**Investigation.** Развёртывание прошло полностью — образ вытянут, контейнер запущен, ID выведен. Падение на следующей команде.

**Evidence.** `sleep5: not found` и **код 127** — стандартный код «команда не найдена».

**Root cause.** Пропущен пробел: `sleep5` вместо `sleep 5`. Оболочка ищет команду с таким именем.

**Fix.**

```bash
sed -i 's/- sleep5/- sleep 5/' .gitlab-ci.yml
```

**Verification.** Пайплайн зелёный, health-check выполнился.

**Замечание.** Самая простая из шести ошибок и единственная, не связанная с пониманием системы. Но она дошла до сервера, потому что YAML не проверяет содержимое команд — синтаксически файл был корректен.

Отсюда практика: код возврата 127 всегда означает опечатку в имени команды или отсутствующий пакет, искать надо именно там.

---

## Сводка

| # | Симптом | Причина | Где искать в следующий раз |
|---|---|---|---|
| 1 | `Connection refused` на порту БД | Protected-переменная пуста в feature-ветке | настройки переменных, `${#VAR}` |
| 2 | `cp: unrecognized option: ---BEGIN` | тип переменной Variable вместо File | настройки переменной, поле Type |
| 3 | `error in libcrypto: unsupported` | нет завершающего перевода строки в ключе | `ssh-keygen -y -f` для проверки |
| 4 | `Could not resolve hostname :` | Protected-переменная пуста | пустота в сообщении об ошибке |
| 5 | контейнер `Exited (1)` при зелёной задаче | нет `--network` и `-e` | `docker logs`, отсутствие health-check |
| 6 | `exit code 127` | опечатка в имени команды | сама команда |

---

## Три вида ошибок подключения

За время работы встретились все три. По тексту сразу видна стадия сбоя:

| Сообщение | Что произошло | Куда смотреть |
|---|---|---|
| `could not translate host name` | контейнера нет или он в другой сети — DNS-записи не существует | `--network`, запущен ли сервис |
| `Connection refused` | контейнер работает, но сервис внутри не слушает порт | инициализация сервиса, логи контейнера |
| `password authentication failed` | сервис отвечает, отвергает учётные данные | переменные, состояние тома с базой |

Различение экономит время: по первой строке понятно, на каком уровне искать.

---

## Общие приёмы

**Диагностика снизу вверх.** От симптома к причине, каждый шаг — измерение, а не догадка. В сбое 1 между настройкой галки и наблюдаемым симптомом было пять ступеней.

**Точная формулировка ошибки — половина диагноза.** Пустое место в `Could not resolve hostname :` указывало на пустую переменную, а не на проблему с сетью.

**Фоновые процессы теряют вывод.** `команда &` скрывает причину падения. Перенаправление в файл и вывод его до следующего шага делает причину видимой.

**Проверка длины вместо значения.** `${#VAR}` показывает, пришла ли переменная, не раскрывая секрет.

**Успешная команда не означает работающий сервис.** `docker run -d` возвращает 0, как только контейнер создан. Нужна отдельная проверка результата.

**Код возврата говорит о типе проблемы.** 127 — команда не найдена, 22 — curl получил ошибку HTTP, 255 — ошибка SSH, 1 — общий сбой.
