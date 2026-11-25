# Веб-сервер FastAPI

## Эндпоинты
* `GET /healthcheck` — Проверка статуса работоспособности сервера.

## Быстрый запуск
Для запуска проекта используется менеджер пакетов `uv`:

```bash
uv run main.py
```
Сервер будет доступен по адресу: http://localhost:8080/healthcheck


# Развёртывание backend «ЯЕда»

Инструкция по развёртыванию backend на виртуальной машине Linux с использованием Docker и Docker Compose.

## 1. Архитектура

Backend разворачивается на одной виртуальной машине в Yandex Cloud.

Используемые компоненты:

- **FastAPI** — HTTP API приложения.
- **Docker** — контейнеризация backend.
- **Docker Compose** — сборка и запуск контейнеров.
- **SQLite** — хранение данных приложения.
- **Дополнительный HDD** — постоянное хранение базы данных.

Структура развёртывания:

```text
Yandex Cloud VM
│
├── Docker Engine
│   │
│   └── Docker Compose
│       │
│       ├── backend
│       │   └── FastAPI :8080
│       │
│       └── yaeda network
│
└── /mnt/students-data/yaeda/
    ├── app.db
    ├── app.db-wal
    └── app.db-shm
```

Backend работает в Docker-контейнере и автоматически перезапускается после перезагрузки сервера.

Telegram-бот запускается как отдельный сервис в той же Docker-сети при включении профиля `bot`.

## 2. Требования к серверу

Для развёртывания необходимы:

- Linux (рекомендуется Ubuntu 24.04).
- Docker Engine.
- Docker Compose V2.
- Git.
- OpenSSL.
- curl.
- Дополнительный диск, смонтированный в `/mnt/students-data`.

Пользователь сервера должен иметь возможность запускать Docker.

## 3. Подготовка сервера

### Установка Docker

Если Docker ещё не установлен:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2 git openssl curl
```

Включение автоматического запуска Docker:

```bash
sudo systemctl enable --now docker
```

Добавление пользователя в группу Docker:

```bash
sudo usermod -aG docker $USER
```

После этого необходимо выйти из SSH-сессии и подключиться заново.

Проверка установки:

```bash
docker --version
docker compose version
```

### Подготовка диска

Дополнительный диск должен быть смонтирован в:

```text
/mnt/students-data
```

Создание директории для базы данных:

```bash
sudo mkdir -p /mnt/students-data/yaeda
sudo chown 10001:10001 /mnt/students-data/yaeda
sudo chmod 700 /mnt/students-data/yaeda
```

Контейнер запускается от пользователя с UID 10001, поэтому директория должна быть доступна для записи этому пользователю.

## 4. Первый запуск

Клонирование репозитория:

```bash
git clone https://github.com/zxcbebra67/IFood.git
cd IFood
```

Запуск приложения:

```bash
bash deploy/deploy.sh
```

Скрипт автоматически:

1. Проверяет наличие Docker и Docker Compose.
2. Создаёт `.env` из `.env.example`, если файл отсутствует.
3. Генерирует `BOT_API_KEY`, если он не задан.
4. Собирает Docker-образ backend.
5. Создаёт Docker-сеть.
6. Запускает контейнер.
7. Ожидает успешного прохождения healthcheck.

После успешного запуска контейнер должен получить статус `healthy`.

## 5. Проверка работоспособности

Проверка состояния контейнеров:

```bash
bash deploy/deploy.sh status
```

Проверка API:

```bash
bash deploy/deploy.sh check
```

Либо напрямую:

```bash
curl http://127.0.0.1:8080/healthcheck
```

Ожидаемый ответ:

```json
{"status":"ok"}
```

Просмотр логов:

```bash
bash deploy/deploy.sh logs backend
```

Просмотр последних 50 строк логов:

```bash
docker compose logs --tail=50 backend
```

## 6. Хранение данных

SQLite хранится на дополнительном HDD сервера.

Путь на сервере:

```text
/mnt/students-data/yaeda/app.db
```

Путь внутри контейнера:

```text
/data/app.db
```

В `docker-compose.yml` используется bind mount:

```yaml
volumes:
  - /mnt/students-data/yaeda:/data
```

Это означает, что база данных находится вне контейнера и сохраняется при:

- перезапуске контейнера;
- пересборке Docker-образа;
- пересоздании контейнера;
- обновлении приложения.

SQLite также может создавать файлы `app.db-wal` и `app.db-shm`. Они необходимы для работы базы в режиме WAL. Не следует удалять их вручную во время работы приложения.

**Важно:** удаление контейнера или выполнение `docker compose down` не удаляет базу данных. Однако удаление или форматирование самого HDD приведёт к потере данных.

Рекомендуется периодически создавать резервные копии базы данных.

## 7. Конфигурация приложения

Настройки хранятся в `.env` в корне проекта.

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `BOT_API_KEY` | Генерируется автоматически | Ключ авторизации запросов от бота |
| `BACKEND_PORT` | `8080` | Локальный порт сервера |
| `BACKEND_BIND` | `127.0.0.1` | Адрес публикации порта на сервере |

Пример `.env`:

```env
BOT_API_KEY=сгенерированный_секретный_ключ
BACKEND_PORT=8080
BACKEND_BIND=127.0.0.1
```

Файл `.env` содержит секретные данные и не должен попадать в Git.

Для проверки наличия ключа без вывода его значения:

```bash
grep -q '^BOT_API_KEY=.' .env \
  && echo "BOT_API_KEY задан"
```

Backend использует заголовок:

```http
X-API-Key: <BOT_API_KEY>
```

Не передавайте значение ключа через публичные чаты и не добавляйте его в репозиторий.

## 8. Сетевое взаимодействие

### Подключение из Docker-сети

Для сервисов, находящихся в одной Docker-сети `yaeda`, backend доступен по адресу:

```text
http://backend:8080
```

Публиковать порт backend во внешний интернет для этого не требуется.

### Подключение с другой машины

По умолчанию порт backend опубликован только на `127.0.0.1`.

Для внешнего подключения потребуется отдельно настроить безопасный доступ через HTTPS reverse proxy, например Caddy или Nginx.

Backend самостоятельно обслуживает HTTP-запросы. TLS и HTTPS должны обеспечиваться reverse proxy.

Не рекомендуется открывать порт 8080 всему интернету.

## 9. Подключение Telegram-бота

Скрипт `deploy/deploy.sh` автоматически подключает `deploy/compose.d/bot.yml`.
Для запуска бота укажи `BOT_TOKEN` и `COMPOSE_PROFILES=bot` в корневом `.env`.
Там же указывается `OPENROUTER_API_KEY` для генерации. Настройка прокси и порядок проверки описаны в [инструкции по развёртыванию](deploy/README.md).

```bash
bash deploy/deploy.sh up
```

## 10. Обновление приложения

Для обновления backend на сервере:

```bash
cd ~/IFood
git pull
bash deploy/deploy.sh up
```

Скрипт пересоберёт образ и пересоздаст контейнер при необходимости.

База данных при этом останется на дополнительном HDD.

### Удалённое обновление

Для запуска обновления со своего компьютера можно использовать:

```bash
bash deploy/deploy.sh remote
```

Для этого необходимо настроить файл `deploy/.deploy.env`:

```env
DEPLOY_HOST=ubuntu@адрес-сервера
DEPLOY_DIR=/home/ubuntu/IFood
DEPLOY_BRANCH=main
```

Файл `deploy/.deploy.env` необходимо добавить в `.gitignore`.

## 11. Команды управления

Все команды выполняются из корневой директории проекта.

| Команда | Назначение |
|---|---|
| `bash deploy/deploy.sh` | Сборка и запуск |
| `bash deploy/deploy.sh up` | Сборка и запуск |
| `bash deploy/deploy.sh down` | Остановка и удаление контейнеров |
| `bash deploy/deploy.sh restart` | Перезапуск контейнеров |
| `bash deploy/deploy.sh status` | Состояние контейнеров |
| `bash deploy/deploy.sh logs` | Просмотр логов |
| `bash deploy/deploy.sh logs backend` | Логи backend |
| `bash deploy/deploy.sh check` | Проверка доступности API |
| `bash deploy/deploy.sh remote` | Удалённое обновление |

## 12. SSH-доступ

Для подключения к серверу используется SSH-аутентификация по ключу.

Создание нового ключа:

```bash
ssh-keygen -t ed25519 -C "your-name@yaeda"
```

Публичный ключ находится в:

```text
~/.ssh/id_ed25519.pub
```

Приватный ключ нельзя передавать другим пользователям.

Для предоставления доступа другому разработчику необходимо добавить его публичный ключ в `~/.ssh/authorized_keys` пользователя сервера:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh

echo "ssh-ed25519 AAAA... your-name@yaeda" >> ~/.ssh/authorized_keys

chmod 600 ~/.ssh/authorized_keys
```

## 13. Структура проекта

```text
IFood/
├── main.py
├── db.py
├── search.py
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
├── .env.example
│
├── deploy/
│   ├── deploy.sh
│   ├── README.md
│   └── compose.d/
│
└── telegram_bot/
    └── ...
```

Основные файлы:

- `main.py` — FastAPI-приложение и API-эндпоинты.
- `db.py` — работа с SQLite.
- `search.py` — логика поиска.
- `Dockerfile` — сборка образа backend.
- `docker-compose.yml` — конфигурация сервисов.
- `.env.example` — пример переменных окружения.
- `deploy/deploy.sh` — автоматизация развёртывания.
- `deploy/compose.d/` — дополнительные сервисы Docker Compose.

---

**Примечание:** Terraform используется отдельно для создания и настройки инфраструктуры Yandex Cloud (VM, диски, сеть и правила доступа). Для обычного обновления backend запускать Terraform не требуется.

`Dockerfile`, `docker-compose.yml`, `.dockerignore` и `.env.example` лежат в корне репозитория, `deploy/deploy.sh` и `deploy/compose.d/` в папке `deploy/`.
