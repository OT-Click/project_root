# SRD: Системные требования и Архитектура

Этот документ описывает техническую архитектуру, компоненты, хранилища данных и инфраструктурные аспекты продукта, **начиная с MVP и этапы последующего развития**.

## 1. Архитектура

### 1.1 Компоненты (MVP)
| ID | Сервис | Стек | Очереди / API | Ответственность (MVP) |
|----|--------|------|---------------|-----------------------|
| S1 | **bot-svc** | FastAPI · aiogram | `POST /telegram/webhook` → RabbitMQ `vacancy.raw` | Получение сообщений, генерация тикетов, базовый rate-limit, отправка сообщений |
| S2 | **ingest** | FastStream · spaCy (базовый) | sub: `vacancy.raw` → pub: `vacancy.parsed` | Базовое NLP-обогащение вакансии, запись в Mongo. |
| S3 | **matcher** | FastAPI · scikit-learn (базовый) | sub: `vacancy.parsed` → pub: `vacancy.matched` | Базовый выбор CV (e.g., первое CV пользователя из S5), запись match в Mongo. |
| S4 | **file-uploader** | FastAPI | `POST /upload` (требует auth от S5) | Прием PDF, передача в S5 для сохранения. |
| S5 | **monolith-gateway** | FastAPI · Supabase Client | API Gateway; Auth (см. 1.3); Users/CV (Supabase DB); Files (Supabase Storage via S4); → pub/sub RabbitMQ | Ядро MVP: Управление пользователями, CV, файлами; Простая авторизация; Взаимодействие с другими сервисами через RabbitMQ. |

**Post-MVP Развитие Компонентов:**
- **S1 (bot-svc):** Улучшение логики, передача ответственности за отправку сообщений сервису S6.
- **S2 (ingest):** Улучшение NLP (spaCy/ADK/LLM).
- **S3 (matcher):** Внедрение TF-IDF/BM25.
- **S4 (file-uploader):** Замена на более функциональную версию, интегрированную с выделенным MinIO S3.
- **S5 (monolith-gateway):** **Вывод из эксплуатации.** Функциональность заменяется новыми сервисами.
- **Добавление новых сервисов:**
    - **S6 (`sender-svc`):** Выделенный сервис для отправки откликов (Telegram/Email), получает ответственность от S1.
    - **S7 (`billing-gateway`):** Выделенный сервис для обработки вебхуков платежных систем.
    - **S8 (`subscription-core`):** Выделенный сервис для управления планами и подписками.
- **Интеграция Безопасности:** Замена простой MVP-аутентификации на Keycloak (OIDC/JWT) и внедрение API Gateway (Kong) перед всеми сервисами (S1-S4, S6-S8).

### 1.2 Шина сообщений (RabbitMQ - MVP)
- **Обменники (Exchanges):**
    - `vacancy` (Type: Topic): Маршрутизация сообщений `vacancy.raw`, `vacancy.parsed`, `vacancy.matched`.
    - *Обменники `subscription.*` и `files` (fanout) - Post-MVP.*
- **Очереди (Queues):**
    - Стандартные очереди для MVP. *Quorum Queues и Sharding Plugin - Post-MVP для HA.*
- **Протокол:** AMQP.

### 1.3 Сетевой периметр и Безопасность (MVP)
- **API Gateway:** Функции выполняет `monolith-gateway` (S5).
- **Аутентификация/Авторизация:**
    - **MVP:** Супер-простая схема "логин/пароль" **без шифрования паролей** в БД Supabase (через S5). **КРАЙНЕ НЕБЕЗОПАСНО! ИСКЛЮЧИТЕЛЬНО ДЛЯ ЛОКАЛЬНЫХ ТЕСТОВ И ДЕМОНСТРАЦИИ MVP.**
    - **Post-MVP:** Переход на Keycloak (OIDC) и JWT. Внедрение Kong Gateway.
- **Внутренние коммуникации:** Через RabbitMQ (AMQP).
- **Шифрование:** HTTPS для Supabase API и Storage. *Обязательный HTTPS/TLS 1.3 для всех внешних API - Post-MVP (при внедрении Kong).*

---

## 2. Хранилище данных (MVP)
| Слой | Технология | Что храним (MVP) | Паттерн |
|------|------------|-----------------|---------|
| **Документы** | MongoDB | `vacancy`, `match` | - |
| **Бэкенд-сервис** | Supabase (PostgreSQL) | `users`, `cv` (метаданные), базовая информация для Auth | BaaS |
| **Файловое хранилище** | Supabase (Storage - S3 совместимый) | PDF CV (через S4/S5) | BaaS |

**Post-MVP Развитие Хранилища:**
- Миграция бизнес-логики (Billing, Subscriptions, etc.) на выделенный кластер PostgreSQL (Patroni + pgBouncer) с паттерном Database-per-Service.
- Миграция файлов на выделенный кластер MinIO S3.
- Внедрение MongoDB Sharded Cluster.

---

## 3. Бэкапы и Point-in-Time Recovery (MVP)
- **Supabase:** Используются встроенные механизмы резервного копирования Supabase.
- **MongoDB:** Настройка базовых бэкапов (e.g., `mongodump` по расписанию).

**Post-MVP Развитие Бэкапов:**
- Внедрение `wal-g` и `pgBackRest` для гранулярного PITR выделенного PostgreSQL.
- Автоматизация тестов восстановления.
- Определение и внедрение строгой Retention Policy.

---

## 4. Высокая доступность (HA) и масштабирование (MVP)
- **Зависимость от Supabase:** HA PostgreSQL и Storage обеспечивается платформой Supabase.
- **RabbitMQ/MongoDB:** Для MVP может использоваться single-node инсталляция (локально/staging). Масштабирование через увеличение ресурсов инстанса.
- **Микросервисы (S1-S5):** Базовое масштабирование через увеличение числа реплик (e.g., Docker Swarm, простой Kubernetes).

**Post-MVP Развитие HA:**
- Внедрение кластеров RabbitMQ (Quorum), MongoDB (Sharded), PostgreSQL (Patroni), MinIO (Distributed Erasure Coding) как описано в изначальном плане.
- Проведение тестов отказоустойчивости.

---

## 5. DevOps и Инфраструктура (MVP)
| Задача | Инструмент / Подход (MVP) |
|--------|---------------------------|
| CI/CD | GitHub Actions (Lint, Test, Build Docker) → Docker Hub/GHCR → Docker Compose / Простой деплой на VPS |
| Оркестрация | Docker Compose (локально, staging) |
| Мониторинг | Базовый мониторинг инстансов VPS/сервисов. *Prometheus/Grafana - Post-MVP.* |
| Трейсинг | *Не реализуется в MVP (OpenTelemetry/Jaeger - Post-MVP).* |
| Логирование | Стандартный вывод логов контейнеров. *Централизованное логирование (Loki/EFK) - Post-MVP.* |
| Управление секретами | Переменные окружения / GitHub Secrets. *Vault/Sealed Secrets - Post-MVP.* |
| Управление конфигурациями | Переменные окружения. |
| Бэкапы (Автоматизация) | Ручной запуск / Простые скрипты `mongodump`. *Автоматизация через CronJobs/K8s CronJobs - Post-MVP.* |

**Post-MVP Развитие DevOps:**
- Переход на Kubernetes (Helm) для оркестрации.
- Внедрение полного стека Observability (Prometheus, Grafana, Jaeger, Loki).
- Внедрение Vault/Sealed Secrets.
- Автоматизация бэкапов через CronJobs.

---

## 6. Открытые технические вопросы (Post-MVP)
1. Оптимальная стратегия retention для `wal-g` и `pgBackRest` (30 дней vs 90)?
2. Необходимость read‑replica PostgreSQL для будущих BI‑отчётов?
3. Целесообразность выноса Keycloak в отдельный кластер?
4. Стабильность конкретной версии RabbitMQ Sharding Plugin на целевой версии RabbitMQ?
5. Требуемый SLA доступности (99.9% или 99.95%) и его влияние на архитектуру HA. 