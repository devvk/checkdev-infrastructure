# CheckDev Infrastructure

Инфраструктурный проект для локального запуска микросервисов CheckDev с использованием Docker Compose.

## Требования

- Docker Desktop
- Docker Compose

Локальная установка Java и Maven для сборки и запуска системы через Docker не требуется.

Проверить установку Docker Compose:

```bash
docker compose version
```

## Подготовка

В корне проекта `checkdev-infrastructure` создайте файл `.env`:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password

HH_TOKEN=your_hh_token

TG_USERNAME=your_bot_username
TG_TOKEN=your_bot_token

KEYCLOAK_CLIENT_SECRET=your_client_secret
```

Файл `.env` содержит секреты и не должен добавляться в Git.

## Сборка и запуск

Микросервисы используют multi-stage Docker build.

На первом этапе Docker использует Maven + JDK 21:

```text
исходный код
    ↓
mvn verify
    ↓
компиляция + тесты + JAR
```

На втором этапе создаётся финальный runtime image на базе JRE 21:

```text
JRE 21
  +
app.jar
```

Поэтому предварительно выполнять `mvn package` на host-машине не требуется.

Собрать images и запустить всю систему:

```bash
docker compose up -d --build
```

Или отдельно выполнить сборку:

```bash
docker compose build
```

а затем запуск:

```bash
docker compose up -d
```

Проверить состояние контейнеров:

```bash
docker compose ps
```

## Сервисы

Docker Compose запускает:

- PostgreSQL
- Keycloak
- Kafka
- Kafka UI
- Eureka
- Auth
- Desc
- Generator
- Mock
- Site
- Notification

Все сервисы подключаются к общей Docker-сети и могут обращаться друг к другу по имени сервиса.

Например:

```text
postgres:5432
kafka:29092
eureka:9009
auth:9900
site:8080
```

## Docker Volumes

Для постоянного хранения данных используются named volumes:

```text
postgres_data
keycloak_data
kafka_data
```

Volumes существуют независимо от жизненного цикла контейнеров.

Поэтому после:

```bash
docker compose down
```

контейнеры удаляются, но данные сохраняются.

Посмотреть volumes:

```bash
docker volume ls
```

## Настройка Keycloak

После первого запуска откройте административную консоль:

```text
http://localhost:8081
```

Войдите:

```text
Username: admin
Password: admin
```

### Создание Realm

Создайте Realm:

```text
checkdev
```

### Создание Client

Создайте Client:

```text
Client ID: notification
```

В разделе **Capability config** включите:

- Client authentication — On
- Service account roles — On

После создания клиента откройте вкладку **Credentials**, скопируйте **Client Secret** и добавьте его в `.env`:

```env
KEYCLOAK_CLIENT_SECRET=your_client_secret
```

После изменения `.env` пересоздайте сервис `notification`:

```bash
docker compose up -d --force-recreate notification
```

## Проверка авторизации

Перед выполнением команды убедитесь, что переменная `KEYCLOAK_CLIENT_SECRET` доступна в текущем shell.

Например:

```bash
export KEYCLOAK_CLIENT_SECRET=your_client_secret
```

Получить Service Token:

```bash
TOKEN=$(curl -s -X POST http://localhost:8081/realms/checkdev/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=notification" \
  -d "client_secret=$KEYCLOAK_CLIENT_SECRET" \
  | jq -r '.access_token')
```

Проверить доступ к защищённому endpoint:

```bash
curl -i http://localhost:9900/profiles/tg/1 \
  -H "Authorization: Bearer $TOKEN"
```

Ожидаемый результат:

```text
HTTP/1.1 200 OK
```

Проверка без токена:

```bash
curl -i http://localhost:9900/profiles/tg/1
```

Ожидаемый результат:

```text
HTTP/1.1 401 Unauthorized
```

## Просмотр логов

Логи всех сервисов:

```bash
docker compose logs -f
```

Логи отдельного сервиса:

```bash
docker compose logs -f site
```

Например:

```bash
docker compose logs -f desc
docker compose logs -f auth
docker compose logs -f notification
```

## Доступ к приложениям

- Site: `http://localhost:8080`
- Eureka: `http://localhost:9009`
- Keycloak: `http://localhost:8081`
- Kafka UI: `http://localhost:8085`

## Остановка

Остановить и удалить контейнеры:

```bash
docker compose down
```

При обычном `docker compose down` named volumes не удаляются.

Данные PostgreSQL, Keycloak и Kafka сохраняются.

Для удаления контейнеров вместе с volumes:

```bash
docker compose down -v
```

> `docker compose down -v` удаляет данные из named volumes. Используйте эту команду только тогда, когда данные больше не нужны.

## Проверка сохранения данных

1. Запустить систему:

```bash
docker compose up -d --build
```

2. Создать данные в приложении.

3. Остановить и удалить контейнеры:

```bash
docker compose down
```

4. Повторно запустить:

```bash
docker compose up -d
```

5. Убедиться, что:

- данные PostgreSQL сохранились;
- Realm `checkdev` сохранился;
- Client `notification` сохранился;
- данные Kafka сохранились.

## Полезные команды

```bash
docker compose up -d --build     # собрать и запустить систему
docker compose down              # остановить и удалить контейнеры
docker compose ps                # состояние сервисов
docker compose logs -f           # следить за логами
docker compose logs -f site      # логи конкретного сервиса
docker compose restart site      # перезапустить сервис
docker compose config            # проверить Compose-конфигурацию

docker images                    # локальные images
docker volume ls                 # Docker volumes
docker stats                     # CPU/RAM контейнеров
docker system df                 # использование диска Docker
```
