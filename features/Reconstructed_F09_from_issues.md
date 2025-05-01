## Feature: [[F09] Загрузка PDF-CV кандидатом](https://github.com/OT-Click/project_root/issues/68)

_**_As a_** соискатель, **_I want to_** загрузить свой PDF-CV через Mini App/SPA, **_So that_** я мог предоставить системе резюме и получить публичную ссылку._

### Scenario: [[S09.01] Загрузка PDF-CV](https://github.com/OT-Click/project_root/issues/67)

- **_Given_** пользователь открыт раздел «Моё резюме»
- **_When_** он загружает файл PDF через форму
- **_Then_** вызывается `POST /upload`
- **_And_** событие `file.uploaded` публикуется в очередь
- **_And_** система возвращает public URL для скачивания

---- 