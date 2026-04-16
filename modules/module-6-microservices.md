# Модуль 6: Микросервисная архитектура

## Содержание

- [Для кого этот модуль](#для-кого-этот-модуль)
- [Зачем этот модуль](#зачем-этот-модуль)
- [FR и NFR в этом модуле](#fr-и-nfr-в-этом-модуле)
- [Технологический стек и инструменты](#технологический-стек-и-инструменты)
- [Что строим / как проект эволюционирует](#что-строим--как-проект-эволюционирует)
- [Архитектура модуля 6: DDD-декомпозиция монолита на микросервисы](#архитектура-модуля-6-ddd-декомпозиция-монолита-на-микросервисы)
- [Перед стартом](#перед-стартом)
- [Как проверить результат](#как-проверить-результат)
- [Публичные ресурсы](#публичные-ресурсы)

---

## Для кого этот модуль

**Модуль 5 завершён** — party service работает как Spring Boot приложение с REST API, PostgreSQL, ролевой моделью и MockMvc-тестами. Теперь проект превращается из монолита в набор взаимодействующих микросервисов.

**Или вы не проходили модули 1–5**, но знакомы с Java, Spring Boot, Docker, базами данных и CI/CD. В таком случае стартуйте отсюда — вам понадобится самостоятельно воспроизвести FR/NFR предыдущих модулей.

---

## Зачем этот модуль

Модуль 5 дал проекту HTTP API, Spring Boot runtime, JPA-слой данных, транзакции и ролевую модель. Но всё это живёт в одном приложении: одна база данных, один процесс, один набор конфигураций.

В реальных backend-системах единый монолит превращается в набор сервисов, каждый из которых владеет своей частью данных и бизнес-логики. Главное — понять, **как проводить границы между сервисами**: не по техническим слоям (отдельный сервис для DAO, отдельный для API), а по доменным контекстам — bounded contexts из Domain-Driven Design. Event-service владеет мероприятиями, registration-service владеет записями — у каждого своя модель, своя база, своя ответственность.

Монолит разделяется на 4 бизнес-сервиса по DDD bounded contexts. Каждый bounded context — это бизнес-граница, а не техническое разбиение. Регистрация и мероприятия масштабируются по-разному — database per service даёт им независимый рост. PDI: Problem (сервисы связаны, падение одного тянет другие) → Decision (Resilience4j Circuit Breaker) → Impact (пользователь записывается, даже если уведомления недоступны).

Вместе с декомпозицией появляются новые вызовы: как сервисы общаются между собой, как обеспечить целостность данных при распределённых операциях, кто проверяет токены, как тестировать межсервисные контракты и что делать, когда один сервис временно недоступен.

Модуль 6 показывает, как монолит превращается в несколько сервисов с отдельными базами данных, API-шлюзом и авторизацией. Всё это запускается локально через Docker Compose — без Kubernetes и без Kafka. Service Discovery и orchestration остаются за пределами модуля: в enterprise они решаются через Kubernetes, что покрывается в модуле 7.

**Результат модуля:** набор из 4 бизнес-сервисов + API-шлюз, поднимающихся одной командой через Docker Compose, с декомпозицией по DDD bounded contexts, Database per Service, Feign-клиентами для межсервисных вызовов, Retry + Circuit Breaker, Saga/Outbox для распределённых операций, WireMock-тестами контрактов и OpenAPI-документацией на каждый сервис.

Общие правила roadmap, FR/NFR и ИИ — в [корневом README roadmap](../README.MD).

> **Ключевая граница модуля:** DDD-декомпозиция монолита на bounded contexts, API Gateway, Database per Service, Feign-клиент для межсервисных вызовов, Retry + Circuit Breaker, Saga Pattern (Outbox без Kafka), WireMock контрактное тестирование, HashiCorp Vault для секретов, Docker Compose multi-service.

---

## FR и NFR в этом модуле

> Общая рамка: [README → Как работать с требованиями](../README.MD#как-работать-с-требованиями).

### Функциональные требования

**Наследование из модулей 1–5:** все FR сохраняются — создание мероприятия, регистрация, отмена, waitlist, undo, RBAC, OpenAPI.

**Декомпозиция (DDD):**

- Монолит разделён на 4 бизнес-сервиса + API-шлюз по границам bounded contexts: каждый сервис владеет собственным поддоменом и данными — бизнес-мотивация: мероприятия и регистрации меняются с разной скоростью и масштабируются по-разному; разделение по доменным границам позволяет развивать их независимо
- Границы сервисов определены по доменной ответственности (event, registration, user, notification), а не по техническим слоям — бизнес-мотивация: техническое разбиение (сервис для DAO, сервис для API) не даёт независимости; бизнес-границы позволяют менять реализацию одного контекста, не затрагивая другие
- Каждый сервис владеет собственной базой данных (отдельная PostgreSQL-схема); сервисы не обращаются к чужим таблицам напрямую — бизнес-мотивация: database per service даёт каждому bounded context независимый жизненный цикл данных и схем

**API Gateway:**

- API-шлюз — единая точка входа; маршрутизирует запросы к бизнес-сервисам
- Шлюз скрывает внутреннюю топологию от клиентов

**Межсервисные вызовы:**

- Сервисы общаются через REST API; синхронные вызовы через Feign-клиент (декларативный стиль)
- В сценариях, где ответ пользователю собирается из нескольких независимых downstream-сервисов, вызовы выполняются параллельно, а не последовательной цепочкой
- RestClient упомянут как перспективный инструмент Spring Framework 7 — для ознакомления, не как основной

**Устойчивость (Resilience):**

- Retry: автоматический повтор при временной недоступности downstream-сервиса — бизнес-мотивация: кратковременный сбой сети не должен ломать сценарий записи участника
- Circuit Breaker: при повторных ошибках вызовы к сервису приостанавливаются — система не тратит ресурсы на заведомо нерабочие вызовы; бизнес-мотивация: пользователь записывается, даже если уведомления недоступны
- Ошибка одного сервиса не разрушает health всей системы: gateway возвращает понятный ответ, даже если downstream-сервис недоступен — бизнес-мотивация: частичное падение инфраструктуры не должно полностью блокировать сервис для пользователей

**Распределённые данные:**

- End-to-end сценарий проходит цепочку: пользователь → gateway → event-service → registration-service → notification-service
- Saga pattern рассмотрен как подход к консистентности между сервисами: регистрация участника затрагивает event-service (места) и registration-service (запись) — это распределённая операция — бизнес-мотивация: если уменьшение мест в event-service прошло, а запись в registration-service упала — система в неконсистентном состоянии; Saga обеспечивает целостность бизнес-операции
- Transactional Outbox: таблица outbox в БД сервиса + polling-обработчик обеспечивают надёжную доставку межсервисных сообщений без message broker — бизнес-мотивация: бизнес-событие (участник записался) не должно потеряться из-за временной недоступности downstream-сервиса

**Секреты:**

- Пароли БД, API-ключи и другие секреты хранятся в HashiCorp Vault (dev mode в Docker Compose), а не в application.yml
- Сервисы читают секреты из Vault через Spring Cloud Vault

**Контракты и тестирование:**

- OpenAPI-контракт на каждом сервисе; контракт доступен через Swagger UI
- WireMock-stub'ы обеспечивают изолированное тестирование межсервисных HTTP-контрактов
- Actuator health endpoint доступен на каждом сервисе
- Docker Compose поднимает весь стек одной командой

**★ Авторизация (Keycloak):**

- Keycloak запускается в Docker Compose как централизованный auth-сервер
- Сервисы не авторизуют друг друга напрямую — API-шлюз проверяет JWT-токен
- Это задание со звёздочкой: на реальной работе авторизация — инфраструктурная тема, которой обычно занимается отдельная команда. Java-разработчик интегрируется с готовым auth-решением (Keycloak, Okta), а не создаёт свой auth-сервер. В Kubernetes-окружении аналогичную задачу решают через Open Policy Agent + Ingress-контроллер

**★ Внешние интеграции (продолжение из модуля 5):**

- **DaData**: интеграция переносится в event-service — адрес обогащается через DaData API при создании мероприятия
- **iCal-генерация**: переносится в notification-service — .ics-файл прикрепляется к уведомлению о создании мероприятия
- **Telegram Bot**: notification-service отправляет push-уведомления через Telegram Bot API (полностью бесплатный, без лимитов)
- **Email**: notification-service отправляет email через Resend (3 000 писем/мес бесплатно) или через SMTP-провайдера

### Нефункциональные требования

- Compose-стек стартует воспроизводимо на чистой машине
- Actuator health endpoint доступен на каждом сервисе; compose healthchecks зелёные
- Каждый сервис живёт в собственном репозитории (или чётко выделенном модуле в монорепо)
- Межсервисные вызовы через Feign-клиент; без Kafka и без gRPC
- Retry и Circuit Breaker настроены на ключевых межсервисных вызовах — через Resilience4j или Spring Cloud Circuit Breaker
- Контрактные тесты с WireMock покрывают ключевые межсервисные сценарии в изоляции
- Outbox-таблица обрабатывается polling-механизмом (`@Scheduled` + `SELECT FOR UPDATE SKIP LOCKED`) — без Debezium и без Kafka; `SKIP LOCKED` гарантирует, что при горизонтальном масштабировании (несколько экземпляров сервиса) два экземпляра не обработают одну и ту же outbox-запись — каждый экземпляр забирает незаблокированную строку, а не ждёт освобождения чужой
- Параллельные fan-out сценарии ограничены и диагностируемы: у каждой ветки есть понятный timeout/cancel budget, медленный downstream не удерживает остальные ветки бесконечно
- Контекст безопасности, корреляции и аудита корректно доходит до параллельных веток и не смешивается между запросами
- Если для агрегации ответов используется общий in-memory state, выбран потокобезопасный или иммутабельный способ доступа
- Секреты не хранятся в коде и application.yml — только в безопасном хранилище
- Локальная диагностика доступна: при latency-спайке или задержке межсервисного вызова разработчик может получить состояние потоков приложения (thread dump) и проанализировать, где потоки ожидают.
- Профилирование доступно локально: разработчик может запустить приложение с профилировщиком и получить данные о горячих методах, потреблении памяти и времени выполнения.

**AI-инструменты и агентная работа:**

- Per-service AGENTS.md: каждый сервис содержит файл правил (bounded context, API-контракт, зависимости) — AI-агент соблюдает границы сервиса при работе с кодом
- Docker MCP: AI управляет multi-service compose-стеком (старт/стоп отдельных сервисов, health, логи) — без ручного переключения между терминалами
- Swagger MCP: AI проверяет совместимость endpoint-ов между сервисами через OpenAPI-контракты
- Архитектурные решения документированы с бизнес-обоснованием (почему сервисы разделены так, а не иначе)
- Сгенерированный AI код (Feign-клиенты, конфигурация Resilience4j, Outbox-обработчик) проверяется через сборку, контрактные тесты и запуск стека — AI не снимает ответственности за корректность межсервисного взаимодействия

### SLI/SLO: метрики надёжности микросервисов

Микросервисная архитектура делает систему распределённой — каждый сервис отвечает за свою часть функциональности. Чтобы понимать, работает ли система в целом, нужны количественные метрики надёжности:

- **SLI** (Service Level Indicator) — измеримый показатель: доля успешных HTTP-ответов, медианная задержка, время доступности сервиса
- **SLO** (Service Level Objective) — целевое значение SLI: «99% запросов к event-service возвращают ответ за 200ms», «доступность registration-service не ниже 99.5%»
- **SLA** (Service Level Agreement) — формальный контракт с последствиями за нарушение SLO (в internal-сервисах часто не формализуется, но концепция важна для понимания)

В этом модуле SLI/SLO закладываются как **концептуальная база**: каждый сервис через Actuator и Micrometer экспортирует метрики (HTTP-запросы, latency, ошибки, Circuit Breaker state). Полноценная реализация — SLI/SLO-дашборды, alerting и error budget — в модуле 7, когда появится Prometheus + Grafana.

---

## Технологический стек и инструменты

| Категория | Что используем | Минимум для Done |
|---|---|---|
| Framework | Spring Boot 3.4.x | каждый сервис стартует как Boot-приложение |
| Microservices | Spring Cloud 2024.0.x (Moorgate) | Gateway, Feign, Resilience4j работают |
| Декомпозиция | DDD bounded contexts | сервисы выделены по доменным границам, а не по техническим слоям |
| API Gateway | Spring Cloud Gateway | маршрутизация запросов к сервисам |
| База данных | PostgreSQL (отдельная на сервис) | каждый сервис владеет своими данными |
| Межсервисные вызовы | Spring Cloud OpenFeign | декларативные REST-клиенты между сервисами |
| Перспектива | RestClient + HTTP Interface | ознакомление: будущий стандарт Spring для HTTP-клиентов |
| Resilience | Resilience4j (Retry + Circuit Breaker) | межсервисные вызовы защищены от каскадных сбоев |
| Контрактное тестирование | WireMock 3.x | stub'ы для изолированного тестирования контрактов |
| Секреты | HashiCorp Vault (dev mode) | DB credentials и пароли хранятся в Vault |
| Auth (★) | Keycloak (Docker Compose) | JWT-токены выдаются и валидируются |
| Документация API | springdoc OpenAPI / Swagger UI | контракт доступен на каждом сервисе |
| Health Checks | Spring Actuator | `/actuator/health` на каждом сервисе |
| Локальная инфраструктура | Docker Compose (profiles) | весь стек поднимается одной командой |
| AI | Qwen Code CLI + Docker MCP + Swagger MCP (per-service), per-service AGENTS.md | Multi-service AI-оператор; AI соблюдает границы сервисов и проверяет контракты |

---

## Что строим / как проект эволюционирует

**Стартовая точка:** Spring Boot монолит из модуля 5 — REST API, PostgreSQL, ролевая модель, MockMvc-тесты, Actuator.

### До → после

| Слой / аспект | До (модуль 5) | После (модуль 6) |
|---|---|---|
| Архитектура | один монолит | 4 бизнес-сервиса + gateway по DDD bounded contexts |
| Декомпозиция | по техническим слоям (controller/service/repository) | по доменным контекстам (event, registration, user, notification) |
| Данные | одна PostgreSQL | отдельная БД на сервис (database per service) |
| Вход в систему | прямой доступ к REST API | через API-шлюз (единая точка входа) |
| Межсервисные вызовы | нет | Feign-клиент для синхронных REST-вызовов |
| Resilience | нет | Retry + Circuit Breaker на межсервисных вызовах |
| Авторизация | Spring Security RBAC в монолите | Keycloak (★) + JWT через gateway |
| Секреты | в application.yml | HashiCorp Vault |
| Тестирование | MockMvc для API | MockMvc + WireMock для межсервисных контрактов |
| Инфраструктура | `docker compose` для app + db | `docker compose` для 6+ сервисов + их БД + Vault |
| Паттерны | нет | API Gateway, Database per Service, Retry, Circuit Breaker, Saga/Outbox, Contract Testing |

### К концу модуля система умеет

- обслуживать end-to-end сценарий через цепочку сервисов: gateway → event → registration → notification
- маршрутизировать запросы через API-шлюз к нужному сервису
- объяснить, ПОЧЕМУ сервисы выделены именно так (bounded contexts, а не технические слои)
- общаться между сервисами через Feign-клиент с защитой Retry + Circuit Breaker
- параллельно опрашивать независимые сервисы в одном пользовательском сценарии, когда это сокращает latency и не ломает консистентность
- обеспечивать надёжность межсервисных операций через Outbox pattern (без Kafka)
- хранить секреты в Vault, а не в конфигурационных файлах
- тестировать межсервисные контракты в изоляции через WireMock
- подниматься целиком через `docker compose --profile full up`

---

## Архитектура модуля 6: DDD-декомпозиция монолита на микросервисы

Главная задача модуля — разделить монолит на сервисы по границам доменных контекстов (bounded contexts), а не по техническим слоям. Каждый сервис владеет своей частью домена и данных, имеет собственный API-контракт и может развиваться независимо.

### Декомпозиция сервисов

| Сервис | Bounded Context | Ответственность | Данные |
|---|---|---|---|
| **event-service** | Event Management | CRUD мероприятий, локации, вместимость, статусы, проверка свободных мест | Своя БД: events, locations |
| **registration-service** | Registration | Запись на мероприятия, waitlist, подтверждение/отклонение заявок, статусы регистрации | Своя БД: registrations, waitlist_entries, outbox |
| **user-service** | User Management | Профили пользователей, роли (admin/organizer/participant) | Своя БД: users, roles |
| **notification-service** | Notification | Уведомления о регистрациях, подтверждениях, изменениях мероприятий; простая доставка | Своя БД: notifications, delivery_log |
| **api-gateway** | Cross-cutting | Единая точка входа, маршрутизация, агрегация health, JWT-валидация (★ Keycloak) | Без собственной БД |

### Границы данных

Каждый сервис владеет только собственными данными. Сервисы не обращаются к чужим таблицам напрямую — только через REST API (Feign-клиент).

### Параллелизм в scope модуля

Модуль 6 вводит параллельные синхронные fan-out вызовы к независимым сервисам с ограничением параллелизма, явными timeout/cancel budget и изоляцией request-context между ветками.

### Паттерны в scope модуля

| Паттерн | Где применяется | Обязательность |
|---|---|---|
| Bounded Context (DDD) | Декомпозиция сервисов по доменным границам | Обязательно |
| API Gateway | api-gateway сервис | Обязательно |
| Database per Service | Каждый сервис со своей БД | Обязательно |
| Feign Client | Межсервисные REST-вызовы | Обязательно |
| Retry | Повтор при временной недоступности downstream | Обязательно |
| Circuit Breaker | Защита от каскадных сбоев | Обязательно |
| Saga / Outbox | Распределённая операция (registration → event + notification) | Обязательно (Outbox polling, без Kafka) |
| WireMock Contract Testing | Изолированное тестирование межсервисных контрактов | Обязательно |
| Vault for Secrets | DB credentials и пароли в Vault | Обязательно |
| Centralized Auth (Keycloak) | JWT-токены через Keycloak | ★ Задание со звёздочкой |
| RestClient + HTTP Interface | Перспективный инструмент Spring 7 | Ознакомление |

### Что вне scope модуля 6

- Observability (Prometheus, Grafana, distributed tracing) — модуль 7
- Kafka и асинхронные события — модуль 8
- Kubernetes и кластерный деплой — модуль 7
- Service Discovery (Consul / K8s DNS / Istio) — модуль 7
- Debezium CDC и Kafka-based Outbox — модуль 8

---

## Перед стартом

- модуль 5 завершён: Spring Boot сервис с REST API, PostgreSQL, ролями и MockMvc-тестами работает;
- `./gradlew build` зелёный;
- Docker и `docker compose` установлены и работают;
- есть понимание HTTP, REST API, Spring Boot и JPA базиса;
- есть базовое понимание, что такое доменная модель и как бизнес-операции группируются вокруг сущностей.

---

## Как проверить результат

| Что проверяем | Как проверить | Что ожидаем |
|---|---|---|
| Compose-стек | `docker compose --profile full up -d` | все сервисы стартуют, healthchecks зелёные |
| Health каждого сервиса | HTTP-запрос к `/actuator/health` каждого сервиса | статус `UP` |
| API Gateway | HTTP-запрос через gateway (например, к списку мероприятий) | запрос маршрутизируется к event-service |
| End-to-end сценарий | Создание события → запись участника → уведомление | цепочка сервисов отрабатывает через gateway |
| Feign-клиент | Запись участника (registration-service вызывает event-service) | межсервисный вызов через Feign отрабатывает |
| Параллельные downstream-вызовы | Вызвать агрегирующий сценарий с искусственной задержкой в двух независимых сервисах | суммарная latency близка к самой медленной ветке, а не к сумме задержек двух сервисов |
| Retry + Circuit Breaker | Остановить downstream-сервис и вызвать через gateway | retry срабатывает, затем circuit открывается, gateway возвращает понятный ответ |
| Outbox | Инициировать распределённую операцию (регистрация) | outbox-таблица содержит запись, polling-обработчик доставляет |
| Outbox concurrency | Запустить 2+ экземпляра одного сервиса и инициировать распределённую операцию | каждая outbox-запись обработана ровно один раз (SKIP LOCKED предотвращает двойную обработку) |
| Изоляция контекста в fan-out | Одновременно выполнить 2 агрегирующих запроса от разных пользователей | роли, correlation id и результаты не смешиваются между параллельными ветками |
| Vault | Проверить конфигурацию сервисов | DB credentials читаются из Vault, а не из application.yml |
| Контрактные тесты | `./gradlew test` в каждом сервисе | WireMock-тесты контрактов зелёные |
| OpenAPI | Swagger UI на каждом сервисе | контракты доступны и читаются |
| Database per Service | Проверка подключений к БД | каждый сервис подключается к собственной БД |
| DDD-границы | Объяснение: почему сервисы выделены именно так | границы обоснованы доменной ответственностью, а не техническими слоями |
| Saga (теория) | Объяснение: почему одна операция через 2+ сервиса — проблема | проблема распределённых транзакций понята, Outbox как подход |
| Keycloak (★) | Получение JWT-токена через Keycloak | токен выдаётся, gateway валидирует, содержит роли |
| RestClient | Ознакомление с `@HttpExchange` | понимание перспективного направления Spring |

---

## Публичные ресурсы

### YouTube — видео по темам модуля

| Тема | Источник |
|---|---|
| Монолит vs микросервисы: когда не надо | [Монолит или микросервисы? Что выбрать в 2026 (Мокевнин + Солодкий, 1ч 43мин)](https://www.youtube.com/watch?v=Ya5oG9sPVBU) · [Микросервисы vs монолит — какую архитектуру выбрать (Павленко, 18мин)](https://www.youtube.com/watch?v=PmIrrFqOfn8) |
| DDD и bounded contexts | [Влад Хононов: Как DDD меняет разработку](https://www.youtube.com/watch?v=PRL3vVfv1dA) |
| Event Storming | [Сергей Баранов: моделирование микросервисов через Event Storming](https://www.youtube.com/watch?v=cG9DVbcPc9M) |
| Saga Pattern | [SAGA pattern. Choreography and Orchestration (javaguru)](https://www.youtube.com/watch?v=UhaZb0r8TlE) |

### DDD и декомпозиция микросервисов

| Тема | Источник |
|---|---|
| Декомпозиция: почему по доменам, а не по слоям | [Хабр — 5 паттернов против распределённого монолита (Прощаев, 2026)](https://habr.com/ru/companies/otus/articles/994140/) |
| Паттерны декомпозиции для начинающих | [Хабр — обзор паттернов (Громов)](https://habr.com/ru/companies/reksoft/articles/864206/) |
| Bounded Context Canvas | [Перевод Владика Хоненкова (Хабр)](https://habr.com/ru/companies/oleg-bunin/articles/500506/) |
| Event Storming на практике | [Сергей Баранов (Хабр)](https://habr.com/ru/company/oleg-bunin/blog/537862/) |
| DDD: монолиты или микросервисы | [vc.ru — DDD про моделирование, а не архитектуру деплоя](https://vc.ru/dev/2004403-ddd-monolity-ili-mikroservisy-dlya-biznes-logiki) |
| Микросервисные паттерны | [Microservices.io — Chris Richardson](https://microservices.io/patterns/index.html) · [VK Cloud: 26 паттернов](https://cloud.vk.com/blog/26-osnovnyh-patternov-mikroservisnoj-razrabotki/) |
| Каталог ресурсов по интеграции | [system-design.space — микросервисы и интеграция](https://system-design.space/theme/microservices-integration/) |

### Монолит vs микросервисы: когда НЕ нужны микросервисы

| Тема | Источник |
|---|---|
| Возможно, микросервисы вам не нужны | [Хабр (RuVDS) — Возможно, микросервисы вам не нужны (2024)](https://habr.com/ru/companies/ruvds/articles/819941/) |
| Monolith First: почему не стартуют с микросервисов | [Хабр (Qtim) — Почему вам не нужны микросервисы для старта нового проекта (2024)](https://habr.com/ru/companies/qtim/articles/826916/) |
| Микросервисы vs Монолит: плюсы и минусы | [Хабр (CUSTIS, Шаталкин, 2025)](https://habr.com/ru/companies/custis/articles/894438/) |
| Когда пора внедрять микросервисы | [Хабр (Слёрм) — Микросервисы: плюсы, минусы, когда и зачем внедрять (2022)](https://habr.com/ru/companies/slurm/articles/674600/) |

### API Gateway (Spring Cloud Gateway)

| Тема | Источник |
|---|---|
| Spring Cloud Gateway | [Spring Docs — Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/reference/) |
| Getting Started Guide | [Spring — Building a Gateway](https://spring.io/guides/gs/gateway) |
| Практика | [Baeldung — Spring Cloud Gateway](https://www.baeldung.com/spring-cloud-gateway) |

### Database per Service

| Тема | Источник |
|---|---|
| Паттерн Database per Service | [Microservices.io — Database per Service](https://microservices.io/patterns/data/database-per-service.html) |
| Shared Database антипаттерн | [Microservices.io — Shared Database](https://microservices.io/patterns/data/shared-database.html) |

### Межсервисные вызовы: Feign + RestClient

| Тема | Источник |
|---|---|
| Spring Cloud OpenFeign | [Spring Docs — OpenFeign](https://docs.spring.io/spring-cloud-openfeign/reference/) |
| Feign: практический обзор | [Baeldung — Spring Cloud Feign](https://www.baeldung.com/spring-cloud-feign) |
| RestClient: новый стандарт Spring | [spring.io — The state of HTTP clients in Spring](https://spring.io/blog/2025/09/30/the-state-of-http-clients-in-spring) |
| HTTP Interface (@HttpExchange) | [Baeldung — HTTP Interface в Spring 6](https://www.baeldung.com/spring-6-http-interface) |

### Retry + Circuit Breaker (Resilience4j)

| Тема | Источник |
|---|---|
| Resilience4j | [Resilience4j Docs](https://resilience4j.readme.io/) |
| Spring Cloud Circuit Breaker | [Spring Docs — Circuit Breaker](https://docs.spring.io/spring-cloud-circuitbreaker/reference/) |
| Практика | [Baeldung — Resilience4j](https://www.baeldung.com/resilience4j) |
| Паттерны устойчивости | [Microservices.io — Circuit Breaker](https://microservices.io/patterns/reliability/circuit-breaker.html) · [Microservices.io — Retry](https://microservices.io/patterns/reliability/retry.html) |

### Saga Pattern и Transactional Outbox

| Тема | Источник |
|---|---|
| Saga Pattern | [Microservices.io — Saga](https://microservices.io/patterns/data/saga.html) |
| Transactional Outbox | [Microservices.io — Outbox](https://microservices.io/patterns/data/transactional-outbox.html) |
| Saga: практический разбор (рус) | [Proselyte — Saga Pattern](https://proselyte.net/saga-pattern/) |
| Outbox: надёжная доставка событий | [lightboxapi.ru — Outbox Pattern](https://lightboxapi.ru/blog/outbox-pattern-reliable-event-delivery) |
| Saga в микросервисах (рус) | [Purpleschool — SAGA паттерн](https://purpleschool.ru/knowledge-base/microservices/reliability/saga-mikroservisy) |
| Microsoft Learn — Saga | [Microsoft — Saga pattern (рус)](https://learn.microsoft.com/ru-ru/azure/architecture/patterns/saga) |

### Contract Testing (WireMock)

| Тема | Источник |
|---|---|
| WireMock — Getting Started | [WireMock Docs](https://wiremock.org/docs/) |
| WireMock — JUnit 5 | [WireMock — JUnit Jupiter](https://wiremock.org/docs/junit-jupiter/) |
| WireMock — Spring Boot | [WireMock — Spring Boot Integration](https://wiremock.org/docs/spring-boot/) |
| Введение в WireMock | [Baeldung — Introduction to WireMock](https://www.baeldung.com/introduction-to-wiremock) |

### HashiCorp Vault для секретов

| Тема | Источник |
|---|---|
| Spring Guide: Vault Config | [Spring — Getting Started with Vault](https://spring.io/guides/gs/vault-config) |
| Vault + Spring: reload secrets | [HashiCorp — Spring Vault Tutorial](https://developer.hashicorp.com/vault/tutorials/app-integration/spring-reload-secrets) |
| Vault на практике (рус) | [Хабр — практика Vault, AppRole](https://habr.com/ru/articles/653927/) |
| Введение в Vault (рус) | [dotsandbrackets — секреты приложения](https://dotsandbrackets.com/application-secrets-ru/) |

### Keycloak (★)

| Тема | Источник |
|---|---|
| Keycloak + Spring Boot | [Baeldung — Spring Boot Keycloak](https://www.baeldung.com/spring-boot-keycloak) |
| Полный гайд: Docker + Keycloak + Spring | [IAMDevBox — Keycloak Spring Boot OAuth2](https://iamdevbox.hashnode.dev/keycloak-spring-boot-oauth2-integration-complete-developer-guide) |
| Сравнение Auth0 vs Keycloak vs Spring Auth Server | [Medium — сравнение](https://medium.com/@ilya.kovalkov/auth0-vs-keycloak-vs-spring-authorization-server-ba3350fae8c9) |

### Внешние интеграции (★)

| Тема | Источник |
|---|---|
| DaData — подсказки адресов | [DaData API](https://dadata.ru/api/suggest/address/) · [Spring-клиент](https://github.com/KuliginStepan/dadata-client) |
| iCal4j — генерация .ics | [iCal4j Documentation](https://www.ical4j.org/) |
| Telegram Bot API | [Telegram Bot API](https://core.telegram.org/bots/api) · [spring-boot-starter-telegram (Habr)](https://habr.com/ru/articles/790826/) |
| Resend — email API | [Resend Docs](https://resend.com/docs) |
| Mailpit — email sandbox | [Mailpit GitHub](https://github.com/axllent/mailpit) |

### Health Checks (Spring Actuator)

| Тема | Источник |
|---|---|
| Spring Boot Actuator | [Spring Docs — Actuator](https://docs.spring.io/spring-boot/reference/actuator/) |
| Actuator Endpoints | [Spring Docs — Endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html) |
| Полный гайд Actuator | [Хабр](https://habr.com/ru/companies/otus/articles/1008360/) |

### SLI/SLO/SLA

| Тема | Источник |
|---|---|
| Google SRE: SLI/SLO/SLA | [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) |
| SLI/SLO на практике | [Хабр (Авито) — SLO: как мы начали измерять надёжность](https://habr.com/ru/companies/avito/articles/501864/) |
| SRE для разработчиков | [Хабр (Яндекс) — Практика SRE](https://habr.com/ru/companies/yandex/articles/439546/) |
| Error budget и SLO | [Google SRE Workbook — Error Budgets](https://sre.google/workbook/error-budget-policy/) |

### Docker Compose Multi-Service

| Тема | Источник |
|---|---|
| Docker Compose Profiles | [Docker Docs — Profiles](https://docs.docker.com/compose/profiles/) |
| Compose networking | [Docker Docs — Networking](https://docs.docker.com/compose/networking/) |
| Multi-service Compose | [Хабр — Docker Compose для нескольких сервисов](https://habr.com/ru/articles/703594/) |

### AI-инструменты для микросервисной разработки

| Тема | Источник |
|---|---|
| Docker MCP — управление compose через AI: старт/стоп отдельных сервисов, health aggregation | [GitHub](https://github.com/ckreiling/mcp-server-docker) |
| Swagger MCP — генерация MCP-инструментов из OpenAPI: AI проверяет совместимость контрактов | [GitHub](https://github.com/LostInBrittany/swagger-to-mcp-generator) |
| Microcks (опционально) — API mocking и контрактное тестирование | [microcks.io](https://microcks.io/) |
| Per-service AGENTS.md — файлы правил в каждом микросервисе: bounded context, контракт, зависимости | — |
| Принцип: AI должен понимать границы сервиса; per-service инструкции обеспечивают контекст | — |
