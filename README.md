# Training App — GitLab CI/CD with Docker Runner

Автоматический пайплайн: push в GitLab запускает тесты, собирает Docker-образ, публикует его в реестр и разворачивает на целевом сервере. Без ручного подключения по SSH.

```
git push → test → build → registry → deploy → health check
```

---

## Быстрый старт

```bash
git clone git@gitlab.com:tikokhlghatyan/traning-app-ci.git
cd traning-app-ci
cp .env.example .env        # заполнить реальными значениями
docker compose up -d
curl http://localhost:8080/health
```

Пайплайн запускается сам при push в любую ветку.

---

## Структура

```
.
├── .gitlab-ci.yml          пайплайн: test → build → deploy
├── Dockerfile              сборка образа приложения
├── src/
│   └── app.py              HTTP-сервер на http.server + psycopg2
├── tests/
│   └── test_app.py         три теста на pytest
├── nginx/
│   └── default.conf        reverse proxy для локального запуска
├── docker-compose.yml      локальная среда: nginx + app + postgres
├── docs/
│   ├── architecture.md     какая машина что делает
│   ├── runner.md           Docker executor, эксперименты
│   ├── variables.md        путь переменной от GitLab до приложения
│   └── troubleshooting.md  разбор шести сбоев
├── .env.example            структура переменных без значений
└── .gitignore
```

---

## Приложение

HTTP-сервер на стандартной библиотеке Python (`http.server`), без фреймворка. Два эндпоинта:

| Путь | Ответ | Код |
|---|---|---|
| `GET /` | `Hello from app` + счётчик визитов из БД | 200 |
| `GET /health` | статус приложения и базы раздельно | 200 |
| любой другой | `Not Found` | 404 |

Конфигурация читается из переменных окружения:

```python
db_host = os.environ.get("DB_HOST", "postgres")
db_name = os.environ.get("DB_NAME", "appdb")
db_user = os.environ.get("DB_USER", "appuser")
db_password = os.environ.get("DB_PASSWORD", "")
```

У пароля дефолт пустой намеренно — реальное значение в коде оказалось бы в git.

При старте выполняется `init_db()`, создающая таблицу `visits`, если её нет.

---

## Пайплайн

### Этапы

```yaml
stages:
  - test
  - build
  - deploy
```

Порядок задаёт барьер: провал этапа останавливает следующие. Проверено — при упавшем тесте `build` и `deploy` получают статус **Skipped**, сломанный код физически не доходит до сервера.

### Задачи

| Задача | Этап | Образ | Что делает |
|---|---|---|---|
| `runner-info` | test | docker:27 | диагностика окружения задачи |
| `run-tests` | test | python:3.12-slim | поднимает приложение и прогоняет pytest |
| `build-image` | build | docker:27 | собирает образ, публикует в реестр, отдаёт тег через dotenv |
| `check-vars` | deploy | docker:27 | демонстрация передачи данных между задачами |
| `deploy` | deploy | alpine:latest | разворачивает образ на сервере по SSH |

Все задачи помечены `tags: - docker-build` — это направляет их на собственный раннер с Docker executor.

### Триггер

Пайплайн запускается **автоматически при каждом push**. Отдельной настройки не требуется: GitLab реагирует на наличие `.gitlab-ci.yml` в корне репозитория.

---

## Docker-образ

**Сборка** идёт внутри контейнера `docker:dind`, поднятого как сервис рядом с задачей.

**Тег — короткий SHA коммита:**

```bash
docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
```

Полное имя получается вида:

```
registry.gitlab.com/tikokhlghatyan/traning-app-ci:87610780
```

### Почему не `latest`

**Старый образ теряет имя.** При пересборке Docker снимает тег с предыдущего образа и вешает на новый. Предыдущий остаётся на диске безымянным — в `docker images` он виден как `<none>`. Обратиться к нему можно только по ID, который надо было записать заранее.

**Версия перестаёт быть определённой.** Ответ «в проде latest» не означает ничего. Тег `87610780` открывается в GitLab как конкретный коммит.

**Обманчивая идентичность.** Два сервера могут держать `latest` с разным содержимым — один тянул образ вчера, другой сегодня.

**Откат становится невозможен.** Проверено на практике: см. раздел Rollback.

**Хранилище** — GitLab Container Registry, встроенный в проект. Доступен по адресу из `CI_REGISTRY_IMAGE`.

---

## Развёртывание

### Схема

```
GitLab
   │ задача deploy
   ▼
контейнер задачи (alpine)
   │ SSH
   ▼
сервер intern-srv-l2
   │ docker pull
   ▼
Container Registry
```

Задача выполняется в изолированном контейнере и не имеет доступа к докер-демону сервера. Подключение идёт по SSH, а все команды Docker выполняются уже на целевой машине.

### Команды на сервере

```bash
docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
docker pull $IMAGE_NAME:$IMAGE_TAG
docker rm -f training-app || true
docker run -d --name training-app \
  --network traning-app-ci_default \
  -p 8081:9000 \
  -e DB_HOST=postgres -e DB_NAME=appdb -e DB_USER=appuser \
  -e DB_PASSWORD=$DEPLOY_DB_PASSWORD -e APP_ENV=training \
  $IMAGE_NAME:$IMAGE_TAG
```

**`|| true` у `docker rm`** — при первом развёртывании контейнера ещё нет, и команда вернула бы ошибку, оборвав цепочку `&&`.

**`--network traning-app-ci_default`** обязателен: имя `postgres` резолвится только внутри сети, созданной Compose. Без этого флага контейнер попадает в дефолтный `bridge`, где встроенного DNS нет.

**Переменные разворачиваются на стороне задачи**, до отправки строки по SSH.

### Проверка

```bash
sleep 5
ssh ... "curl -f http://localhost:8081/health"
```

Флаг `-f` заставляет curl вернуть ненулевой код при ответе 4xx/5xx. Без проверки пайплайн считал бы успешным развёртывание контейнера, который запустился и тут же упал — именно это и происходило до того, как были добавлены переменные окружения.

---

## Rollback

Образы в реестре неизменяемы и адресуются по тегу, поэтому откат сводится к запуску предыдущей версии.

**Проверено практикой.** Развёрнут был `22dc8f08`; возврат к `87610780`:

```bash
docker rm -f training-app
docker run -d --name training-app --network traning-app-ci_default -p 8081:9000 \
  -e DB_HOST=postgres -e DB_NAME=appdb -e DB_USER=appuser -e DB_PASSWORD=<пароль> \
  registry.gitlab.com/tikokhlghatyan/traning-app-ci:87610780
```

Результат:

```
$ docker ps --format "{{.Image}}" | grep traning
registry.gitlab.com/tikokhlghatyan/traning-app-ci:87610780

$ curl http://localhost:8081/health
Application: OK
Database: OK
```

**Пересборка не понадобилась.** Ни `git revert`, ни прогона пайплайна — образ уже лежал в реестре под своим тегом.

С `latest` это было бы невозможно: предыдущий образ потерял бы имя.

**Ограничение.** Откат кода не откатывает базу. Если новая версия применила необратимую миграцию, возврат к старому образу приведёт к несовместимости схемы. Обратимость миграций — отдельное требование, которое это задание не покрывает.

---

## Переменные и секреты

Подробный разбор — в `docs/variables.md`. Коротко:

| Где | Что | Пример |
|---|---|---|
| `.gitlab-ci.yml` | конфигурация тестовой среды, не секреты | `POSTGRES_PASSWORD: testpass` |
| GitLab CI/CD Variables | секреты и адреса | `SSH_PRIVATE_KEY`, `DEPLOY_DB_PASSWORD`, `DEPLOY_HOST` |
| Предопределённые GitLab | учётные данные реестра | `CI_REGISTRY_PASSWORD` |
| dotenv artifact | данные между задачами | `IMAGE_TAG`, `IMAGE_NAME` |
| `docker run -e` | конфигурация приложения | `DB_HOST`, `APP_ENV` |

**В репозитории секретов нет.** `.gitignore` создан до первого коммита — попавший в историю git секрет остаётся в старых коммитах, и удаление требует переписывания истории.

**Пароль тестовой базы задан прямо в пайплайне намеренно.** База создаётся на минуту внутри изолированной сети задачи, порт наружу не проброшен, после завершения уничтожается. Этим паролем нельзя воспользоваться — объекта уже не существует. Настоящие секреты живут в настройках проекта.

---

## Команды

**Локальная разработка:**

```bash
docker compose up -d --build
docker compose logs app --tail 20
curl http://localhost:8080/health
python3 -m pytest tests/ -v
```

**Состояние развёрнутого приложения:**

```bash
docker ps -a | grep training-app
docker logs training-app
docker ps --format "{{.Image}}" | grep traning    # какая версия работает
curl http://localhost:8081/health
```

**Диагностика раннеров:**

```bash
sudo gitlab-runner list
sudo systemctl status gitlab-runner
sudo journalctl -u gitlab-runner -n 30 --no-pager
```

**Реестр:**

```bash
docker login registry.gitlab.com
docker pull registry.gitlab.com/tikokhlghatyan/traning-app-ci:<TAG>
```

---

## Документация

- [`docs/architecture.md`](docs/architecture.md) — три машины схемы, кому что нужно
- [`docs/runner.md`](docs/runner.md) — Docker executor, эксперименты с окружением задачи
- [`docs/variables.md`](docs/variables.md) — путь переменной от GitLab до приложения
- [`docs/troubleshooting.md`](docs/troubleshooting.md) — шесть сбоев с разбором
