# Memrizr

**Memrizr** — учебный backend-проект на Go, в котором реализован REST API для управления учётными записями пользователей: регистрация, аутентификация, обновление профиля и работа с аватарами.

Проект демонстрирует построение многослойного HTTP-приложения, работу с PostgreSQL, Redis и Google Cloud Storage, JWT-аутентификацию и модульное тестирование с использованием моков.

## Технологии

| Технология | Назначение |
| --- | --- |
| Go, Gin | HTTP API |
| PostgreSQL, sqlx | Хранение пользователей и выполнение SQL-запросов |
| Redis | Учёт действующих refresh-токенов и управление сессиями |
| JWT (RS256 / HS256) | ID- и refresh-токены |
| Google Cloud Storage | Хранение изображений профиля |
| Docker Compose, Traefik | Локальное окружение и маршрутизация |
| golang-migrate | Версионирование схемы БД |
| testify, моки | Модульное тестирование |

## Возможности

- Регистрация пользователя и вход по email и паролю.
- Хеширование паролей перед сохранением в БД.
- Выдача пары ID- и refresh-токенов.
- Обновление пары токенов с отзывом использованного refresh-токена.
- Выход из аккаунта с удалением refresh-токенов пользователя из Redis.
- Получение и редактирование данных профиля.
- Загрузка и удаление аватара через Google Cloud Storage (JPEG/PNG).
- Валидация входных данных, ограничение размера загружаемого файла и middleware для авторизации и тайм-аутов.

## Архитектура

Приложение разделено на три основных слоя:

```text
HTTP-запрос
    |
    v
handler/       HTTP-маршруты, валидация, middleware, ответы
    |
    v
service/       Бизнес-логика, пользователи, пароли и токены
    |
    v
repository/    Доступ к PostgreSQL, Redis и Google Cloud Storage
```

Контракты между слоями объявлены интерфейсами в `model/interface.go`. Конкретные реализации связываются при запуске в `account/injection.go` через внедрение зависимостей. Такой подход позволяет изолированно тестировать обработчики и сервисы с помощью моков.

```text
account/
├── handler/           # HTTP-обработчики и middleware
├── model/             # Модели, интерфейсы, ошибки и моки
├── repository/        # Реализации хранилищ
├── service/           # Бизнес-логика
├── migrations/        # SQL-миграции
├── data_sources.go    # Инициализация внешних зависимостей
├── injection.go       # Сборка зависимостей приложения
└── main.go            # Запуск HTTP-сервера и graceful shutdown
```

## HTTP API

Префикс маршрутов задаётся переменной `ACCOUNT_API_URL` (например, `/api/account`).

| Метод | Маршрут | Описание | Авторизация |
| --- | --- | --- | --- |
| POST | `/signup` | Регистрация | Нет |
| POST | `/signin` | Вход | Нет |
| POST | `/tokens` | Обновление токенов | Refresh-токен в теле |
| GET | `/me` | Получение профиля | ID-токен |
| POST | `/signout` | Выход из аккаунта | ID-токен |
| PUT | `/details` | Изменение данных профиля | ID-токен |
| POST | `/image` | Загрузка аватара | ID-токен |
| DELETE | `/image` | Удаление аватара | ID-токен |

**Пример регистрации:**

```http
POST /api/account/signup
Content-Type: application/json

{"email":"user@example.com","password":"strongpassword"}
```

При успешной регистрации сервер возвращает HTTP `201` и объект `tokens` с полями `idToken` и `refreshToken`.

**Пример запроса к защищённому маршруту:**

```http
GET /api/account/me
Authorization: Bearer <idToken>
```

**Пример обновления токенов:**

```http
POST /api/account/tokens
Content-Type: application/json

{"refreshToken":"<refreshToken>"}
```

Для загрузки изображения используйте `multipart/form-data` с полем `imageFile`.

## Аутентификация и управление сессиями

- **ID-токен** подписывается RSA-ключом по алгоритму **RS256** и проверяется публичным ключом. В claims включены данные пользователя, время выпуска и срок действия.
- **Refresh-токен** подписывается по алгоритму **HS256**. В claims находятся UUID пользователя, уникальный идентификатор токена (`jti`) и временные метки.
- В Redis сохраняется ключ вида `<userID>:<tokenID>` с TTL. При обновлении токенов предыдущий ключ удаляется, после чего создаётся новая пара.
- При выходе из аккаунта приложение удаляет refresh-токены пользователя из Redis.

## Локальный запуск

**Требования:** Docker с Compose, OpenSSL, `golang-migrate` для применения миграций, а также настроенный доступ к Google Cloud Storage и существующий bucket.

1. Клонируйте репозиторий:

   ```bash
   git clone https://github.com/dabudidabudai97/memrizr.git
   cd memrizr
   ```

2. Сгенерируйте RSA-ключи:

   ```bash
   make create-keypair ENV=dev
   ```

3. Создайте файл `account/.env.dev` и заполните переменные окружения. Пример конфигурации:

   ```dotenv
   PG_HOST=postgres-account
   PG_PORT=5432
   PG_USER=postgres
   PG_PASSWORD=password
   PG_DB=postgres
   PG_SSL=disable
   REDIS_HOST=redis-account
   REDIS_PORT=6379
   GC_IMAGE_BUCKET=your-gcs-bucket
   PRIV_KEY_FILE=./rsa_private_dev.pem
   PUB_KEY_FILE=./rsa_public_dev.pem
   REFRESH_SECRET=replace-with-a-long-random-secret
   ID_TOKEN_EXP=900
   REFRESH_TOKEN_EXP=604800
   ACCOUNT_API_URL=/api/account
   HANDLER_TIMEOUT=10
   MAX_BODY_BYTES=5242880
   GOOGLE_APPLICATION_CREDENTIALS=/go/src/app/serviceAccount.json
   ```

   Значения выше — **пример локальной конфигурации**, а не готовые production-настройки. Для Google Cloud Storage потребуется файл учётных данных `account/serviceAccount.json` либо другой поддерживаемый способ аутентификации Google Cloud. Не публикуйте ключи и секреты в Git.

4. Запустите PostgreSQL и Redis:

   ```bash
   docker compose up -d postgres-account redis-account
   ```

5. Примените миграции (команда использует PostgreSQL на `localhost:5432`):

   ```bash
   make migrate-up
   ```

6. Запустите приложение и Traefik:

   ```bash
   docker compose up --build
   ```

   В текущей конфигурации Traefik маршрутизирует запросы с заголовком `Host: malcorp.test` и префиксом `/api/account`. Для локального тестирования можно отправлять запросы на `http://localhost` с заголовком `Host: malcorp.test` либо настроить соответствующую запись в `hosts`.

   **Примечание:** `docker-compose.yml` использует development-режим с Reflex и открытой панелью Traefik. Это окружение не предназначено для развёртывания в production без дополнительной настройки безопасности.

## Тестирование

В репозитории присутствуют unit-тесты обработчиков и сервисов. Зависимости подменяются моками, что позволяет проверять логику без подключения реальных хранилищ.

```bash
cd account
go test ./...
```

Для отчёта о покрытии:

```bash
go test ./... -cover
```

## Что демонстрирует проект

- Разделение ответственности между HTTP-, сервисным и инфраструктурным слоями.
- Применение интерфейсов и Dependency Injection.
- Работу с реляционной БД, key-value-хранилищем и облачным объектным хранилищем.
- Реализацию JWT-аутентификации, ротации refresh-токенов и отзыва сессий.
- Unit-тестирование с использованием моков.
- Настройку локальной инфраструктуры через Docker Compose и SQL-миграции.

## Статус

Учебный проект и часть портфолио Go backend-разработчика. Код и конфигурация предназначены прежде всего для демонстрации реализованных подходов и дальнейшего развития проекта.
