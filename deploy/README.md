# Продакшен-деплой (docker compose + Traefik)

Каталог `deploy/` — production-стек. Дев-окружение в `docker/` не трогаем, оно для локальной разработки.

Файлы не содержат секретов и площадко-специфичных значений: домен, пароли и параметры Traefik берутся из переменных окружения (см. `.env.example`). Compose падает с понятной ошибкой, если обязательная переменная не задана.

## Как это устроено

- Образ собирается на сервере из канонического `images/custom/Containerfile` репозитория [frappe_docker](https://github.com/frappe/frappe_docker) (контекст сборки — их git-репа, запинена на коммит `380b9d0`).
- Приложение `crm` ставится из этого репозитория, ветка указана в `apps.json`. **В контейнер попадает то, что запушено в GitHub**, а не локальный checkout: перед деплоем — `git push`.
- Стек: gunicorn-backend, nginx-frontend, socketio-websocket, два воркера очередей, scheduler, MariaDB 10.11, два Redis. Один общий образ `frappe-crm`.
- Наружу смотрит только `frontend` — через внешнюю docker-сеть Traefik (`TRAEFIK_NETWORK`, по умолчанию `proxy`) и лейблы (entrypoint и certresolver тоже настраиваются через env). Портов на хост нет.

## Настройка стека в Komodo

1. Repo: этот репозиторий, ветка из `apps.json`.
2. Путь compose-файла: `deploy/compose.yaml`.
3. Environment: задать `CRM_SITE_NAME` (домен), `CRM_DB_ROOT_PASSWORD`, `CRM_ADMIN_PASSWORD`.
4. Включить сборку при деплое (build before deploy).
5. DNS: A-запись домена на сервер с Traefik.

Первый деплой: `create-site` дождётся БД и создаст сайт `CRM_SITE_NAME` с установленным приложением `crm` (логин Administrator, пароль из `CRM_ADMIN_PASSWORD`). Повторные деплои сайт не пересоздают.

## Обновление кода

1. Запушить изменения в ветку из `apps.json`.
2. Поднять `CRM_CACHE_BUST` (иначе Docker переиспользует закешированный слой с кодом).
3. Redeploy в Komodo.
4. После обновления с миграциями выполнить в контейнере backend: `bench --site <CRM_SITE_NAME> migrate`.

## Заметки

- `frappe` пинится на `version-15` — приложение требует `frappe~=15.0.0` (см. `pyproject.toml`); Python 3.11 и Node 20 подобраны под него.
- Имя сайта Frappe = домену (`CRM_SITE_NAME`), поэтому nginx (`FRAPPE_SITE_NAME_HEADER`) и Traefik-роутер настраиваются одной переменной.
