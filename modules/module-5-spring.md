# Модуль 5: Spring Boot, HTTP API и слой данных

## Содержание

- [Для кого этот модуль](#для-кого-этот-модуль)
- [Зачем этот модуль](#зачем-этот-модуль)
- [FR и NFR в этом модуле](#fr-и-nfr-в-этом-модуле)
- [Технологический стек и инструменты](#технологический-стек-и-инструменты)
- [Что строим / как проект эволюционирует](#что-строим--как-проект-эволюционирует)
- [Архитектура модуля 5: HTTP API + слои приложения](#архитектура-модуля-5-http-api--слои-приложения)
- [Перед стартом](#перед-стартом)
- [Как проверить результат](#как-проверить-результат)
- [Публичные ресурсы](#публичные-ресурсы)

---

### Для кого этот модуль

**Ты прошёл модуль 4** — party service работает с PostgreSQL, миграции не пугают, JDBC-репозиторий за интерфейсом работает. Теперь проект становится backend-сервисом с HTTP API.

**Или ты не проходил модули 1-4**, но знаком с Java, Docker, базами данных и CI/CD. В таком случае стартуй отсюда — тебе понадобится самостоятельно реализовать FR/NFR предыдущих модулей и подвести к ним HTTP-слой.

---

## Зачем этот модуль

Модуль 4 дал проекту базу данных, SQL-мышление, миграции и привычку работать с persistence. В модуле 5 проект впервые становится **HTTP API на Spring Boot**: система общается с клиентом через HTTP-запросы и JSON, а не через консольный ввод.

До Spring Boot стоит коротко понять фундамент, на котором он стоит: что такое **IoC Container**, **bean** и **Dependency Injection** — как Spring управляет объектами и зачем это нужно. Это не отдельная тема для глубокого изучения, а мостик: без него магия `@Autowired` и starter'ов останется магией.

Консольное приложение превращается в backend-сервис. API проектируется от потребителя — организатор и участник получают разные endpoints. Идемпотентность — не технический detail, а бизнес-требование: повторный клик "записаться" не создаёт дубль регистрации. Spec-first: OpenAPI-контракт пишётся до или вместе с реализацией.

Этот модуль показывает, как backend-сервис устроен изнутри: где HTTP-слой, где бизнес-логика, где данные, и почему их смешивать опасно. С HTTP API возникает вопрос: **кто вызывает endpoint?** В проекте появляются роли **организатора**, **участника** и **администратора**. Организатор создаёт события, задаёт локацию и вместимость, решает нужен ли approve для записи, подтверждает заявки и может записывать участников на свои события. Участник записывается сам через web-слой, в том числе на события других организаторов. Администратор получает отдельный слой управления, если нужно модерировать общие сценарии и видеть систему целиком. Организатор тоже может быть участником чужих событий. Система обслуживает множество организаторов, участников и мероприятий одновременно. В этом же модуле появляется базовый **RBAC через Spring Security**: API уже должно отражать разделение ролей, даже если продвинутая security-конфигурация будет развиваться дальше.

**Результат модуля:** backend-сервис на Spring Boot с REST API, многоролевыми сценариями (организатор / участник), базовым RBAC через Spring Security, структурой `controller → service → repository`, JPA, транзакционными границами, MockMvc-тестами и OpenAPI-документацией.

Общие правила работы с roadmap, FR/NFR и ИИ — в [корневом README roadmap](../README.MD).

**Вопросы, на которые стоит ответить по ходу модуля:**

- Где здесь ресурс, а где RPC? Как спроектировать endpoint как REST-ресурс?
- Когда вернуть `200`, `201`, `400`, `404`? Почему HTTP-статусы важны для клиента?
- Кто вызывает endpoint? Как роль влияет на доступ к операциям?
- Где security rule действительно нужна, а где достаточно service-логики?
- Чем entity отличается от DTO? Почему нельзя отдавать entity как публичный API-контракт?
- Что такое транзакция в прикладном смысле и почему self-invocation в Spring опасен?
- Где могут появиться lazy loading и N+1? Как их заметить?

> **Ключевая граница модуля:** Spring basics (IoC, bean, DI), Spring Boot, HTTP API, роли (администратор / организатор / участник), RBAC через Spring Security, слои приложения, JPA, транзакции, MockMvc, OpenAPI.

---

## FR и NFR в этом модуле

> Общая рамка: [README → Как работать с требованиями](../README.MD#как-работать-с-требованиями).

### Функциональные требования

- Приложение запускается как Spring Boot сервис с HTTP API
- Для ключевых сущностей проекта доступны REST-эндпоинты с понятными URI и HTTP-методами
- Входные данные принимаются через DTO и валидируются
- Слои `controller`, `service`, `repository` разведены по ответственности
- Доступ к данным реализован через Spring Data JPA
- Операции изменения данных выполняются в понятных транзакционных границах
- Контракт API доступен через OpenAPI / Swagger UI — бизнес-мотивация: спецификация — это контракт между backend и потребителями (frontend, мобильное приложение, партнёрский сервис); без контракта API — чёрный ящик
- Базовый дизайн REST API опирается на Richardson maturity model и не скатывается в RPC-стиль — бизнес-мотивация: REST-ресурсы моделируют сущности предметной области (мероприятия, регистрации), а не внутренние операции; такой API понятен потребителю и устойчив к изменениям
- Организатор создаёт и управляет мероприятиями через REST API
- Организатор задаёт локацию, вместимость и режим записи: требуется подтверждение или нет
- Участник записывается на мероприятие через HTTP API; самостоятельная запись получает статус `pending`, если событие требует подтверждения
- Организатор подтверждает или отклоняет заявки участников на запись
- Организатор может записать участника на своё событие напрямую, минуя подтверждение
- Повторная отправка одного и того же прикладного запроса не создаёт дубли и не ломает состояние системы: сценарии записи на мероприятие, прямой записи организатором и подтверждения заявки проектируются идемпотентно — бизнес-мотивация: пользователь может случайно кликнуть дважды или потерять сеть и переотправить запрос; система защищена от дублей на уровне API-дизайна
- Сервис корректно обрабатывает одновременные HTTP-запросы разных пользователей: данные текущего запроса, роли и временный контекст выполнения не смешиваются между собой
- Организатор может быть участником событий других организаторов
- Администратор имеет отдельный слой управления и обзора системы
- API учитывает role-based access через Spring Security RBAC для сценариев администратора, организатора и участника — бизнес-мотивация: организатор не должен иметь доступ к административным операциям, а участник — к управлению чужими мероприятиями; API отражает бизнес-границы ролей
- JPA-модель поддерживает множество организаторов, участников, мероприятий и связей между ними

**Фильтрация и поиск мероприятий:**

- Список мероприятий поддерживает фильтрацию по статусу, дате, локации, организатору и наличию свободных мест
- Полнотекстовый поиск по названию и описанию мероприятия
- Динамические запросы строятся через Spring Data JPA Specification (`JpaSpecificationExecutor`)

**Пагинация и сортировка:**

- Списочные API возвращают постраничные результаты через Spring Data `Pageable` (`PageRequest`)
- Сортировка поддерживается по нескольким полям
- Ответ содержит метаданные пагинации: общее количество элементов, номер страницы, общее количество страниц

**★ Внешние интеграции:**

- **DaData**: автозаполнение адреса при создании мероприятия — REST-клиент к DaData Suggestions API (бесплатный лимит: 10 000 запросов/день)
- **iCal-генерация**: при создании мероприятия генерируется .ics-файл, который можно импортировать в Google Calendar, Яндекс.Календарь и любой другой календарь — без OAuth и внешних сервисов (библиотека iCal4j или аналогичная)
- **Email sandbox**: Mailpit добавляется в Docker Compose как sandbox для email-уведомлений — письма перехватываются локально, не уходят наружу

### Нефункциональные требования

- Сервис стабильно собирается и запускается локально через `./gradlew build` и `./gradlew bootRun`
- Приложение запускается локально через Docker Compose (Spring Boot + PostgreSQL как сервисы)
- API можно проверить без браузера: `curl`, Postman или HTTP Client в IDE
- Контроллеры не содержат бизнес-логику и SQL
- Тесты подтверждают happy path и ошибки в API через MockMvc
- Документация API доступна и читается без похода в код
- Изменение данных не расползается по случайным местам — транзакции остаются на сервисных операциях
- Ошибки отдаются в предсказуемом формате (RFC 7807 ProblemDetail — стандартный формат ошибок API)
- Одновременные HTTP-запросы не делят случайное изменяемое состояние: временный запросный контекст изолирован и очищается после завершения обработки
- Если между запросами хранится общее состояние в памяти (кэш, таблица дедупликации, буфер), выбран потокобезопасный или иммутабельный способ доступа
- Фильтрация и пагинация покрываются API-тестами: комбинации параметров фильтрации, корректность метаданных пагинации
- Есть воспроизводимая проверка двух и более параллельных запросов: роли, correlation context и результаты не смешиваются
- API-эндпоинты разделены по ролям: администраторские, организаторские и участковые операции не смешиваются в один бесформенный слой
- RBAC через Spring Security отражён в API-слое и не размазан хаотично по коду
- Приложение контейнеризировано: CI pipeline в GitHub Actions собирает Docker-образ и пушит в Container Registry (например, GitHub Container Registry)
- CI pipeline содержит отдельные стадии (checkstyle, build, test, push to registry) и проходит на каждом push/PR
- Покрытие тестами JaCoCo >= 80%
- В проекте есть файл правил для AI-инструмента (AGENTS.md или CLAUDE.md): стек, конвенции, архитектурные решения, антипаттерны — AI учитывает контекст проекта при работе с кодом и не предлагает решений, нарушающих принятую структуру слоёв
- MCP-плагины подключены: spring-docs (документация Spring), Context7 (документация библиотек), Swagger MCP (OpenAPI-контракты), Docker MCP (управление compose) — AI-агент использует MCP-плагины для доступа к документации Spring и библиотек; ответы по фреймворку сверяются через официальные источники, а не угадываются
- AI-агент с подключённым Swagger MCP способен читать OpenAPI-спецификацию сервиса и помогать проверять эндпоинты: создавать тестовые запросы, проверять статусы, находить расхождения между контрактом и реализацией
- OpenAPI-контракт доступен и актуален; AI сверяется с ним при работе с эндпоинтами
- AI-агент управляет docker compose через Docker MCP: запускает и останавливает сервисы, проверяет health, читает логи — без ручного переключения между терминалом и агентом
- Сгенерированный AI код (контроллеры, сервисы, конфигурация) проверяется через сборку, тесты и запуск — AI не снимает ответственности за корректность REST-дизайна и обработки ошибок
- Идемпотентность: повторный HTTP-запрос не создаёт дубль и не переводит систему в неконсистентное состояние

---

## Технологический стек и инструменты

| Категория | Что используем | Минимум для Done |
|---|---|---|
| Framework | Spring Boot 3.2+ | сервис поднимается через `bootRun` |
| Web | Spring Web MVC | рабочие REST-эндпоинты |
| DI | Spring IoC Container | зависимости через конструкторы |
| Persistence | Spring Data JPA | `JpaRepository` работает |
| База данных | PostgreSQL | локальный профиль подключается к PostgreSQL |
| Транзакции | Spring Transaction Management | write-операции через `@Transactional` |
| Validation | Jakarta Bean Validation | невалидный JSON даёт 4xx |
| API tests | MockMvc | покрыты happy path и error cases |
| Документация | springdoc OpenAPI / Swagger UI | `/swagger-ui.html` доступен |
| Наблюдаемость | Spring Boot Actuator | `/actuator/health` отвечает |
| AI | Qwen Code CLI + MCP-экосистема (spring-docs, Context7, Swagger MCP, Docker MCP), CodeRabbit (опционально) | AI управляет compose-стеком, сверяется с документацией, проверяет API-контракты |

---

## Что строим / как проект эволюционирует

**Стартовая точка:** приложение из модуля 4 умеет работать с PostgreSQL и persistence, но ещё не является backend-сервисом с HTTP-контрактом.

### До → после

| Слой / аспект | До | После |
|---|---|---|
| Вход в систему | консольный сценарий | HTTP API |
| Дизайн API | команды и локальные сценарии | ресурсный REST-подход по Richardson maturity model |
| Роли | организатор за консолью; участники — данные | администратор + организатор + участник через раздельные endpoint-группы и RBAC |
| Runtime | Java приложение без web-слоя | Spring Boot backend-сервис |
| Структура | код вокруг persistence | `controller → service → repository` |
| Доступ к данным | JDBC и ручной SQL | JPA и Spring Data repositories |
| Проверка | локальные тесты и ручные сценарии | MockMvc + HTTP-проверки |
| Документация | README и заметки | OpenAPI / Swagger UI |
| Эксплуатация | "запустилось локально" | health-check и наблюдаемость |
| AI-инструменты | Qwen Code CLI | Qwen Code CLI + MCP-экосистема (spring-docs, Context7, Swagger MCP, Docker MCP), CodeRabbit (опционально), проектные правила (AGENTS.md/CLAUDE.md), валидация через сборку и тесты |

### К концу модуля сервис умеет

- принимать HTTP-запросы на CRUD-сценарии и возвращать JSON с корректными HTTP-кодами;
- проектировать endpoint'ы как REST-ресурсы, а не RPC-команды: с понятными URI, HTTP-методами и статусами по Richardson maturity model;
- разделять endpoint-группы для организатора (создание мероприятий, управление записью, подтверждение заявок) и участника (просмотр, самостоятельная запись) через базовый RBAC;
- валидировать входные DTO и читать/сохранять данные через JPA;
- хранить у события локацию, вместимость и признак "требует ли событие подтверждения записи";
- хранить статус записи (pending / approved / rejected) и поддерживать подтверждение заявок;
- поддерживать сценарий, где организатор может быть участником событий других организаторов;
- обновлять данные в транзакционных границах;
- обслуживать множество организаторов, участников и мероприятий одновременно;
- не хранить изменяемое состояние запроса в singleton-компонентах, изолировать временный request-context и не опираться на случайный shared state между запросами;
- проходить базовые API-тесты и показывать контракт через OpenAPI;
- запускаться локально через Docker Compose (приложение + PostgreSQL);
- использовать AI-инструмент с MCP-экосистемой (spring-docs, Context7, Swagger MCP, Docker MCP) для документации Spring, OpenAPI-контракта и управления docker compose, с проектными правилами (AGENTS.md/CLAUDE.md) и валидацией через сборку и тесты.

---

## Архитектура модуля 5: HTTP API + слои приложения

Модуль 5 закрепляет первое "взрослое" разделение backend-кода. Каждый слой отвечает за свой кусок работы.

### Слои и ответственность

| Слой | Ответственность | Что туда НЕ кладём |
|---|---|---|
| `controller` | принимает HTTP-запрос, вызывает use case, собирает HTTP-ответ | SQL, транзакции, бизнес-логику |
| `service` | бизнес-правила, координация сценария, транзакционные границы | JSON, HTTP-детали |
| `repository` | persistence через JPA и Spring Data | бизнес-правила |
| `entity` / `domain` | состояние и связи данных | логику контроллеров, сериализацию |
| `dto` | контракт запроса и ответа API | persistence-аннотации |

### Что особенно важно проговорить

- Контроллер не должен знать, как JPA сохраняет сущность.
- Сервис не должен принимать на себя HTTP-статусы и JSON.
- DTO нужны, чтобы не выдавать entity как публичный API-контракт.
- `@Transactional` живёт на сервисной операции, а не на контроллере.

### Параллелизм в scope модуля

Модуль 5 вводит web-runtime параллелизм: одновременные HTTP-запросы не смешивают request-context, singleton-компоненты не хранят mutable request state, а общий in-memory state между запросами допускается только при потокобезопасной или иммутабельной модели доступа.

### Роли и их влияние на слои

С HTTP API «кто вызывает endpoint» становится осмысленным вопросом. В этом модуле появляются три роли, и каждая по-своему проходит через слои:

| Роль | Что делает через API | Где отражается |
|---|---|---|
| **Администратор** | видит систему целиком, управляет общими сценариями и модерирует при необходимости | отдельный admin-layer в API и сервисах |
| **Организатор** | создаёт мероприятия, задаёт локацию/вместимость/approve-режим, подтверждает/отклоняет заявки, записывает участников напрямую | отдельные endpoint-группы, сервисные операции с approve-логикой |
| **Участник** | просматривает мероприятия, записывается сам, может быть организатором в других сценариях | отдельные endpoint-группы, запись получает статус `pending` при approve-flow |

**Влияние на слои:**
- **Web-слой (controller):** раздельные endpoint-группы для администратора, организатора и участника; URI и HTTP-методы оформлены как REST-ресурсы, а не RPC-команды; доступ к группам endpoint'ов ограничивается базовыми ролями через Spring Security.
- **Service-слой:** содержит бизнес-правила подтверждения (pending → approved / rejected), различает прямую запись организатором от самостоятельной записи участника, проверяет вместимость и режим approve для события.
- **JPA-слой (repository + entity):** хранит множество организаторов, участников, мероприятий и связей (registration entity со статусом); event aggregate включает локацию, вместимость и флаг подтверждения.

Продвинутая security-конфигурация (JWT, OAuth2, production-hardening) — за пределами этого модуля. Но базовый RBAC через Spring Security уже входит в scope, чтобы роли были не только на словах, но и в API-контракте.

---

## Перед стартом

- модуль 4 завершён: PostgreSQL, миграции и локальный запуск БД не пугают;
- `./gradlew build` зелёный;
- есть минимальное понимание SQL-транзакции и rollback;
- IDE готова запускать Gradle-задачи и тесты.

---

## Как проверить результат

> **Дисклеймер:** примеры endpoint'ов ниже — это один из возможных вариантов для самопроверки. Если твои названия и пути отличаются — это допустимо. Главное — чтобы FR/NFR выполнялись, а API читался как REST-ресурсы, а не RPC-команды.

| Что проверяем | Как проверить | Что ожидаем |
|---|---|---|
| Сборка | `./gradlew build` | сборка и тесты проходят |
| Локальный запуск | `./gradlew bootRun` | сервис поднимается без ошибок |
| Docker Compose | `docker compose up` | приложение + PostgreSQL поднимаются вместе, API доступно |
| Чтение списка | HTTP-запрос к списку мероприятий (например, `GET /api/events`) | JSON-ответ, `200 OK` |
| Создание (организатор) | HTTP-запрос на создание мероприятия | `201 Created`, у события есть локация, вместимость и approve-флаг |
| Admin-layer | отдельные административные endpoint'ы | есть отдельные административные сценарии и обзор системы |
| Запись участника | HTTP-запрос на запись участника на мероприятие | запись создана, статус `pending` если событие требует подтверждения |
| Подтверждение (организатор) | HTTP-запрос на подтверждение заявки | статус → `approved` |
| Прямая запись (организатор) | HTTP-запрос на прямую запись организатором | запись создана, статус `approved` |
| RBAC | admin/organizer/participant endpoints | доступ различается по ролям Spring Security |
| Мульти-роль | организатор как участник чужого события | сценарий поддержан моделью и API |
| Параллельные запросы | Одновременно отправить 2+ запроса от разных ролей на запись или подтверждение | статусы, роли и данные не смешиваются; результат каждого запроса независим |
| Изоляция request-context | Проверить логи или аудит при двух одновременных запросах с разными correlation/user данными | временный контекст одного запроса не "протекает" в другой и очищается после завершения |
| Shared state (если есть) | Нагрузить кэш, буфер или таблицу дедупликации параллельными запросами | гонок и смешивания данных между запросами нет |
| Ошибка валидации | невалидный JSON | `400 Bad Request` |
| Health-check | `/actuator/health` | статус `UP` |
| Документация | Swagger UI | список эндпоинтов с роль-разделением |
| REST design review | URI + HTTP methods + статусы | нет RPC-дрейфа, endpoints читаются как ресурсы |
| API-тесты | `./gradlew test` | MockMvc тесты зелёные, роль-сценарии покрыты |
| Покрытие тестами | `./gradlew jacocoTestReport` | JaCoCo >= 80% |
| Контейнеризация | `docker build` в CI pipeline, образ доступен в Container Registry | |
| CI pipeline | push/PR в GitHub | pipeline со стадиями checkstyle → build → test → push to registry проходит |

---

## Публичные ресурсы

### YouTube — видео по темам модуля

| Тема | Источник |
|---|---|
| **Евгений Борисов** | |
| Spring-потрошитель, ч. 1 (Борисов, 1:04) | [YouTube](https://www.youtube.com/watch?v=BmBr5diz8WA) |
| Spring-потрошитель — Ремейк: вооруженный Дебаггером (Борисов, 1:43, 2026) | [YouTube](https://www.youtube.com/watch?v=Kb82DYpaHGY) |
| Spring-построитель (Борисов, 2:24, лайв-кодинг) | [YouTube](https://www.youtube.com/watch?v=rd6wxPzXQvo) |
| Spring Patterns для взрослых (Борисов) | [YouTube](https://www.youtube.com/watch?v=GL1txFxswHA) |
| **alishev** | |
| Spring Framework — плейлист (alishev, 28 уроков) | [YouTube playlist](https://www.youtube.com/playlist?list=PLAma_mKffTOR5o0WNHnY0mTjKxnCgSXrZ) |
| CRUD, REST, DAO — урок 21 (alishev, 36 мин) | [YouTube](https://www.youtube.com/watch?v=D58pIymCew4) |
| **Практика REST и транзакций** | |
| Spring REST: первый REST-сервис на Spring Boot (Артём Михайлов, 1:25) | [YouTube](https://www.youtube.com/watch?v=749xa3kaZPM) |
| Spring Framework — транзакции (Изучаем Java, 25 мин) | [YouTube](https://www.youtube.com/watch?v=eKoH_39QZiQ) |
| **EN** | |
| Building RESTful Web Services (Spring Academy, EN) | [YouTube](https://www.youtube.com/watch?v=7cvQzHC2kDE) |
| Spring Boot 2025 (Amigoscode, EN) | [YouTube](https://www.youtube.com/watch?v=Cw0J6jYJtzw) |
| Spring Data JPA (Amigoscode, EN) | [YouTube](https://www.youtube.com/watch?v=8SGI_XS5OPw) |
| @Transactional: Propagation, Proxy (EN) | [YouTube](https://www.youtube.com/watch?v=ZWuvSOCRs3Q) |
| Spring Boot 3 Swagger UI (EN) | [YouTube](https://www.youtube.com/watch?v=xZyUOhRpuQ0) |

### Spring IoC, bean, DI

| Тема | Источник |
|---|---|
| IoC Container | [Spring IoC Container](https://docs.spring.io/spring-framework/reference/core/beans.html) · [DI](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html) |
| Spring IoC и DI (Habr) | [Habr](https://habr.com/ru/articles/473502/) |

### Spring Boot: старт

| Тема | Источник |
|---|---|
| Официальная документация | [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/) |
| Генератор проекта | [Spring Initializr](https://start.spring.io/) |
| REST API на Java (Habr) | [Habr](https://habr.com/ru/articles/435144/) |

### Web MVC и REST API

| Тема | Источник |
|---|---|
| Spring Web MVC | [Spring Web MVC](https://docs.spring.io/spring-framework/reference/web/webmvc.html) |
| gs/rest-service | [Spring Guide](https://spring.io/guides/gs/rest-service/) |
| Richardson Maturity Model | [Martin Fowler](https://martinfowler.com/articles/richardsonMaturityModel.html) · [Модель зрелости Ричардсона](http://art-in-stamps.ru/development/richardson-maturity-model.shtml) |
| Идемпотентность HTTP/API | [YouTube — источник от пользователя](https://youtu.be/hTciTPbReKw?si=RDAtfBqQq84UVw8C) |
| Spring Boot REST API (SYSOUT) | [SYSOUT](https://sysout.ru/spring-boot-rest-api/) |

### DTO, Entity, валидация

| Тема | Источник |
|---|---|
| Entity vs DTO | [Baeldung](https://www.baeldung.com/java-entity-vs-dto) · [Прослойка из DTO (JavaRush)](https://javarush.com/groups/posts/3139-spring---ehto-ne-strashno-prosloyka-iz-dto) |
| Bean Validation | [Spring Validation](https://docs.spring.io/spring-framework/reference/core/validation/beanvalidation.html) · [Java & Spring Validation (Habr)](https://habr.com/ru/articles/424819/) |

### Spring Data JPA

| Тема | Источник |
|---|---|
| Spring Data JPA Reference | [Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/) |
| Практические разборы | [Spring Data JPA (Habr)](https://habr.com/ru/articles/435114/) · [Доводим напильником (Habr)](https://habr.com/ru/articles/444240/) |

### Specification и динамические запросы

| Тема | Источник |
|---|---|
| Spring Data JPA Specification | [Spring Data — Specification](https://docs.spring.io/spring-data/jpa/reference/repositories/core-extensions.html#specifications) |
| Динамические запросы с Specification | [Хабр — Построение динамических запросов](https://habr.com/ru/articles/870698/) · [Foxminded — Динамическая фильтрация](https://foxminded.ua/ru/dinamicheskaya-filtratsiya-v-spring-data-jpa-1/) |
| JPA Criteria API (фундамент Specification) | [Habr — OTUS JPA Criteria](https://habr.com/ru/companies/otus/articles/707724/) |

### Пагинация и сортировка

| Тема | Источник |
|---|---|
| Spring Data Pagination | [Spring Data — Paging and Sorting](https://docs.spring.io/spring-data/jpa/reference/repositories/core-extensions.html#core-extensions.pageable) |
| Пагинация в Spring Data JPA | [SYSOUT — Pagination и Sorting](https://sysout.ru/pagination-i-sorting-v-spring-data-jpa/) · [Baeldung — Pagination and Sorting](https://www.baeldung.com/spring-data-jpa-pagination-sorting) |

### N+1, Lazy Loading, транзакции

| Тема | Источник |
|---|---|
| N+1 в JPA и Hibernate | [Habr](https://habr.com/ru/articles/896618/) · [N+1 запросы (Habr)](https://habr.com/ru/companies/otus/articles/529692/) |
| Spring Transaction Management | [Spring Docs](https://docs.spring.io/spring-framework/reference/data-access/transaction.html) · [@Transactional в деталях (Habr)](https://habr.com/ru/articles/682362/) |

### Spring Security / RBAC

| Тема | Источник |
|---|---|
| Spring Security Reference | [Spring Security](https://docs.spring.io/spring-security/reference/index.html) |
| Spring Security basics | [Habr](https://habr.com/ru/articles/349784/) |

### Тестирование: MockMvc

| Тема | Источник |
|---|---|
| MockMvc | [Spring Docs](https://docs.spring.io/spring-framework/reference/testing/mockmvc.html) · [Testing the Web Layer](https://spring.io/guides/gs/testing-web/) |
| MockMVC практические разборы | [Habr](https://habr.com/ru/companies/otus/articles/746414/) · [SYSOUT](https://sysout.ru/testirovanie-kontrollerov-s-pomoshhyu-mockmvc/) |

### OpenAPI / Swagger

| Тема | Источник |
|---|---|
| springdoc-openapi | [springdoc.org](https://springdoc.org/) · [OpenAPI Spec](https://swagger.io/specification/) |
| OpenAPI в Spring Boot | [Habr](https://habr.com/ru/companies/otus/articles/965178/) · [Генерация OpenAPI (Habr)](https://habr.com/ru/articles/814061/) |

### Actuator

| Тема | Источник |
|---|---|
| Spring Boot Actuator | [Spring Docs](https://docs.spring.io/spring-boot/reference/actuator/) |
| Практические гайды | [Полный гайд (Habr)](https://habr.com/ru/companies/otus/articles/1008360/) · [Введение (Habr)](https://habr.com/ru/companies/otus/articles/452624/) |

### Внешние интеграции (★)

| Тема | Источник |
|---|---|
| DaData Suggestions API | [DaData — Подсказки по адресам](https://dadata.ru/api/suggest/address/) · [Spring-клиент (GitHub)](https://github.com/KuliginStepan/dadata-client) |
| iCal4j — генерация .ics | [iCal4j Documentation](https://www.ical4j.org/) · [iCal4j GitHub](https://github.com/ical4j/ical4j) |
| Mailpit — email sandbox | [Mailpit GitHub](https://github.com/axllent/mailpit) · [Docker Hub](https://hub.docker.com/r/axllent/mailpit) |

### CI: GitHub Actions + Docker

| Тема | Источник |
|---|---|
| Java with Gradle | [GitHub Docs](https://docs.github.com/en/actions/use-cases-and-examples/building-and-testing/building-and-testing-java-with-gradle) |
| Publish Docker image | [GitHub Docs](https://docs.github.com/en/actions/use-cases-and-examples/publishing-packages/publishing-docker-images) |

### AI-инструменты

| Тема | Источник |
|---|---|
| Context7 — документация библиотек: AI сверяется с официальными источниками | [GitHub](https://github.com/upstash/context7) |
| spring-docs MCP — документация Spring Boot: AI отвечает на основе reference docs | [GitHub](https://github.com/spring-projects/spring-boot) |
| Swagger MCP — генерация MCP-инструментов из OpenAPI spec; AI проверяет контракты | [GitHub](https://github.com/LostInBrittany/swagger-to-mcp-generator) |
| Docker MCP — управление docker compose через AI: старт/стоп/health/логи | [GitHub](https://github.com/ckreiling/mcp-server-docker) |
| CodeRabbit (опционально) — AI code review для pull requests | [coderabbit.ai](https://www.coderabbit.ai/) |
| Принцип: AI без access к документации — источник галлюцинаций; MCP даёт доступ к docs | — |
