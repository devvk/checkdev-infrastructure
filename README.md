# CheckDev Infrastructure

Инфраструктурный проект для локального запуска микросервисов CheckDev с использованием Docker Compose.

## Требования

- Docker Desktop
- Docker Compose
- Java 21
- Maven

Проверить установку Docker Compose:

```bash
docker compose version
```

## Подготовка

Перед запуском необходимо собрать каждый микросервис:

```bash
mvn clean package
```

В корне проекта `checkdev-infrastructure` создайте файл `.env`:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password

HH_TOKEN=your_hh_token

TG_USERNAME=your_bot_username
TG_TOKEN=your_bot_token

KEYCLOAK_CLIENT_SECRET=your_client_secret
```

## Запуск

Собрать Docker-образы:

```bash
docker compose build
```

Запустить все сервисы:

```bash
docker compose up -d
```

Проверить состояние контейнеров:

```bash
docker compose ps
```

## Настройка Keycloak

После первого запуска откройте административную консоль:

http://localhost:8081

Войдите:

```
Username: admin
Password: admin
```

### Создание Realm

Создайте Realm:

```
checkdev
```

### Создание Client

Создайте Client:

```
Client ID:
notification
```

В разделе **Capability config** включите:

- Client authentication — On
- Service account roles — On

После создания клиента откройте вкладку **Credentials**, скопируйте **Client Secret** и добавьте его в `.env`:

```env
KEYCLOAK_CLIENT_SECRET=your_client_secret
```

После изменения `.env` пересоздайте сервис notification:

```bash
docker compose up -d --force-recreate notification
```

## Проверка авторизации

Получить Service Token:

```bash
TOKEN=$(curl -s -X POST http://localhost:8081/realms/checkdev/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=notification" \
  -d "client_secret=$KEYCLOAK_CLIENT_SECRET" \
  | jq -r '.access_token')
```

Проверить доступ к защищенному endpoint:

```bash
curl -i http://localhost:9900/profiles/tg/1 \
  -H "Authorization: Bearer $TOKEN"
```

Ожидаемый результат:

```
HTTP/1.1 200 OK
```

Проверка без токена:

```bash
curl -i http://localhost:9900/profiles/tg/1
```

Ожидаемый результат:

```
HTTP/1.1 401 Unauthorized
```

## Просмотр логов

Все сервисы:

```bash
docker compose logs -f
```

Логи отдельного сервиса:

```bash
docker compose logs -f site
```

## Доступ к приложениям

- Site: http://localhost:8080
- Eureka: http://localhost:9009
- Keycloak: http://localhost:8081

## Остановка

Остановить и удалить контейнеры:

```bash
docker compose down
```

При обычной остановке данные PostgreSQL и Keycloak сохраняются в Docker Volumes.

Для удаления контейнеров вместе со всеми данными:

```bash
docker compose down -v
```

## Проверка сохранения данных

1. Запустить проект.
2. Создать данные в приложении.
3. Выполнить:

```bash
docker compose down
docker compose up -d
```

4. Убедиться, что:
    - данные PostgreSQL сохранились;
    - Realm `checkdev` сохранился;
    - Client `notification` сохранился.