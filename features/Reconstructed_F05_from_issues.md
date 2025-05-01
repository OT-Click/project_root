## Feature: [[F05] Управление платной подпиской](https://github.com/OT-Click/project_root/issues/60)

_**_As a_** зарегистрированный пользователь, **_I want to_** оформить и продлевать подписку через удобного платёжного провайдера, **_So that_** я мог получать расширенные возможности сервиса._

### Scenario: [[S05.01] Активация подписки после успешного платежа](https://github.com/OT-Click/project_root/issues/58)

- **_Given_** пользователь оформил план «PRO»
- **_When_** платёжный провайдер отправляет Web‑hook `subscription.renewed`
- **_Then_** статус подписки устанавливается «active»
- **_And_** `period_end` обновляется согласно плану

### Scenario Outline: [[S05.02] Обработка Web‑hook'ов разных платёжных систем](https://github.com/OT-Click/project_root/issues/59)

- **_Given_** событие `<event>` поступило от `<provider>`
- **_When_** billing‑gateway валидирует подпись Web‑hook'а
- **_Then_** создаётся запись в `payment_log` со статусом `<status>`
- **_And_** транслируется событие `subscription.*` в RabbitMQ

### Examples:
| provider        | event                  | status    |
| --------------- | ---------------------- | --------- |
| Telegram Wallet | tx в TON подтверждён   | succeeded |
| YooMoney        | payment.succeeded      | succeeded |
| Stripe Billing  | invoice.payment_failed | failed    |

---- 