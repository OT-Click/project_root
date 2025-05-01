## Feature: [[F01] Молниеносный отклик через Telegram‑бота](https://github.com/OT-Click/project_root/issues/47)

_**_As a_** пользователь, **_I want to_** пересылать вакансии боту, **_So that_** система автоматически обрабатывала их и отправляла отклики._

### Background

- **_Given_** пользователь авторизован в Telegram‑боте

### Scenario: [[S01.01] Пересылка вакансии и получение тикета](https://github.com/OT-Click/project_root/issues/42)

- **_When_** пользователь пересылает сообщение‑вакансию из публичного канала
- **_Then_** система распознает, что сообщение переслано из другого канала
- **_And_** бот отвечает тикетом с уникальным ID вакансии
- **_And_** статус вакансии устанавливается в «New»

### Scenario: [[S01.02] Автопарсинг вакансии ADK‑агентами](https://github.com/OT-Click/project_root/issues/43)

- **_Given_** вакансия находится в статусе «New»
- **_When_** очередь vacancy.raw отправляет документ в сервис ingest
- **_Then_** заполняется не менее 80% полей Vacancy
- **_And_** статус вакансии меняется на «Parsed»

### Scenario: [[S01.03] Матчинг и выбор релевантного CV](https://github.com/OT-Click/project_root/issues/44)

- **_Given_** вакансия находится в статусе «Parsed»
- **_When_** сервис matcher рассчитывает TF‑IDF/BM25 оценки
- **_Then_** выбирается одно CV со score не менее 0.55
- **_And_** статус вакансии меняется на «Matched»

### Scenario: [[S01.04] Отправка отклика от имени пользователя](https://github.com/OT-Click/project_root/issues/45)

- **_Given_** вакансия находится в статусе «Matched»
- **_When_** сервис sender вызывает Bot API sendMessage и sendDocument в ответ на исходное сообщение
- **_Then_** в чате появляется отклик с вложенным PDF‑резюме
- **_And_** статус вакансии меняется на «Sent»

### Scenario: [[S01.05] Отображение статуса «Sent» в канбане](https://github.com/OT-Click/project_root/issues/46)

- **_Given_** Mini App открыта пользователем
- **_And_** WebSocket‑соединение установлено
- **_When_** статус вакансии меняется на «Sent»
- **_Then_** карточка вакансии перемещается в колонку «Отправлено» в течение не более 15 с

---- 