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

Просмотреть логи всех сервисов:

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

## Остановка

Остановить и удалить контейнеры:

```bash
docker compose down
```

При обычной остановке данные PostgreSQL сохраняются в Docker Volume.

Для удаления контейнеров вместе с данными базы данных:

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

4. Убедиться, что данные сохранились после повторного запуска контейнеров.