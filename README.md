[RU](#1-быстрый-старт)

## 1. Quick Start

Copy `.env.example` to `.env` and fill in the required values:

```bash
cp .env.example .env
```

Run the setup script (creates required files and starts all services):

```bash
./up.sh up -d
```

Or manually:

```bash
docker compose up -d
```

On first boot, the system will:
1. Pull/build all images
2. Initialize the Chatwoot database (schema + migrations)
3. Create a seed admin user with credentials from `CW_SEED_EMAIL` / `CW_SEED_PASSWORD`
4. Write the access token to a shared volume
5. Optionally create a test inbox in Chatwoot (if `GW_CREATE_TEST_INBOX` is set)
6. Start the Gateway with the auto-obtained token

On subsequent boots, steps 2–4 are skipped (token persists on a Docker volume).

### Stopping

```bash
docker compose down
```

To wipe all data (databases, tokens) and start fresh:

```bash
docker compose down -v
```

### Accessing the UI

- **Chatwoot**: http://localhost:8080 (or whatever `CW_PORT` is set to)
- **Gateway API**: http://localhost:8000 (or whatever `GW_PORT` is set to)

Login with `CW_SEED_EMAIL` / `CW_SEED_PASSWORD`.

## 2. Architecture

The compose file runs 7 services across 3 isolated networks:

| Service | Description | Network |
|---------|-------------|---------|
| **cw-rails** | Chatwoot web server | chatwoot-net, shared |
| **cw-sidekiq** | Chatwoot background jobs | chatwoot-net, shared |
| **cw-postgres** | PostgreSQL for Chatwoot | chatwoot-net |
| **cw-redis** | Redis for Chatwoot | chatwoot-net |
| **gw-init** | Init container: waits for token, optionally creates inbox | shared |
| **gw-app** | Channel Gateway application | gateway-net, shared |
| **gw-db** | PostgreSQL with PGMQ for Gateway | gateway-net |

Services use `extends` from source project compose files (`chatwoot/docker-compose.dev.yml`, `gateway/docker-compose.dev.yml`) to inherit image definitions.

## 3. Environment Variables

Variables marked with **required** will cause `docker compose` to fail immediately if missing or empty.

### Chatwoot

| Variable | Required | Description |
|----------|----------|-------------|
| `CW_SECRET_KEY_BASE` | **yes** | Secret key for cookie signing. Generate with `rake secret`. Alphanumeric only. |
| `CW_FRONTEND_URL` | **yes** | Public URL of the Chatwoot instance (e.g. `http://0.0.0.0:8080`). |
| `CW_PORT` | no | Host port for Chatwoot web UI. Default: `8080`. |
| `CW_POSTGRES_USER` | **yes** | PostgreSQL username. |
| `CW_POSTGRES_PASSWORD` | **yes** | PostgreSQL password. |
| `CW_POSTGRES_DB` | **yes** | PostgreSQL database name. |
| `CW_POSTGRES_PORT` | no | Host port exposed for Chatwoot Postgres (for debugging). Default: `5432`. |
| `CW_REDIS_PASSWORD` | **yes** | Password for the Redis instance. Can be any string. |
| `CW_REDIS_PORT` | no | Host port exposed for Chatwoot Redis. Default: `6379`. |
| `CW_SEED_EMAIL` | **yes** | Email for the admin user created on first boot. |
| `CW_SEED_PASSWORD` | **yes** | Password for the admin user created on first boot. |
| `CW_MAILER_SENDER_EMAIL` | no | Sender email for outgoing mail. Format: `email@domain.com` or `Name <email@domain.com>`. |
| `CW_SMTP_ADDRESS` | no | SMTP server address. If empty, Chatwoot uses sendmail. |
| `CW_SMTP_PORT` | no | SMTP port. Default: `587`. |
| `CW_SMTP_USERNAME` | no | SMTP username. |
| `CW_SMTP_PASSWORD` | no | SMTP password. |

### Gateway

| Variable | Required | Description |
|----------|----------|-------------|
| `GW_PORT` | no | Host port for Gateway API. Default: `8000`. |
| `GW_POSTGRES_VERSION` | no | PostgreSQL major version for the PGMQ image. Default: `16`. |
| `GW_PGMQ_VERSION` | no | PGMQ image tag. Default: `v1.9.0`. |
| `GW_DB_USER` | **yes** | PostgreSQL username for Gateway. |
| `GW_DB_PASS` | **yes** | PostgreSQL password for Gateway. |
| `GW_DB_NAME` | **yes** | PostgreSQL database name for Gateway. |
| `GW_POSTGRES_PORT` | no | Host port for Gateway Postgres. Default: `5433`. |
| `GW_CHATWOOT_ACCESS_TOKEN` | no | If set, overrides the auto-obtained token. Get from Chatwoot → Profile Settings → Access Token. |
| `GW_CREATE_TEST_INBOX` | no | Set to any value to create a "Test Email Gateway" inbox on first boot. Leave empty to skip. |
| `GW_WH_DOMAIN` | no | Public domain for webhook URLs. Default: `http://localhost:8000`. |
| `GW_BOTS_CONFIG` | no | Telegram bot config (JSON array). See Gateway README for format. |
| `GW_SECRET_TOKEN` | **yes** | Secret token for webhook header verification. |
| `GW_MAILBOXES_CONFIG` | no | Email adapter config (JSON array). See Gateway README for format. |
| `GW_INCOMING_QUEUE_NAME` | no | PGMQ queue name for incoming messages. Default: `to_cw`. |
| `GW_OUTGOING_QUEUE_NAME` | no | PGMQ queue name for outgoing messages. Default: `from_cw`. |
| `GW_GROUP` | no | Plugin auto-discovery group. Default: `helpline.channel_adapters`. |
| `GW_ENVIRONMENT` | no | Environment mode: `LOCAL`, `DEV`, `STAGE`, `PROD`. Default: `LOCAL`. |
| `GW_WORKERS` | no | Number of worker processes. `0` or `1` = single process. Default: `1`. |
| `GW_LOG_LEVEL` | no | Log level: `DEBUG`, `INFO`, `WARNING`, `ERROR`. Default: `INFO`. |
| `GW_ANONYMIZE_USERS` | no | Generate random usernames. Default: `True`. |
| `GW_OTEL_ENABLED` | no | Enable OpenTelemetry. Default: `false`. |
| `GW_OTEL_SERVICE_NAME` | no | OTEL service name. Default: `gateway`. |
| `GW_OTEL_ENDPOINT` | no | OTLP endpoint. Leave empty to send traces only to Sentry. |
| `GW_SENTRY_DSN` | no | Sentry DSN for error reporting. |

### Extends Resolution Variables

These are defined at the bottom of `.env` and exist solely to satisfy variable references in the source compose files during `extends` resolution. **Do not modify** unless you know what you're doing.

| Variable | Maps to |
|----------|---------|
| `POSTGRES_VERSION` | `${GW_POSTGRES_VERSION}` |
| `PGMQ_VERSION` | `${GW_PGMQ_VERSION}` |
| `DB_USER` | `${GW_DB_USER}` |

## 4. Files

| File | Description |
|------|-------------|
| `docker-compose.yml` | Main compose file — runs Chatwoot + Gateway together |
| `.env` | Environment variables (gitignored) |
| `.env.example` | Template with all variables and descriptions |
| `up.sh` | Wrapper script — creates required files and runs `docker compose` |
| `cloud-init.yml` | Hetzner cloud-init config for provisioning a dev VM |

## 5. Troubleshooting

**"database does not exist" error on first boot** — This is normal. `db:chatwoot_prepare` creates the database automatically. The error appears briefly before creation.

**Onboarding screen appears** — Run `docker compose down -v` to wipe volumes and restart. The seed script deletes the onboarding flag.

**gw-init exits with error** — Check if Chatwoot is healthy. The init container retries automatically, but if cw-rails fails to start, gw-init will eventually time out.

**Port conflict** — Change `CW_POSTGRES_PORT`, `CW_REDIS_PORT`, `GW_POSTGRES_PORT` in `.env` if local services are using the default ports.

**Fresh start** — `docker compose down -v` removes all volumes (databases, token file). Next `up` will re-run the full first-boot flow.

---

[EN](#1-quick-start)

## 1. Быстрый старт

Скопируйте `.env.example` в `.env` и заполните обязательные значения:

```bash
cp .env.example .env
```

Запустите через скрипт (создаёт необходимые файлы и поднимает все сервисы):

```bash
./up.sh up -d
```

Или вручную:

```bash
docker compose up -d
```

При первом запуске система:
1. Скачает/соберёт все образы
2. Инициализирует базу данных Chatwoot (схема + миграции)
3. Создаст администратора с учётными данными из `CW_SEED_EMAIL` / `CW_SEED_PASSWORD`
4. Запишет access token на общий том
5. Опционально создаст тестовый inbox в Chatwoot (если задан `GW_CREATE_TEST_INBOX`)
6. Запустит Gateway с автоматически полученным токеном

При повторных запусках шаги 2–4 пропускаются (токен сохраняется на Docker-томе).

### Остановка

```bash
docker compose down
```

Для полного сброса данных (базы, токены):

```bash
docker compose down -v
```

### Доступ к интерфейсу

- **Chatwoot**: http://localhost:8080 (или значение `CW_PORT`)
- **Gateway API**: http://localhost:8000 (или значение `GW_PORT`)

Логин: `CW_SEED_EMAIL` / `CW_SEED_PASSWORD`.

## 2. Архитектура

Compose-файл запускает 7 сервисов в 3 изолированных сетях:

| Сервис | Описание | Сеть |
|--------|----------|------|
| **cw-rails** | Веб-сервер Chatwoot | chatwoot-net, shared |
| **cw-sidekiq** | Фоновые задачи Chatwoot | chatwoot-net, shared |
| **cw-postgres** | PostgreSQL для Chatwoot | chatwoot-net |
| **cw-redis** | Redis для Chatwoot | chatwoot-net |
| **gw-init** | Init-контейнер: ждёт токен, опционально создаёт inbox | shared |
| **gw-app** | Приложение Channel Gateway | gateway-net, shared |
| **gw-db** | PostgreSQL с PGMQ для Gateway | gateway-net |

Сервисы используют `extends` из compose-файлов исходных проектов (`chatwoot/docker-compose.dev.yml`, `gateway/docker-compose.dev.yml`) для наследования определений образов.

## 3. Переменные окружения

Переменные, отмеченные как **обязательные**, приведут к ошибке `docker compose` если не заданы или пусты.

### Chatwoot

| Переменная | Обязательная | Описание |
|------------|--------------|----------|
| `CW_SECRET_KEY_BASE` | **да** | Секретный ключ для подписи cookies. Сгенерируйте через `rake secret`. Только буквы и цифры. |
| `CW_FRONTEND_URL` | **да** | Публичный URL Chatwoot (например `http://0.0.0.0:8080`). |
| `CW_PORT` | нет | Порт хоста для веб-интерфейса Chatwoot. По умолчанию: `8080`. |
| `CW_POSTGRES_USER` | **да** | Имя пользователя PostgreSQL. |
| `CW_POSTGRES_PASSWORD` | **да** | Пароль PostgreSQL. |
| `CW_POSTGRES_DB` | **да** | Имя базы данных PostgreSQL. |
| `CW_POSTGRES_PORT` | нет | Порт хоста для PostgreSQL Chatwoot (для отладки). По умолчанию: `5432`. |
| `CW_REDIS_PASSWORD` | **да** | Пароль для Redis. Любая строка. |
| `CW_REDIS_PORT` | нет | Порт хоста для Redis Chatwoot. По умолчанию: `6379`. |
| `CW_SEED_EMAIL` | **да** | Email администратора, создаваемого при первом запуске. |
| `CW_SEED_PASSWORD` | **да** | Пароль администратора, создаваемого при первом запуске. |
| `CW_MAILER_SENDER_EMAIL` | нет | Email отправителя. Формат: `email@domain.com` или `Имя <email@domain.com>`. |
| `CW_SMTP_ADDRESS` | нет | Адрес SMTP-сервера. Если пусто — используется sendmail. |
| `CW_SMTP_PORT` | нет | Порт SMTP. По умолчанию: `587`. |
| `CW_SMTP_USERNAME` | нет | Имя пользователя SMTP. |
| `CW_SMTP_PASSWORD` | нет | Пароль SMTP. |

### Gateway

| Переменная | Обязательная | Описание |
|------------|--------------|----------|
| `GW_PORT` | нет | Порт хоста для Gateway API. По умолчанию: `8000`. |
| `GW_POSTGRES_VERSION` | нет | Мажорная версия PostgreSQL для образа PGMQ. По умолчанию: `16`. |
| `GW_PGMQ_VERSION` | нет | Тег образа PGMQ. По умолчанию: `v1.9.0`. |
| `GW_DB_USER` | **да** | Имя пользователя PostgreSQL для Gateway. |
| `GW_DB_PASS` | **да** | Пароль PostgreSQL для Gateway. |
| `GW_DB_NAME` | **да** | Имя базы данных PostgreSQL для Gateway. |
| `GW_POSTGRES_PORT` | нет | Порт хоста для PostgreSQL Gateway. По умолчанию: `5433`. |
| `GW_CHATWOOT_ACCESS_TOKEN` | нет | Если задан — переопределяет автоматически полученный токен. Берётся из Chatwoot → Настройки профиля → Access Token. |
| `GW_CREATE_TEST_INBOX` | нет | Любое значение — создаёт "Test Email Gateway" inbox при первом запуске. Пусто — пропускает. |
| `GW_WH_DOMAIN` | нет | Публичный домен для webhook URL. По умолчанию: `http://localhost:8000`. |
| `GW_BOTS_CONFIG` | нет | Конфигурация Telegram-ботов (JSON-массив). Формат см. в README Gateway. |
| `GW_SECRET_TOKEN` | **да** | Секретный токен для верификации заголовка вебхука. |
| `GW_MAILBOXES_CONFIG` | нет | Конфигурация email-адаптера (JSON-массив). Формат см. в README Gateway. |
| `GW_INCOMING_QUEUE_NAME` | нет | Имя очереди PGMQ для входящих сообщений. По умолчанию: `to_cw`. |
| `GW_OUTGOING_QUEUE_NAME` | нет | Имя очереди PGMQ для исходящих сообщений. По умолчанию: `from_cw`. |
| `GW_GROUP` | нет | Группа для авто-обнаружения плагинов. По умолчанию: `helpline.channel_adapters`. |
| `GW_ENVIRONMENT` | нет | Режим окружения: `LOCAL`, `DEV`, `STAGE`, `PROD`. По умолчанию: `LOCAL`. |
| `GW_WORKERS` | нет | Количество worker-процессов. `0` или `1` = один процесс. По умолчанию: `1`. |
| `GW_LOG_LEVEL` | нет | Уровень логирования: `DEBUG`, `INFO`, `WARNING`, `ERROR`. По умолчанию: `INFO`. |
| `GW_ANONYMIZE_USERS` | нет | Генерировать случайные имена пользователей. По умолчанию: `True`. |
| `GW_OTEL_ENABLED` | нет | Включить OpenTelemetry. По умолчанию: `false`. |
| `GW_OTEL_SERVICE_NAME` | нет | Имя сервиса OTEL. По умолчанию: `gateway`. |
| `GW_OTEL_ENDPOINT` | нет | OTLP endpoint. Пусто — трейсы только в Sentry. |
| `GW_SENTRY_DSN` | нет | Sentry DSN для отчётов об ошибках. |

### Переменные для разрешения extends

Определены в конце `.env`. Нужны для подстановки переменных в исходных compose-файлах при `extends`. **Не изменяйте**, если не уверены.

| Переменная | Соответствует |
|------------|---------------|
| `POSTGRES_VERSION` | `${GW_POSTGRES_VERSION}` |
| `PGMQ_VERSION` | `${GW_PGMQ_VERSION}` |
| `DB_USER` | `${GW_DB_USER}` |

## 4. Файлы

| Файл | Описание |
|------|----------|
| `docker-compose.yml` | Основной compose-файл — запускает Chatwoot + Gateway вместе |
| `.env` | Переменные окружения (в gitignore) |
| `.env.example` | Шаблон со всеми переменными и описаниями |
| `up.sh` | Скрипт-обёртка — создаёт необходимые файлы и запускает `docker compose` |
| `cloud-init.yml` | Cloud-init конфиг Hetzner для развёртывания VM разработчика |

## 5. Решение проблем

**Ошибка "database does not exist" при первом запуске** — Это нормально. `db:chatwoot_prepare` создаёт базу автоматически. Ошибка появляется на короткое время до создания.

**Появляется экран онбординга** — Выполните `docker compose down -v` для сброса томов и перезапустите. Скрипт удаляет флаг онбординга.

**gw-init завершается с ошибкой** — Проверьте, запустился ли Chatwoot. Init-контейнер повторяет попытки автоматически, но если cw-rails не стартует, gw-init в итоге упадёт.

**Конфликт портов** — Измените `CW_POSTGRES_PORT`, `CW_REDIS_PORT`, `GW_POSTGRES_PORT` в `.env`, если локальные сервисы используют стандартные порты.

**Полный сброс** — `docker compose down -v` удаляет все тома (базы данных, файл токена). Следующий `up` выполнит полный цикл первого запуска.
