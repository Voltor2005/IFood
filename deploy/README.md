# Развёртывание backend «ЯЕда»

Одна команда на сервере: `bash deploy/deploy.sh`. Backend собирается в Docker, данные (SQLite) лежат в постоянном томе, порт 8080 открыт для Telegram-бота.

## Что нужно на сервере

Linux, Docker Engine с плагином Compose v2, `git`, `openssl`, `curl` и открытый порт (по умолчанию `8080/tcp`).

## Первый запуск

```bash
git clone <URL репозитория> yaeda && cd yaeda
bash deploy/deploy.sh
```

Скрипт создаст `.env` из `.env.example`, сгенерирует `BOT_API_KEY` (если он пуст), соберёт образ, запустит контейнер и дождётся статуса healthy. Проверка:

```bash
bash deploy/deploy.sh check          # или: curl http://<сервер>:8080/healthcheck
grep BOT_API_KEY .env                # ключ для бота; в открытый чат и git не отправлять
```

## Обновление

```bash
git pull && bash deploy/deploy.sh    # на сервере
bash deploy/deploy.sh remote         # со своей машины по SSH (настройка ниже)
```

## Команды

| Команда | Что делает |
|---|---|
| `up` (по умолчанию) | собрать и перезапустить все сервисы |
| `down` | остановить и удалить контейнеры, данные остаются |
| `restart` | перезапустить без пересборки |
| `status` | состояние контейнеров |
| `logs [сервис]` | логи |
| `check` | проверить `/healthcheck` |
| `remote` | выполнить `up` на сервере по SSH |

## Настройки (`.env`)

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `BOT_API_KEY` | генерируется | ключ бот → backend (заголовок `X-API-Key`) |
| `BACKEND_PORT` | `8080` | порт на хосте |
| `BACKEND_BIND` | `0.0.0.0` | `127.0.0.1`, если порт не нужен снаружи |

База лежит в томе `backend-data` (`/data/app.db`) и переживает перезапуск и пересборку. Команда `docker compose down -v` удалит том вместе с данными.

## Как подключается бот

- **В той же compose-сети**: адрес `http://backend:8080`, заголовок `X-API-Key: <BOT_API_KEY>`.
- **С другой машины**: `http://<сервер>:<BACKEND_PORT>`, на сервере порт должен быть открыт (`ufw allow 8080/tcp`).
- **HTTPS**: контракт требует его для рабочих вызовов, а сам backend отвечает по HTTP. Для боевого режима поставьте перед ним reverse proxy (Caddy или nginx) с сертификатом.

## Запуск бэкенда и Telegram-бота вместе

Сервис бота уже описан в `deploy/compose.d/bot.yml`. Скрипт запуска
подключает этот файл автоматически.

1. В корне проекта создай `.env` из `.env.example`, если файла ещё нет.
2. В корневом `.env` задай `BOT_TOKEN` и включи `COMPOSE_PROFILES=bot`.
   Оставь `BOT_API_KEY` пустым при первом запуске, скрипт сгенерирует общий ключ.
   Для размещения бота и сервера на одной машине укажи `BACKEND_BIND=127.0.0.1`.
3. Останови локальную копию Telegram-бота с тем же токеном и запусти:

   ```bash
   bash deploy/deploy.sh up
   bash deploy/deploy.sh check
   bash deploy/deploy.sh logs bot
   ```

Бот дождётся готовности бэкенда и будет обращаться к нему по адресу
`http://backend:8080` внутри Docker-сети. Для проверки пришли боту `/start`
в Telegram, затем выполни поиск и генерацию рецепта.

Чтобы запускать только бэкенд на чистой установке, оставь профиль бота выключенным.

## Доступ к серверу по SSH

Приватный ключ у каждого свой и никому не передаётся. В чат отправляется только публичный.

```bash
ssh-keygen -t ed25519 -C "ваше-имя@yaeda"
cat ~/.ssh/id_ed25519.pub            # Windows (PowerShell): type $env:USERPROFILE\.ssh\id_ed25519.pub
```

Тот, у кого есть доступ к серверу, добавляет присланный публичный ключ:

```bash
echo "ssh-ed25519 AAAA... имя@yaeda" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys
```

Удалённый деплой настраивается файлом `deploy/.deploy.env` (добавьте его в `.gitignore`):

```bash
DEPLOY_HOST=user@server
DEPLOY_DIR=/home/user/yaeda
DEPLOY_BRANCH=master
```

## Файлы

`Dockerfile`, `docker-compose.yml`, `.dockerignore` и `.env.example` лежат в корне репозитория, `deploy/deploy.sh` и `deploy/compose.d/` в папке `deploy/`.
