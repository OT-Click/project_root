## Feature: [[F06] Ручное обновление статуса в Chrome‑плагине](https://github.com/OT-Click/project_root/issues/62)

_**_As a_** рекрутер, **_I want to_** вручную обновлять статус вакансии и видеть агрегированную воронку, **_So that_** я мог учитывать офлайн‑коммуникацию с работодателем._

### Scenario: [[S06.01] Изменение статуса в плагине](https://github.com/OT-Click/project_root/issues/61)

- **_Given_** рекрутер открыл карточку вакансии в плагине
- **_When_** он выбирает статус «Interview»
- **_Then_** поле `Vacancy.status` обновляется до «Interview»
- **_And_** канбан Mini App показывает новую колонку «Interview» без перезагрузки страницы

---- 