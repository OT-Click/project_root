## Feature: [[F07] Отправка отклика по Email](https://github.com/OT-Click/project_root/issues/64)

_**_As a_** система, **_I want to_** отправлять отклик по email, если в тексте вакансии найден адрес, а chat_id отсутствует, **_So that_** можно было увеличить охват вакансий без Telegram‑контактов._

### Background

- **_Given_** вакансия находится в статусе «Matched»
- **_And_** поле `contact_email` заполнено

### Scenario: [[S07.01] Email‑отклик при отсутствии Telegram‑контакта](https://github.com/OT-Click/project_root/issues/63)

- **_When_** сервис sender обнаруживает, что `vacancy.contact_email` не пуст
- **_And_** `vacancy.telegram_chat_id` отсутствует
- **_Then_** сервис sender формирует письмо с PDF‑CV и текстом сопроводительного письма
- **_And_** отправляет его через SMTP‑шлюз
- **_And_** статус вакансии меняется на «Sent» с признаком `channel="email"`
- **_And_** событие `vacancy.sent` транслируется в RabbitMQ
- **_And_** карточка в канбане перемещается в колонку «Отправлено»

---- 