# Детальный Roadmap MVP (Этапы 1-5)

**Цель MVP:** Создать минимально жизнеспособный продукт, демонстрирующий основной сценарий: Соискатель пересылает вакансию боту -> Система обрабатывает её (базово) -> Соискатель получает мок-отклик -> Соискатель видит статус в простом веб-интерфейсе и может загрузить CV. Используется упрощенная архитектура с Supabase и монолитом-гейтвеем.

---

## **Этап 1: Основы и Bot MVP (Локально)**

**Неделя 1: Настройка Проекта и Инициализация Bot Service**

*   **1.1 Создание Репозитория `bot-svc`:** [NS05.01]
    *   Создать репозиторий `bot-svc` на github.
    *   Склонирвоать репозитрий.
*   **1.4 Настройка Среды Разработчика:** [NS05.01]
    *   Инициализировать Python проект с `uv`: `uv init`.
    *   Создать `.vscode/settings.json` в `bot-svc` с настройками форматтеров (Ruff для Python), линтеров (Ruff, Pyright), указать путь к Python интерпретатору в `.venv`.
    *   Настроить Ruff (`pyproject.toml`) и Pyright (`pyproject.toml`) внутри `services/bot-svc`.
    *   Создать базовый `.gitignore`.
    *   Добавить `uv add aiogram faststream[cli] faststream[rabbit] motor`.
    *   Создать структуру: `src/`.
    *   Создать Минимальный файл с ботом aiogram
*   **1.5 Настройка Docker Compose (Локально, MVP):** [NS01.01]
    *   Создать `docker-compose.yml` в *корне рабочей области `bot-svc`* .
    *   Добавить сервисы: `rabbitmq` (стандартный образ, порты 5672, 15672), `mongodb` (стандартный образ, порты 27017).
    *   Настроить базовую сеть `bot-svc-network`.
*   **1.6 Настройка Переменных Окружения (Базовые):** [NS01.01]
    *   Создать `.env.example` в *корне рабочей области* с *минимальным* набором переменных: `TELEGRAM_BOT_TOKEN`, `MONGO_URI`, `RABBITMQ_URI` [NS05.01].

**Неделя 2-4: Разработка Bot Service (Локально)**

*   **1.7 Реализация Приема Сообщений:**
    *   Настроить `aiogram` Bot, Dispatcher в `bot-svc` [S01.01].
    *   Создать обработчик сообщений (`message_handler`), который ловит *все* сообщения [S01.01].
*   **1.8 Настройка Локального Webhook:**
    *   Создать FastAPI приложение (`main.py`) в `bot-svc` [S01.01].
    *   Реализовать эндпоинт `POST /telegram/webhook` для приема обновлений от Telegram [S01.01].
    *   Интегрировать `aiogram` с FastAPI (используя `Dispatcher.feed_webhook_update`) [S01.01].
    *   В `message_handler` добавить логику проверки `message.forward_from_chat` для определения пересланных сообщений [S01.01].
*   **1.9 Генерация Тикетов и Ответ:**
    *   Реализовать функцию генерации уникального ID (e.g., `uuid4`).
    *   При получении пересланного сообщения, генерировать ID и отвечать пользователю (`message.reply`) сообщением вида "Вакансия принята, ID: {ticket_id}" [S01.01].
*   **1.10 Сохранение в MongoDB (Базовое):**
    *   Настроить `motor` клиент для подключения к локальному MongoDB (`MONGO_URI`).
    *   При генерации тикета, сохранять документ в коллекцию `vacancies` с полями: `_id` (ticket_id), `user_id` (telegram user id), `chat_id`, `message_id`, `status: "New"`, `created_at` [S01.01].
*   **1.11 Mock Публикации в RabbitMQ:**
    *   Создать *функцию-заглушку* `publish_to_rabbitmq(queue_name, message)` которая просто логирует вызов (не использует `aio-pika` пока) [S01.01].
    *   Вызывать эту заглушку после сохранения в MongoDB, передавая `vacancy.raw` и сообщение (e.g., `{"ticket_id": ticket_id, "original_message": message.text}`).
*   **1.12 Обработка Ответов Рекрутеров (Mock):** [S03.01]
    *   В `message_handler` добавить проверку `message.reply_to_message`.
    *   Если это ответ на сообщение бота *и* `message.from_user.id` не совпадает с ID пользователя изначальной вакансии (нужно будет где-то временно хранить или доставать из Mongo), симулировать смену статуса.
*   **1.13 Смена Статуса на "Answered" (Mock):**
    *   При обнаружении ответа (1.12), *логировать* сообщение "Статус вакансии {ticket_id} изменен на Answered" [S03.01]. (Реальное обновление статуса в БД будет позже).
*   **1.14 Тестирование:** Написать базовые `pytest` unit-тесты для генерации ID, форматирования ответа, логики определения пересланного сообщения. Запускать локально через `ngrok` или аналоги для получения вебхуков от тестового бота [NS05.02].

---

## **Этап 2: Развертывание Bot Service и CI/CD**

**Неделя 5-6: CI/CD и Первый Деплой**

*   **2.1 Инфраструктура для Bot Service:**
    *   2.1.1 **Организация GitHub:** Проверить наличие, создать при необходимости [NS01.02].
    *   2.1.2 **Настройка CI для `bot-svc`:**
        *   Создать `.github/workflows/ci-bot-svc.yml`.
        *   Добавить шаги: Checkout, Setup Python + `uv`, Install dependencies (`uv sync`), Lint (`ruff check`), Type Check (`ruff format --check` + `pyright`), Tests (`pytest`), Build Docker image (`docker build`), Push Docker image (в GHCR) [NS01.02, NS05.02]. Триггер: push в `main`, PR в `main`.
    *   2.1.3 **Настройка CD для `bot-svc`:**
        *   Создать `.github/workflows/deploy-bot-svc.yml`.
        *   Триггер: `workflow_dispatch` с выбором окружения (staging) [NS01.02].
        *   Добавить шаги: SSH на staging VPS, `docker compose pull bot-svc`, `docker compose up -d --force-recreate bot-svc`.
    *   2.1.4 **Настройка Секретов:**
        *   Добавить `TELEGRAM_BOT_TOKEN`, `MONGO_URI`, `RABBITMQ_URI`, `VPS_SSH_HOST`, `VPS_SSH_USER`, `VPS_SSH_KEY` в секреты репозитория GitHub [NS04.03].
    *   2.1.5 **Подготовка Конфигурации Деплоя:** [NS01.01]
        *   Создать `infra/staging/docker-compose.yml`.
        *   Добавить сервисы `bot-svc`, `rabbitmq`, `mongodb`.
        *   Использовать `${VAR_NAME}` для переменных окружения, которые будут браться из `.env` файла на сервере.
        *   Создать `infra/staging/.env.example`.
*   **2.2 Деплой Bot Service (MVP):**
    *   2.2.1 **Подготовка Staging Сервера:** [NS01.02]
        *   Арендовать VPS (Hetzner CX11 или аналог).
        *   Установить Docker, Docker Compose. Настроить firewall (открыть 22, 80/443).
        *   Настроить SSH доступ по ключу.
        *   Скопировать `infra/staging/docker-compose.yml` и создать `.env` файл с реальными данными на сервере.
        *   Запустить `docker compose up -d rabbitmq mongodb`. Проверить их работу.
    *   2.2.2 **Деплой и Настройка Webhook:**
        *   Запустить `deploy-bot-svc.yml` workflow для Staging. Проверить логи `bot-svc`.
        *   Настроить DNS A-запись для домена/поддомена на IP сервера (если есть домен).
        *   Настроить Nginx/Caddy/Traefik как реверс-прокси для `bot-svc` (маппинг порта, SSL/TLS с Let's Encrypt).
        *   Установить Webhook Telegram бота на `https://<your-domain>/telegram/webhook` [S01.01]. Проверить через API Telegram (`getWebhookInfo`).
        *   Отправить тестовое сообщение боту, проверить ответ и логи.
    *   2.2.3 **Проверка Подключений:**
        *   Убедиться, что `bot-svc` успешно подключается к RabbitMQ и MongoDB на сервере (проверить логи).
    *   2.2.4 **Реальная Публикация в RabbitMQ:**
        *   Заменить mock-функцию `publish_to_rabbitmq` на реальную реализацию с `aio-pika`.
        *   Настроить подключение к RabbitMQ (`RABBITMQ_URI`).
        *   Объявить exchange `vacancy` (topic) и очередь `vacancy.raw`. Связать их с routing key `vacancy.raw`.
        *   Публиковать сообщения в exchange `vacancy` с routing key `vacancy.raw` [S01.01, S01.02].
        *   Обновить код `bot-svc` и запустить CD пайплайн. Проверить публикацию через RabbitMQ Management UI.

---

## **Этап 3: Ingest Service MVP**

**Неделя 7-8: Базовый Парсинг и Интеграция**

*   **3.1 Ingest Service (Разработка):**
    *   3.1.1 **Создание Репозитория `ingest-svc`:** [NS05.01]
        *   Аналогично `bot-svc`: папка `services/ingest-svc`, `uv init`, структура `src/`, `tests/`, `Dockerfile`, `requirements.txt`.
        *   Добавить `uv add faststream motor pydantic spacy[ru]`. Скачать модель spaCy (`python -m spacy download ru_core_news_sm`).
    *   3.1.2 **Подписка на RabbitMQ:**
        *   Настроить `FastStream` приложение.
        *   Создать подписчика (`@broker.subscriber`) на очередь `vacancy.raw`, привязанную к exchange `vacancy` [S01.02].
    *   3.1.3 **Базовый Парсинг:**
        *   В обработчике подписчика: загрузить модель spaCy.
        *   Реализовать *простые* правила извлечения:
            *   Использовать spaCy для токенизации и POS-теггинга.
            *   Искать email/телефон с помощью регулярных выражений.
            *   Искать ключевые слова (Python, Java, Golang, FastAPI, SQL, etc.).
            *   *Не реализовывать сложный NER или классификацию пока.* [Частично S01.02].
    *   3.1.4 **Извлечение Контактов:**
        *   Сохранить найденные `contact_email` и `telegram_chat_id` (если удастся извлечь) в переменные [S01.02, S07.01].
    *   3.1.5 **Обновление в MongoDB:**
        *   Подключиться к MongoDB.
        *   Найти документ `vacancies` по `ticket_id` из сообщения.
        *   Обновить документ, добавив извлеченные поля (`parsed_keywords`, `contact_email`, `telegram_chat_id`) [S01.02].
    *   3.1.6 **Установка Статуса "Parsed":**
        *   Обновить поле `status` на "Parsed" в документе MongoDB [S01.02].
    *   3.1.7 **Публикация в RabbitMQ:**
        *   Объявить очередь `vacancy.parsed` и связать с exchange `vacancy`.
        *   Опубликовать *обогащенное* сообщение (включая `ticket_id` и извлеченные данные) в exchange `vacancy` с routing key `vacancy.parsed` [S01.02, S01.03].
    *   **Тестирование:** Unit-тесты для парсинга (на примерах текста вакансий), тесты интеграции с RabbitMQ (локально) [NS05.02].
*   **3.2 CI/CD и Деплой Ingest Service:**
    *   3.2.1 **Настройка CI/CD для `ingest-svc`:**
        *   Создать `.github/workflows/ci-ingest-svc.yml` и `deploy-ingest-svc.yml` по аналогии с `bot-svc` [NS01.02]. Учесть установку spaCy модели в Dockerfile.
    *   3.2.2 **Деплой:**
        *   Добавить `ingest-svc` в `infra/staging/docker-compose.yml`.
        *   Добавить переменные окружения для `ingest-svc` в `.env` на сервере.
        *   Запустить CD пайплайн для `ingest-svc`. Проверить логи и обработку сообщений из `vacancy.raw`.

---

## **Этап 4: Matcher MVP и Базовая Отправка**

**Неделя 9-10: Соединение MVP Флоу**

*   **4.1 Matcher Service (Разработка):**
    *   4.1.1 **Создание Репозитория `matcher-svc`:** [NS05.01]
        *   Аналогично: папка `services/matcher-svc`, `uv init`, структура, `Dockerfile`, `requirements.txt`.
        *   Добавить `uv add fastapi uvicorn motor pydantic httpx`.
    *   4.1.2 **Подписка на RabbitMQ:**
        *   Настроить FastAPI приложение (или FastStream, если не нужен HTTP API).
        *   Создать подписчика на очередь `vacancy.parsed` [S01.03].
    *   4.1.3 **Базовый Матчинг (Mock):**
        *   В обработчике: извлечь `ticket_id` и `user_id`.
        *   Сделать HTTP запрос к `monolith-gateway` (S5) (его API нужно будет создать в Этапе 5) на эндпоинт типа `GET /users/{user_id}/cvs?limit=1` для получения ID первого CV пользователя.
        *   *Захардкодить ID тестового CV, если S5 еще не готов.* [Частично S01.03].
    *   4.1.4 **Обновление в MongoDB:**
        *   Подключиться к MongoDB.
        *   Обновить документ `vacancies` по `ticket_id`, установить `status: "Matched"`, добавить `matched_cv_id` [S01.03].
        *   Создать новый документ в коллекции `matches` (`vacancy_id`, `cv_id`, `score: 0.0`, `matched_at`).
    *   4.1.5 **Публикация в RabbitMQ:**
        *   Объявить очередь `vacancy.matched` и связать с exchange `vacancy`.
        *   Опубликовать сообщение (`ticket_id`, `matched_cv_id`) в exchange `vacancy` с routing key `vacancy.matched` [S01.03, S01.04].
    *   **Тестирование:** Unit-тесты для логики матчинга (мок), интеграционные тесты с RabbitMQ [NS05.02].
*   **4.2 Логика Sender (в `bot-svc`):**
    *   4.2.2 **Подписка на RabbitMQ:**
        *   В `bot-svc` добавить подписчика (FastStream/aio-pika) на очередь `vacancy.matched` [S01.04, S07.01].
    *   4.2.3 **Отправка Mock Отклика:**
        *   В обработчике `vacancy.matched`: извлечь `ticket_id`.
        *   Найти исходное сообщение пользователя (по `chat_id`, `message_id` из `vacancies` в Mongo).
        *   Используя `aiogram`, отправить *текстовое* сообщение-ответ: "Отклик на вакансию {ticket_id} отправлен." [S01.04].
        *   *Не отправлять PDF пока.*
    *   4.2.4 **Обновление Статуса "Sent":**
        *   Обновить документ `vacancies` в MongoDB, установить `status: "Sent"` [S01.04, S01.05].
    *   4.2.5 **Публикация `vacancy.sent`:**
        *   Объявить очередь `vacancy.sent` (или использовать общую `vacancy.status_changed`).
        *   Опубликовать событие (`ticket_id`, `status: "Sent"`) в exchange `vacancy` [S07.01].
*   **4.3 CI/CD и Деплой:**
    *   4.3.1 **Настройка CI/CD для `matcher-svc`:** Аналогично другим сервисам [NS01.02].
    *   4.3.2 **Деплой:**
        *   Добавить `matcher-svc` в `infra/staging/docker-compose.yml`. Добавить переменные окружения.
        *   Запустить CD для `matcher-svc`.
        *   Обновить код `bot-svc` (с логикой Sender). Запустить CD для `bot-svc`.
        *   **Сквозное Тестирование:** Переслать вакансию боту -> Проверить статусы в MongoDB (New -> Parsed -> Matched -> Sent) -> Проверить сообщения в RabbitMQ -> Проверить ответное сообщение от бота [F01].

---

## **Этап 5: Frontend MVP (Kanban) и Завершение MVP Флоу**

**Неделя 11-12: Интерфейс и Загрузка Файлов**

*   **5.1 File Uploader Service (Базовый):**
    *   5.1.1 **Создание Репозитория `uploader-svc`:** [NS05.01]. `uv add fastapi uvicorn python-multipart`.
    *   5.1.2 **Реализация API `POST /upload`:**
        *   Создать FastAPI эндпоинт, принимающий `UploadFile` [S09.01].
        *   Добавить *заглушку* для проверки auth токена (передаваемого из S5) [NS04.01].
    *   5.1.3 **Интеграция с S5 для Сохранения:**
        *   Сделать HTTP-запрос к S5 (`monolith-gateway`) на новый эндпоинт `POST /files/upload-internal` (который нужно будет создать в S5), передавая файл и user_id (из auth токена) [S09.01]. S5 будет отвечать за загрузку в Supabase Storage.
    *   5.1.4 **Возврат URL:**
        *   Получить public URL файла от S5 и вернуть его клиенту [S09.01].
    *   5.1.5 **CI/CD и Деплой `uploader-svc`:** Настроить CI/CD, добавить в `docker-compose.yml`, развернуть [NS01.02].
*   **5.2 Monolith Gateway (S5 - Разработка):**
    *   **Создание Репозитория `monolith-gateway`:** [NS05.01]. Аналогично. `uv add fastapi uvicorn supabase python-multipart pydantic async-lru websockets`.
    *   **API Пользователей/CV:**
        *   Реализовать эндпоинты CRUD для `users` (профиль) и `cvs` (метаданные: id, user_id, filename, storage_url, uploaded_at). Использовать `supabase-py` клиент для взаимодействия с Supabase DB.
        *   Реализовать `GET /users/{user_id}/cvs` для `matcher-svc` [S01.03].
    *   **Простая Аутентификация:** [NS04.01]
        *   Реализовать `POST /auth/register` (сохраняет user, login, plain_password в Supabase).
        *   Реализовать `POST /auth/login` (проверяет plain_password, возвращает простой токен, e.g., user_id).
        *   Создать FastAPI `Depends` для проверки токена в защищенных эндпоинтах. **ПОМНИТЬ О НЕБЕЗОПАСНОСТИ!**
    *   **API Файлов:** [S09.01]
        *   Реализовать `POST /files/upload-internal` (для S4): получает файл, user_id; использует `supabase-py` для загрузки в Supabase Storage; сохраняет метаданные CV в Supabase DB; возвращает URL.
    *   **API Вакансий (Чтение):** [S02.01]
        *   Реализовать `GET /vacancies` (с фильтрацией по user_id, статусу): читает данные из MongoDB, возвращает список для фронтенда.
        *   Реализовать `GET /vacancies/{ticket_id}`: читает детали одной вакансии.
    *   **WebSocket Endpoint:** [S01.05, S02.01, S03.01]
        *   Реализовать FastAPI WebSocket эндпоинт `/ws/{user_id}`.
        *   Хранить активные соединения (e.g., в словаре).
        *   Создать подписчика RabbitMQ на `vacancy.sent` (и другие события смены статуса).
        *   При получении события из RabbitMQ, найти WebSocket соединение для соответствующего `user_id` и отправить обновление (e.g., JSON с `ticket_id` и новым `status`).
    *   **CI/CD и Деплой S5:** Настроить CI/CD, добавить в `docker-compose.yml`, развернуть [NS01.02].
*   **5.3 Frontend (SPA / Mini App):**
    *   5.2.1 **Создание Репозитория `frontend-app`:** [NS05.01]. Инициализировать проект (React/Vue/etc.), настроить `bun`.
    *   5.2.2 **Разработка Канбан UI:**
        *   Создать компоненты: `KanbanBoard`, `KanbanColumn` (New, Parsed, Matched, Sent, Answered), `VacancyCard` (отображает title/company) [S02.01].
        *   При загрузке, вызвать API S5 (`GET /vacancies`) для получения списка вакансий и распределить карточки по колонкам [S02.01].
    *   5.2.3 **Интеграция WebSocket:**
        *   Установить WebSocket соединение с S5 (`/ws/{user_id}`) после логина [S01.05, S02.01, S03.01].
        *   При получении сообщения по WebSocket, найти соответствующую карточку и переместить ее в нужную колонку.
    *   5.2.4 **Модальное Окно Деталей:**
        *   При клике на карточку, открывать модальное окно.
        *   Вызывать API S5 (`GET /vacancies/{ticket_id}`) для загрузки деталей и отображать их [Частично S02.04].
    *   5.2.5 **Интеграция Загрузки CV:**
        *   Создать страницу/раздел "Профиль" с формой загрузки PDF.
        *   Реализовать логику логина (`POST /auth/login` к S5), сохранить токен [NS04.01].
        *   При загрузке файла, отправлять его на `POST /upload` (S4), передавая токен. Отображать загруженные CV (вызов `GET /users/{user_id}/cvs` к S5) [S09.01].
    *   5.2.6 **CI/CD и Деплой Frontend:** [NS01.02]
        *   Настроить CI: `bun install`, `bun run build`.
        *   Настроить CD: Загрузка статических файлов на хостинг (Nginx на VPS, Netlify, Vercel). Настроить переменные окружения (API_BASE_URL, WS_BASE_URL).
*   **Финальное MVP Тестирование:** Проверить весь флоу от пересылки вакансии до отображения статуса "Sent" на канбане и загрузки CV [F01, F09].

---

## **Этап 6: Финализация Инфраструктуры и Корневой Репозиторий (Пост-MVP / Параллельно)**

*   **6.1 Настройка Корневого Репозитория:** [NS05.02]
    *   Создать главный репозиторий проекта (`project_root`) на GitHub.
    *   Инициализировать структуру папок (`docs`, `services`, `infra`) в корневом репозитории (может использовать Git Submodules или просто копирование/скрипты для сборки).
    *   Настроить `README.md` в корневом репозитории с описанием всего проекта и ссылками на сервисы.
    *   Настроить централизованную Wiki (`docs/wiki` или GitHub Wiki в корневом репозитории) для `PRD.md`, `SRD.md`, `plan.md`, `processes.md` и т.д. Перенести документацию.
    *   Настроить централизованную доску задач (GitHub Projects / Jira / Trello) для всего проекта.
    *   Собрать итоговый `docker-compose.yml` (или Helm charts/Kustomize) в `infra` корневого репозитория для централизованного деплоя всех сервисов MVP [NS01.01].
*   **6.2 Настройка CI/CD для Корневого Репозитория:** [NS01.02]
    *   Создать workflow для сборки и деплоя *всех* сервисов из корневого репозитория.

--- 