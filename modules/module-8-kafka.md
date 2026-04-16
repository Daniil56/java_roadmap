# Модуль 8: События и Kafka

## Содержание

- [Для кого этот модуль](#для-кого-этот-модуль)
- [Зачем этот модуль](#зачем-этот-модуль)
- [Что меняется по сравнению с модулем 7](#что-меняется-по-сравнению-с-модулем-7)
- [FR и NFR](#fr-и-nfr)
- [Технологический стек и инструменты](#технологический-стек-и-инструменты)
- [Что строим / как проект эволюционирует](#что-строим--как-проект-эволюционирует)
- [Архитектура модуля 8: асинхронный контур микросервисов](#архитектура-модуля-8-асинхронный-контур-микросервисов)
- [Перед стартом](#перед-стартом)
- [Как проверить результат](#как-проверить-результат)
- [Публичные ресурсы](#публичные-ресурсы)

---

## Для кого этот модуль

**Модуль 7 завершён** -- микросервисный стек из 4 бизнес-сервисов + API gateway развёрнут в Kubernetes через Helm umbrella chart, health probes настроены, Prometheus + Grafana собирают метрики, Jaeger показывает distributed traces, EFK/Kibana агрегирует логи.

**Или вы не проходили модули 1--7**, но знакомы с Java, Spring Boot, Docker, PostgreSQL, микросервисной архитектурой и Kubernetes. В таком случае стартуйте отсюда -- потребуется самостоятельно воспроизвести FR/NFR предыдущих модулей.

---

## Зачем этот модуль

Модули 6-7 построили синхронную микросервисную архитектуру: сервисы общаются через REST (Feign-клиент), каждый вызов блокирует поток до ответа downstream-сервиса. Это работает, но создаёт проблемы: если notification-service недоступен, регистрация участника может зависнуть или упасть; если event-service перезагружается, запросы теряются.

Kafka решает эти проблемы через асинхронную модель: сервис публикует событие о совершившемся бизнес-действии и не ждёт реакции. Обработчик (consumer) читает событие тогда, когда готов, и выполняет побочный эффект независимо. Это развязывает сервисы по времени и надёжности.

Появляется асинхронный контур. События -- это "что произошло в бизнесе", а не "что произошло в коде". "RegistrationCreated" = бизнес-факт (участник записался), а не техническое событие. Saga обеспечивает консистентность распределённой бизнес-операции, а не просто цепочку HTTP-вызовов.

Модуль вводит Kafka как транспортный слой для уже существующего проекта: Saga (choreography) заменяет синхронные цепочки вызовов, Transactional Outbox обеспечивает атомарность между БД и брокером, Kafka Streams добавляет stateful-обработку событий, а идемпотентный consumer защищает от дублей.

Общие правила roadmap, FR/NFR и принцип "один проект развивается от модуля к модулю" описаны в [корневом README roadmap](../README.MD).

**Результат модуля:** микросервисный стек получает асинхронный контур: Saga (choreography) для распределённых операций, producer/consumer для доменных событий, Kafka Streams для stateful-обработки, идемпотентный consumer для защиты от дублей и Transactional Outbox для надёжности доставки.

---

## Что меняется по сравнению с модулем 7

| Что было (M7 -- синхронные REST) | Что становится (M8 -- асинхронные события) |
|---|---|
| Feign-клиент: синхронный REST-вызов между сервисами | Kafka producer/consumer: асинхронная доставка событий |
| Outbox polling (`@Scheduled` + `SELECT FOR UPDATE SKIP LOCKED`) | Transactional Outbox через Kafka Connect / Debezium CDC |
| Circuit Breaker для защиты от недоступных сервисов | Retry + DLQ для обработки ошибок в асинхронном контуре |
| Ошибка downstream блокирует операцию | Saga (choreography): каждый сервис реагирует независимо |
| Нет stateful-обработки потока событий | Kafka Streams: stateless (filter/map) + stateful (aggregate/join) |
| Метрики HTTP-запросов | Consumer lag, delivery metrics, throughput |
| Межсервисная связь через REST API | Межсервисная связь через Kafka topics |

---

## FR и NFR

> Общая рамка: [README -- Как работать с требованиями](../README.MD#как-работать-с-требованиями).

### Функциональные требования

**Kafka Producer/Consumer:**

- Сервис публикует доменное событие после успешного бизнес-действия (producer)
- Consumer читает событие и выполняет проверяемый побочный эффект
- Для события зафиксирован topic, ключ маршрутизации и формат сериализации
- Docker Compose поднимает приложение, PostgreSQL и Kafka одной командой

**Saga Pattern (Choreography):**

- Saga (choreography): цепочка событий обеспечивает консистентность распределённой операции
- Регистрация участника инициирует цепочку: registration-created -- event-capacity-updated -- notification-sent
- Каждый сервис реагирует на событие независимо, без central orchestrator
- Compensating transaction: при ошибке в цепочке сервис публикует событие-компенсацию

**Kafka Streams:**

- Kafka Streams pipeline: stateless (filter/map/branch) + stateful (aggregate/join) обработка
- Поток событий агрегируется для бизнес-метрик (например, количество регистраций по мероприятию)
- TopologyTestDriver обеспечивает тестирование Streams-топологии без реального брокера

**Идемпотентность:**

- Идемпотентный consumer: повторная доставка не ломает состояние
- Защита от дублей через unique constraint или offset tracking

**Параллелизм в async-обработке:**

- Параллелизм consumer согласован с количеством партиций, лимитами downstream-систем и характером работы
- Независимые побочные эффекты внутри одного Saga-шага могут выполняться параллельно, если это не ломает порядок обработки по ключу и идемпотентность
- Временный контекст обработки изолирован границами одного сообщения и очищается после завершения
- Если для дедупликации, кэша или промежуточных результатов используется память процесса, выбран потокобезопасный или иммутабельный способ доступа
- Ошибки в асинхронных ветках и Saga-шаге обрабатываются явно и не теряются

**★ Виртуальные потоки:**

- Для I/O-bound обработки можно оценить использование виртуальных потоков
- Если такой режим включён, задокументированы риски pinning, starvation и согласованность с connection pool

**Transactional Outbox (рекомендуется):**

- Атомарная запись бизнес-данных + событие в outbox-таблицу
- Kafka Connect / Debezium CDC читает outbox и публикует в Kafka
- Альтернатива: polling outbox из модуля 6, усиленный Kafka publisher

### Нефункциональные требования

- Локальный compose-контур воспроизводим без ручной доводки
- Saga-цепочка диагностируется по логам и трейсам
- Kafka Streams pipeline тестируется через TopologyTestDriver
- Consumer lag и delivery metrics доступны через Actuator/Micrometer
- Идемпотентность проверяема: повторная отправка того же сообщения не создаёт дубль
- Для каждого topic обосновано количество партиций: целевой throughput, размер consumer group и порядок обработки (key-based ordering) учтены при выборе partition count
- Retry и DLQ не подменяют обязательный простой сценарий
- Расширенный путь не ломает базовый: Outbox/Debezium накладывается поверх работающего producer/consumer
- Параллелизм consumer задокументирован: как он согласован с partition count, downstream-лимитами и порядком обработки по ключу
- Если внутри обработчика есть параллельные ветки, они не ломают идемпотентность, не смешивают контекст сообщений и не скрывают ошибки
- Если shared state используется внутри процесса, его модель доступа безопасна при параллельной обработке и понятна по объяснению
- Если виртуальные потоки включены, их использование документировано вместе с runtime-ограничениями и способами диагностики

**AI-инструменты и агентная работа:**

- Kafka MCP подключён: AI читает сообщения, анализирует consumer lag, управляет topics — AI выступает как Kafka-оператор, а не заменяет понимание producer/consumer
- Event schemas проверяются через AI: обязательные поля, типы, backward compatibility
- Saga-цепочка трассируется: AI помогает проследить цепочку событий от start до finish
- Доменные события отделены от технических: "RegistrationCreated" = бизнес-факт, "MessageSentToKafka" = техническое
- Сгенерированный AI код (producer, consumer, Streams topology) проверяется через TopologyTestDriver, интеграционные тесты с Testcontainers и запуск compose-контура — AI не снимает ответственности за корректность асинхронного взаимодействия

### Приоритизация тем

| Приоритет | Тема | Почему |
|---|---|---|
| Обязательно | Kafka core: Producer, Consumer, Topic, Partition, Consumer Group | Базовый навык -- запустить асинхронный поток событий |
| Обязательно | Producer patterns: sync/async send, callbacks, serialization | Правильно отправить событие -- первая задача |
| Обязательно | Consumer patterns: offset commit, consumer groups, rebalancing | Правильно прочитать событие -- вторая задача |
| Обязательно | Saga (choreography) для распределённой операции | Заменяет синхронные цепочки из модуля 6 |
| Обязательно | Идемпотентный consumer | Без идемпотентности асинхронный контур ненадёжен |
| Обязательно | Kafka Streams: stateless + stateful обработка | Stateful processing поверх потока событий |
| Рекомендуется | Transactional Outbox через Debezium CDC | Надёжность доставки между БД и брокером |
| Рекомендуется | Retry/DLQ (non-blocking retries + Dead Letter Topic) | Безнадёжные сообщения не теряются |
| Рекомендуется | Exactly-once semantics | Transactional producer + idempotent consumer |
| Рекомендуется | Consumer lag monitoring | Операционная видимость асинхронного контура |
| Рекомендуется | Параллелизм consumer и async-шагов: partitions, downstream limits, shared state, error handling | Это уже Kafka-специфика, без повтора требований модулей 6-7 |
| Опционально | ★ Виртуальные потоки (Java 21+) для I/O-bound consumer processing | Отдельно оцениваются для I/O-bound обработки |
| Опционально | Schema Registry (Avro/Protobuf) | Типизация контрактов событий на уровне брокера |
| Опционально | Kafka Streams DSL: KTable/KStream advanced joins | Сложные stateful-операции |

### Что вне scope модуля

- Kubernetes deployment Kafka -- предполагается локальный compose-контур или managed Kafka
- Kafka Connect clusters и connectors в production-конфигурации
- Multi-datacenter replication
- KSQL/ksqlDB
- Глубокое изучение Java Memory Model и низкоуровневых concurrency-примитивов
- Structured Concurrency (JDK 25/26 preview)

---

## Технологический стек и инструменты

| Категория | Что используем | Минимум для Done |
|---|---|---|
| Брокер сообщений | Apache Kafka | Локальный compose-контур: приложение + PostgreSQL + Kafka |
| Producer | Spring for Apache Kafka | Сервис публикует доменное событие после бизнес-действия |
| Consumer | Spring for Apache Kafka | Consumer читает событие и выполняет побочный эффект |
| Сериализация | JSON (основной) / Avro (опционально) | Сообщения читаемы и воспроизводимы |
| Saga | Choreography через Kafka topics | Цепочка событий обеспечивает распределённую операцию |
| Streams | Kafka Streams (Spring Kafka Streams) | Stateless + stateful обработка потока событий |
| Outbox | Debezium CDC / polling outbox | Атомарная запись БД + событие |
| Идемпотентность | Unique constraint / offset tracking | Повторная доставка не ломает состояние |
| Retry/DLQ | Spring Kafka Non-Blocking Retries | Безнадёжные сообщения ушли в DLQ |
| Тестирование | TopologyTestDriver + Testcontainers | Streams тестируется без брокера, интеграция -- с брокером |
| Мониторинг | Micrometer + Actuator | Consumer lag, delivery metrics, throughput |
| Инфраструктура | Docker Compose | Один сценарий: app + DB + Kafka (+ Zookeeper/KRaft) |
| AI | Qwen Code CLI + Kafka MCP (Confluent / kafka-mcp-server) | Event-driven AI-оператор: Saga chain tracing, event schema review, consumer lag анализ |

---

## Что строим / как проект эволюционирует

**Стартовая точка:** микросервисный стек из модулей 6--7 -- 4 бизнес-сервиса + API gateway, синхронные REST-вызовы через Feign, Outbox polling, WireMock контрактные тесты, observability stack.

### До -- после

| Слой / аспект | До (модуль 7) | После (модуль 8) |
|---|---|---|
| Межсервисное взаимодействие | Синхронный REST (Feign) | Асинхронные события через Kafka |
| Распределённая операция | Outbox polling без брокера | Saga (choreography) через Kafka topics |
| Обработка ошибок | Circuit Breaker (Resilience4j) | Retry + DLQ для асинхронного контура |
| Stateful обработка | Нет | Kafka Streams: aggregate, join, windowed operations |
| Идемпотентность | Не обеспечивалась | Consumer защищён от дублей |
| Надёжность публикации | Polling outbox | Transactional Outbox через Debezium CDC |
| Наблюдаемость | HTTP-метрики (Prometheus) | Consumer lag, throughput, delivery metrics |

### К концу модуля система умеет

- публиковать доменное событие после бизнес-действия и не ждать реакции
- обрабатывать события через consumer с проверяемым побочным эффектом
- обеспечивать Saga (choreography): цепочка событий проходит через несколько сервисов
- защищать consumer от дублей через идемпотентность
- выполнять stateful-обработку потока событий через Kafka Streams
- уводить безнадёжные сообщения в DLQ после исчерпания retry
- наблюдать consumer lag и delivery metrics через Micrometer/Actuator
- подниматься целиком через `docker compose` с Kafka-брокером

---

## Архитектура модуля 8: асинхронный контур микросервисов

Синхронные REST-вызовы из модулей 6--7 дополняются асинхронными Kafka-событиями. Каждый сервис становится одновременно producer-ом (публикует события о своих бизнес-действиях) и consumer-ом (реагирует на события других сервисов).

### Топики и потоки событий

| Topic | Producer | Consumer | Событие |
|---|---|---|---|
| `registration-events` | registration-service | event-service, notification-service | Создание/отмена регистрации |
| `event-events` | event-service | registration-service, notification-service | Создание/изменение мероприятия, обновление вместимости |
| `notification-events` | notification-service | notification-service (внутренняя обработка) | Запрос на отправку уведомления |
| `registration-stats` (Kafka Streams) | Kafka Streams pipeline | API для запроса статистики | Агрегированная статистика регистраций |

### Saga (Choreography): цепочка регистрации

Регистрация участника на мероприятие -- распределённая операция, затрагивающая 3 сервиса:

```text
registration-service: publishes registration-created
  -> event-service: consumes, updates capacity, publishes capacity-updated
    -> notification-service: consumes, sends notification, publishes notification-sent
      -> registration-service: consumes, marks registration as completed
```

При ошибке на любом этапе сервис публикует compensating event, который откатывает изменения в предыдущих сервисах.

### Kafka Streams: агрегация событий

Kafka Streams читает `registration-events` и агрегирует статистику:

- количество регистраций по мероприятию (KTable + aggregate)
- оконная агрегация: регистрации за последний час (windowed aggregation)
- фильтрация: только подтверждённые регистрации (filter)

Streams-топология тестируется через TopologyTestDriver без реального брокера.

### Идемпотентный consumer

Каждый consumer проверяет, было ли событие уже обработано:

- unique constraint на event_id в таблице обработанных событий
- или offset tracking с проверкой дедупликации
- повторная доставка того же сообщения не создаёт дубль и не ломает состояние

### Параллелизм в scope модуля

Модуль 8 добавляет Kafka-специфику параллелизма: параллелизм consumer, независимые async-ветки без нарушения ordering/idempotency, message-local context, shared state и опциональные виртуальные потоки.

---

## Перед стартом

- модуль 7 завершён: микросервисный стек работает в Kubernetes, observability подключена;
- `./gradlew build` зелёный во всех сервисах;
- Docker и `docker compose` установлены;
- есть понимание разницы между синхронным REST-вызовом и асинхронным событием;
- есть базовое понимание транзакций и Outbox pattern из модуля 6.

---

## Как проверить результат

| Что проверяем | Как проверить | Что ожидаем |
|---|---|---|
| Compose-стек с Kafka | `docker compose up -d` | приложение, PostgreSQL, Kafka запущены |
| Producer | Выполнить бизнес-действие (регистрация) | в логах видна публикация события в topic |
| Consumer | Проверить логи downstream-сервиса | consumer принял событие, побочный эффект выполнен |
| Saga (choreography) | Инициировать регистрацию через gateway | цепочка событий проходит: registration -- capacity -- notification |
| Compensating transaction | Эмулировать ошибку в цепочке | compensating event опубликован, изменения откачены |
| Идемпотентность | Отправить дубликат события | повторная доставка не создаёт дубль в БД |
| Kafka Streams | Проверить output topic агрегации | агрегированная статистика обновляется |
| TopologyTestDriver | `./gradlew test` | Streams-топология тестируется без брокера |
| Retry/DLQ | Эмулировать ошибку обработки | retry срабатывает, затем сообщение уходит в DLQ |
| Consumer lag | Проверить Micrometer/Actuator | lag виден как метрика, растёт под нагрузкой |
| Partition sizing | Проверить конфигурацию topic-ов | для каждого topic указан partition count с учётом throughput, consumer group size и ordering |
| Transactional Outbox | Проверить outbox-таблицу | бизнес-данные + событие записаны атомарно, Debezium публикует |
| Параллелизм consumer | Проверить конфигурацию обработки и нагрузить consumer | значение обосновано: согласовано с partition count, downstream-лимитами и целевым throughput |
| Async-обработчик: ветки, контекст, shared state | Эмулировать независимые побочные эффекты и параллельные сообщения | порядок по ключу и идемпотентность не ломаются, контекст не смешивается, гонок в памяти нет |
| Виртуальные потоки (★) | Включить `spring.threads.virtual.enabled=true` и нагрузить consumer | режим задокументирован, не конфликтует с доступом к БД и внешним вызовам, ожидаемая польза подтверждена замером |

---

## Публичные ресурсы

### Apache Kafka: основы

| Тема | Источник |
|---|---|
| Kafka Quickstart | [kafka.apache.org -- Quickstart](https://kafka.apache.org/quickstart) |
| Kafka Documentation | [kafka.apache.org -- Documentation](https://kafka.apache.org/documentation/) |
| Kafka Design | [kafka.apache.org -- Design](https://kafka.apache.org/documentation/#design) |
| Введение в Kafka (рус) | [Хабр -- Apache Kafka за 45 минут](https://habr.com/ru/articles/440400/) |
| Выбор количества партиций | [Confluent -- How to Choose the Number of Topics/Partitions in a Kafka Cluster](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster/) |
| Event-driven микросервисы | [Хабр -- Event-driven микросервисы с Spring Boot и Kafka](https://habr.com/ru/companies/spring_aio/articles/874488/) |

### Spring for Apache Kafka

| Тема | Источник |
|---|---|
| Spring Kafka Reference | [docs.spring.io -- Spring for Apache Kafka](https://docs.spring.io/spring-kafka/reference/) |
| Spring Kafka Non-Blocking Retries | [docs.spring.io -- Retry Topic](https://docs.spring.io/spring-kafka/reference/retrytopic.html) |
| Spring Kafka Exception Handling | [docs.spring.io -- Error Handling](https://docs.spring.io/spring-kafka/reference/kafka/annotation-error-handling.html) |
| Spring Kafka Monitoring | [docs.spring.io -- Micrometer](https://docs.spring.io/spring-kafka/reference/kafka/micrometer.html) |
| Kafka Consumer в Spring (рус) | [Хабр -- Как работать с Kafka-consumer в Spring](https://habr.com/ru/articles/793134/) |

### Saga Pattern

| Тема | Источник |
|---|---|
| Saga Pattern | [microservices.io -- Saga](https://microservices.io/patterns/data/saga.html) |
| Saga: практический разбор (рус) | [Proselyte -- Saga Pattern](https://proselyte.net/saga-pattern/) |
| Saga в микросервисах (рус) | [Purpleschool -- SAGA паттерн](https://purpleschool.ru/knowledge-base/microservices/reliability/saga-mikroservisy) |
| Microsoft Learn -- Saga | [Microsoft -- Saga pattern (рус)](https://learn.microsoft.com/ru-ru/azure/architecture/patterns/saga) |
| Choreography vs Orchestration | [Хабр -- Паттерны микросервисной транзакционности](https://habr.com/ru/companies/otus/articles/994140/) |

### Transactional Outbox

| Тема | Источник |
|---|---|
| Transactional Outbox | [microservices.io -- Outbox](https://microservices.io/patterns/data/transactional-outbox.html) |
| Debezium Outbox Event Router | [debezium.io -- Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html) |
| Outbox на Java-примере (рус) | [Хабр -- Transactional Outbox на примере двух микросервисов](https://habr.com/ru/articles/991934/) |
| Outbox: консистентность (рус) | [Хабр -- Transactional Outbox: обеспечиваем консистентность](https://habr.com/ru/articles/996080/) |
| Outbox: надёжная доставка | [lightboxapi.ru -- Outbox Pattern](https://lightboxapi.ru/blog/outbox-pattern-reliable-event-delivery) |

### Kafka Streams

| Тема | Источник |
|---|---|
| Kafka Streams Developer Guide | [kafka.apache.org -- Streams](https://kafka.apache.org/documentation/streams/developer-guide/) |
| Spring Kafka Streams | [docs.spring.io -- Kafka Streams](https://docs.spring.io/spring-kafka/reference/kafka/streams.html) |
| TopologyTestDriver | [kafka.apache.org -- Testing](https://kafka.apache.org/documentation/streams/developer-guide/testing.html) |
| Kafka Streams: обзор (рус) | [Хабр -- Kafka Streams: когда данных много и их нужно обработать](https://habr.com/ru/companies/slurm/articles/735116/) |

### Retry, DLQ, идемпотентность

| Тема | Источник |
|---|---|
| Kafka DLQ (рус) | [Хабр -- Kafka: обработка ошибок и Dead Letter Queues](https://habr.com/ru/articles/989608/) |
| Идемпотентный consumer | [Хабр -- Как работать с Kafka-consumer в Spring](https://habr.com/ru/articles/793134/) |
| Consumer Lag (рус) | [Хабр -- Управление consumer lag в Kafka](https://habr.com/ru/companies/otus/articles/905804/) |

### Тестирование Kafka-контура

| Тема | Источник |
|---|---|
| Testcontainers Kafka Module | [testcontainers.org -- Kafka Module](https://java.testcontainers.org/modules/kafka/) |
| Тестирование Kafka-контракта (рус) | [Хабр -- Тестирование асинхронного контракта](https://habr.com/ru/articles/824594/) |

### Видео

| Тема | Источник |
|---|---|
| Kafka: погружение (рус) | [Григорий Кошелев -- Apache Kafka за 45 минут](https://rutube.ru/video/fab4461afe92a29365460f186ab36f42/) |
| Outbox + Debezium (рус) | [Kafka Connect, Debezium и OUTBOX pattern](https://www.youtube.com/watch?v=EP1i4fowjjg) |

### AI-инструменты для работы с Kafka

| Тема | Источник |
|---|---|
| Confluent MCP — topic management, consumer groups, Schema Registry | [GitHub](https://github.com/confluentinc/mcp-confluent) |
| kafka-mcp-server — лёгкий Kafka MCP: producer/consumer, topics | [GitHub](https://github.com/tuannvm/kafka-mcp-server) |
| Принцип: AI отлаживает Saga-цепочки и проверяет event schemas; результат верифицируется через тесты | — |

### Многопоточность: Kafka consumer threading

| Тема | Источник |
|---|---|
| Multi-threaded Kafka consumer | [Confluent -- Multi-Threaded Messaging with Kafka Consumer](https://www.confluent.io/blog/kafka-consumer-multi-threaded-messaging/) |
| Spring Kafka thread safety | [Spring Kafka Docs -- Thread Safety](https://docs.spring.io/spring-kafka/reference/kafka/thread-safety.html) |
| Java concurrent package summary | [Oracle JDK 21 -- `java.util.concurrent`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html) |

### Многопоточность: виртуальные потоки (Virtual Threads)

| Тема | Источник |
|---|---|
| Oracle guide | [Oracle JDK 21 -- Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html) |
| JEP 444 | [OpenJDK -- Virtual Threads](https://openjdk.org/jeps/444) |
| Spring Boot reference | [Spring Boot -- Virtual Threads](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.virtual-threads) |

### Многопоточность: message-local context, ThreadLocal и shared state

| Тема | Источник |
|---|---|
| ThreadLocal API | [Oracle JDK 21 -- `ThreadLocal`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/ThreadLocal.html) |
| InheritableThreadLocal API | [Oracle JDK 21 -- `InheritableThreadLocal`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/InheritableThreadLocal.html) |
| Java concurrent package summary | [Oracle JDK 21 -- `java.util.concurrent`](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/package-summary.html) |
