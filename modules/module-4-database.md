# Модуль 4: База данных

## Содержание

- [Для кого этот модуль](#для-кого-этот-модуль)
- [Зачем этот модуль](#зачем-этот-модуль)
- [FR и NFR в этом модуле](#fr-и-nfr-в-этом-модуле)
- [Технологический стек и инструменты](#технологический-стек-и-инструменты)
- [Что строим / как проект эволюционирует](#что-строим--как-проект-эволюционирует)
- [Архитектура модуля 4: эволюция Repository Layer](#архитектура-модуля-4-эволюция-repository-layer)
- [Связь с предыдущими модулями](#связь-с-предыдущими-модулями)
- [Перед стартом](#перед-стартом)
- [Как проверить результат](#как-проверить-результат)
- [Публичные ресурсы](#публичные-ресурсы)

---

### Для кого этот модуль

**Целевая аудитория:** ты прошёл модуль 3 — party service уже контейнеризован, и готов перевести хранение состояния из памяти в реальную базу данных без перехода на Spring.

Если ты знаком с основами Java, CI/CD и Docker, и тебя не пугает Docker Compose — можешь приступать к модулю 4. Тебе понадобится самостоятельно реализовать основные функциональные требования из модулей 1 и 2 и прикрутить к ним базу данных.

## Зачем этот модуль

Модуль 3 научил собирать Docker-образ, запускать party service в контейнере и пользоваться `docker compose` для одного сервиса. Это решило задачу воспроизводимого запуска, но не решило задачу хранения данных: после остановки процесса данные живут только в памяти (или в CSV-файлах, если было выполнено бонусное задание ★ из модуля 2).

Ключевая цель модуля — заменить in-memory реализацию репозитория (модуль 2) на персистентное хранение в PostgreSQL. Граница между бизнес-логикой и хранением уже проведена в модуле 2 через интерфейс репозитория — теперь за этим интерфейсом появляется реальная БД: PostgreSQL для хранения, JDBC для доступа, Liquibase для миграций схемы, Testcontainers для проверки на реальной БД.

Схема БД — проекция предметной области. Каждая таблица, каждый столбец, каждое ограничение отражают бизнес-правила: лимит участников, уникальность email, статус регистрации. Каждое изменение схемы — бизнес-решение, а не технический артефакт. SQL-запрос формализует бизнес-вопрос: «сколько свободных мест на мероприятии?» — это не `SELECT COUNT`, а ответ на вопрос стейкхолдера.

Это переходный модуль перед Spring. В модуле 5 появятся Spring Boot, HTTP API, JPA и транзакции. Без модуля 4 эти абстракции будут восприниматься как магия — здесь ты проходишь весь путь руками: `compose` поднимает БД, миграции создают схему, JDBC-репозиторий выполняет SQL, Testcontainers доказывает, что persistence работает.

---

## FR и NFR в этом модуле

> Общая рамка: [README → Как работать с требованиями](../README.MD#как-работать-с-требованиями).

### Функциональные требования

**Наследование из модулей 1-2:** все FR сохраняются — создание мероприятия, регистрация, отмена, waitlist, undo.

**Новое FR:**

**Хранение:**
- Бизнес-данные хранятся в PostgreSQL и переживают перезапуск party service

> Если в модуле 2 было выполнено бонусное задание (★) с сохранением в CSV-файлы — это FR **заменяет** файловое хранение на реляционное. Если бонус не выполнялся — данные впервые перестают теряться при остановке процесса.

**Транзакции и целостность данных:**

С появлением БД бизнес-операции затрагивают несколько таблиц одновременно. Регистрация участника — это не один `INSERT`, а создание записи + обновление свободных мест. Без транзакций один шаг может пройти, а другой — нет, и данные рассинхронизируются.

- Регистрация участника на мероприятие атомарна: создание записи + обновление свободных мест выполняются как единая операция — либо оба шага завершаются, либо ни один
- Отмена регистрации атомарна: удаление записи + возврат места выполняются в рамках одной транзакции
- Мероприятие не может быть переподписано: количество подтверждённых регистраций не превышает вместимость (инвариант, который держится на уровне транзакции)
- При конкурирующих регистрациях на одно мероприятие инвариант вместимости сохраняется: одно свободное место не подтверждается двум участникам одновременно
- Для каждой бизнес-операции party service понятно, требует ли она транзакции и почему — на примере регистрации (`INSERT` + `UPDATE` в одной транзакции)

> ACID здесь — не абстрактная тема для изучения, а ответ на вопрос: «почему регистрация в party service не может быть просто двумя отдельными SQL-запросами?». Это создаёт фундамент для Spring `@Transactional` в модуле 5 — понятно, ЧТО именно Spring автоматизирует.

**Аналитика и отчёты (SQL-first):**

В модуле 2 группировка и фильтрация делались через Stream API в памяти. С появлением БД эти операции переходят на SQL — база считает сама, Java получает готовый результат:

- Показывать сводку по мероприятию: сколько регистраций, сколько свободных мест, сколько в waitlist, процент заполненности — одним запросом с агрегацией
- Группировать мероприятия по статусу заполненности: «полные», «свободные», «с waitlist» — через `GROUP BY` и `HAVING`
- Искать участников по фрагменту имени или email — через `ILIKE`, а не линейным проходом по коллекции
- Показывать рейтинг самых популярных мероприятий по числу регистраций — `ORDER BY ... DESC LIMIT`
- Фильтровать регистрации за период — по дате создания, используя типы `timestamp` в PostgreSQL

> Цель этих требований — показать, что SQL решает задачи аналитики и поиска эффективнее, чем Stream API по in-memory коллекциям. Ты не просто «переводишь HashMap на JDBC», а получаешь новый класс возможностей: агрегацию, полнотекстовый поиск, аналитические отчёты — всё это делает БД, а не Java-код.

### Нефункциональные требования

**Среда и воспроизводимость:**

- `docker compose` поднимает `party-app + postgres + pgAdmin` как локальную среду разработки
- Схема БД создаётся и обновляется только через Liquibase-миграции — без ручного `CREATE TABLE`
- Named volume из модуля 3 теперь хранит данные PostgreSQL между рестартами
- Среда воспроизводима на чистой машине одной командой

**Архитектура и качество:**

- In-memory реализация репозитория (из модуля 2) заменяется на JDBC-реализацию: `InMemoryXxxRepository` → `JdbcXxxRepository`
- Интерфейс репозитория не меняется — сервисный слой не замечает подмены хранения
- Доступ к данным через JDBC и `PreparedStatement` — без ORM
- Интеграционные тесты проверяют persistence на реальной PostgreSQL через Testcontainers — без зависимости от локально поднятой БД
- Переход к Spring в модуле 5 не требует переписывать доменную логику

**Нагрузка и производительность:**

- H2 и in-memory базы данных не используются даже для тестов — Testcontainers поднимает реальную PostgreSQL на лету из локального Docker
- Сервис обрабатывает до 100 одновременных регистраций без ошибок консистентности данных
- Консистентность вместимости обеспечивается транзакцией и состоянием БД, а не in-memory счётчиком в приложении
- Тест конкурирующих регистраций воспроизводим: несколько параллельных попыток записи на одно мероприятие подтверждают, что инвариант вместимости не нарушается
- CRUD-операции через JDBC выполняются за O(1) по ключу — паритет с in-memory реализацией из модуля 2
- Время отклика одной операции не превышает 50 мс при работе с локальной PostgreSQL в Docker
- Транзакционные границы в JDBC-коде явные: `autocommit` отключён для write-операций, `commit` / `rollback` управляются вручную — это то, что Spring `@Transactional` возьмёт на себя в модуле 5

**AI-инструменты и агентная работа:**

- Проектные инструкции для AI (CLAUDE.md) созданы и поддерживаются: структура БД, конвенции имён, ограничения
- Postgres MCP подключён; AI видит схему БД и помогает с EXPLAIN-анализом
- AI-предложения по SQL верифицируются через benchmark или EXPLAIN — AI выступает как DBA-консультант, а не заменяет написание SQL
- Сгенерированный AI SQL и миграции проверяются через сборку, тесты и запуск — AI не снимает ответственности за корректность схемы и запросов

### Практическая связь NFR

| NFR (требование) | Как проверить |
|---|---|
| Среда из двух сервисов | `docker compose up` поднимает party-app и postgres |
| Миграции схемы | На чистой БД Liquibase автоматически создаёт таблицы |
| Персистентность | Данные на месте после `docker compose down && docker compose up` |
| JDBC-доступ | `PreparedStatement` для всех параметризованных запросов |
| Persistence boundary | Сервисный слой не содержит SQL и `ResultSet` |
| Замена реализации | `InMemoryXxxRepository` заменён на `JdbcXxxRepository`, интерфейс не изменён |
| Интеграционные тесты | Testcontainers поднимает PostgreSQL, тесты проходят стабильно |
| Нагрузка | 100 регистраций без ошибок, одна операция < 50 мс |
| Транзакции | Регистрация и отмена выполняются в явных транзакционных границах (autocommit off) |
| ACID | Для каждой write-операции понятно, требует ли она транзакции и почему (Atomicity, Consistency) |
| Конкурирующие регистрации | Параллельный интеграционный тест не даёт переподписать мероприятие сверх вместимости |

---

## Технологический стек и инструменты

| Категория | Что используем | Минимум для Done |
|---|---|---|
| База данных | PostgreSQL | сервис БД поднимается в `docker compose` и хранит данные между рестартами |
| Java-доступ к БД | JDBC + PostgreSQL driver | чтение и запись идут через JDBC, без ORM |
| Миграции схемы | Liquibase | схема создаётся и обновляется только миграциями |
| Тестирование | Testcontainers for Java + PostgreSQL module | интеграционные тесты поднимают настоящую PostgreSQL автоматически |
| Оркестрация локальной среды | Docker Compose + named volume | локальная среда воспроизводима одной командой |
| Инспекция БД | pgAdmin (в Docker Compose), IntelliJ Database tool, DBeaver | можно проверить, что данные реально лежат в БД |
| Сборка | Gradle Wrapper | `./gradlew build` остаётся базовой командой качества |
| AI | Qwen Code CLI + Postgres MCP | CLAUDE.md для проекта, EXPLAIN-анализ через AI, верификация AI-SQL через benchmark |

---

## Что строим / как проект эволюционирует

**Стартовая точка:** party service из модуля 3 уже собран в Docker-образ, запускается в контейнере и имеет базовый `docker compose` для одного сервиса.

**Что меняется в модуле 4:**

- `docker compose` перестаёт быть обёрткой только вокруг party-app и становится локальной средой из трёх сервисов: `party-app`, `postgres` и `pgAdmin`
- named volume из модуля 3 получает практическую ценность: теперь он хранит данные PostgreSQL между рестартами
- состояние проекта переезжает из in-memory коллекций в PostgreSQL
- схема БД перестаёт быть “договорённостью в голове” и становится версионируемым артефактом через Liquibase
- in-memory реализация репозитория заменяется на JDBC-реализацию — интерфейс не меняется, SQL остаётся внутри `JdbcXxxRepository`
- интеграционные тесты перестают полагаться на локально поднятую вручную БД и начинают использовать Testcontainers

**Что строим к концу модуля:**

- локальную среду `party-app + PostgreSQL + pgAdmin`, поднимаемую через `docker compose`
- репозиторий на JDBC для ключевых сценариев проекта
- набор Liquibase-миграций, создающих и развивающих схему
- интеграционные тесты, которые доказывают, что миграции и JDBC-репозитории работают на реальной PostgreSQL

**Что сознательно не делаем:**

- не переходим на Spring Boot
- не скрываем SQL за ORM
- не оптимизируем PostgreSQL под production-тюнинг, репликацию или backup

**Итог модуля:**

Проект переводит репозиторий с in-memory хранения на PostgreSQL: данные переживают перезапуск, схема управляется миграциями, доступ идёт через JDBC, всё проверяется на реальной БД. Интерфейс репозитория из модуля 2 остаётся неизменным — меняется только реализация.

---

## Архитектура модуля 4: эволюция Repository Layer

Главная задача — заменить реализацию репозитория с in-memory на JDBC, не меняя интерфейс. Бизнес-логика из модулей 1-2 продолжает работать без изменений — она не знает, что за интерфейсом теперь PostgreSQL, а не `HashMap`.

| Слой | Что делает | Что меняется в модуле 4 |
|---|---|---|
| Entry point | принимает вход, запускает use case | не меняется |
| Service / use case | бизнес-операции и правила | не меняется — работает через интерфейс репозитория |
| Persistence contract | интерфейс доступа к данным | не меняется — тот же интерфейс из модуля 2 |
| JDBC implementation | SQL, маппинг строк в объекты, соединения | **НОВОЕ:** заменяет `InMemoryXxxRepository` |
| InMemory implementation | хранение в коллекциях | удаляется |
| Liquibase migrations | история схемы и изменения БД | **НОВОЕ:** схема управляется версионно |
| Integration tests | проверка репозитория на реальной PostgreSQL | **НОВОЕ:** Testcontainers вместо моков |

### Параллелизм в scope модуля

Модуль 4 вводит конкуренцию на уровне БД: инвариант вместимости держится транзакцией и состоянием PostgreSQL, а корректность при одновременных регистрациях подтверждается воспроизводимым интеграционным тестом.

---

## Связь с предыдущими модулями

### Что уже есть из модуля 3

- party service собран в Docker-образ
- базовый `docker compose` для одного сервиса
- named volume как технический элемент среды

### Что добавляется в модуле 4

- `docker compose` расширяется сервисом PostgreSQL
- named volume теперь хранит реальные данные БД
- переменные окружения для конфигурации подключения
- `docker compose` описывает зависимость party-app от БД

---

## Перед стартом

- Модуль 3 завершён, образ party service собирается и запускается вне IDE
- Базовый `docker compose` понятен хотя бы для одного сервиса
- Ты умеешь читать логи контейнера и проверять состояние среды
- Проект собирается через `./gradlew build`
- Repository Layer уже выделен (модуль 2): сервисный слой работает с данными через интерфейс репозитория, а не напрямую с коллекциями
- In-memory реализация репозитория понятна на уровне классов и сценариев
- Есть общее представление, что такое таблицы, строки и колонки (опыт работы с Excel или CSV достаточно)

---

## Как проверить результат

| Проверка | Что делаешь | Что должно получиться |
|---|---|---|
| Локальная среда | `docker compose up` для `party-app + postgres + pgAdmin` | все три сервиса стартуют без ручных промежуточных действий |
| Схема БД | запускаешь проект на чистой БД | Liquibase автоматически создаёт таблицы и ограничения |
| Persistence | выполняешь ключевой сценарий приложения | данные читаются и пишутся из PostgreSQL, а не из памяти |
| Замена реализации | сервисный слой использует JDBC-репозиторий | интерфейс репозитория не изменился, сервисы не переписывались |
| Сохранность | перезапускаешь контейнер PostgreSQL без удаления volume | записи остаются доступными |
| Инспекция | открываешь pgAdmin (или DBeaver) и подключаешься к PostgreSQL | строки видны в таблицах |
| Интеграционные тесты | запускаешь тесты репозитория | Testcontainers поднимает PostgreSQL, тесты проходят стабильно |
| Rollback | эмулируешь ошибку между изменением регистрации и обновлением вместимости | частично записанного состояния не остаётся |
| Конкурирующие регистрации | запускаешь параллельный тест на одно мероприятие с ограниченной вместимостью | число успешных регистраций не превышает вместимость |

### Признаки, что модуль ещё не закрыт

- схема создаётся руками, а не миграциями
- данные теряются при перезапуске контейнера
- интеграционные тесты зависят от локально поднятой вручную БД
- сервисный слой содержит SQL-строки или детали `ResultSet`

---

## Публичные ресурсы

### YouTube — видео по темам модуля

| Тема | Источник |
|---|---|
| PostgreSQL для начинающих | [Технострим ВШЭ](https://www.youtube.com/playlist?list=PLrCZzMIB1lOVHCZ0Uo5j0CKQo0M5oqLPW) — плейлист |
| SQL за 2 часа | [freeCodeCamp (русские субтитры)](https://www.youtube.com/watch?v=HXV3zeQKqGY) |
| SQL с нуля — полный курс | [Bogdan Stashchuk](https://www.youtube.com/watch?v=U4KESpMJAPY) |
| Docker Compose + PostgreSQL + pgAdmin | [Установка PostgreSQL через Docker Compose](https://www.youtube.com/watch?v=2vwwwA4AEyk) |
| JDBC: подключение к БД и CRUD | [JDBC + CRUD для начинающих](https://www.youtube.com/watch?v=7LMD8-AZcFM) |
| JDBC: транзакции (autocommit, commit, rollback) | [Образование онлайн (14 мин)](https://www.youtube.com/watch?v=zYXjXHRTR5k) |
| ACID: атомарность транзакций | [Владимир Кузнецов (13 мин)](https://www.youtube.com/watch?v=1ObmD7sE_MA) |
| Liquibase: управление миграциями | [Миграции БД на примере Liquibase](https://www.youtube.com/watch?v=tOGDqoP-MtY) |
| Миграции БД: Flyway и Liquibase | [Nerzon (31 мин)](https://www.youtube.com/watch?v=6u93yNqUJ9o) |
| Агрегатные функции PostgreSQL (GROUP BY, HAVING) | [Полный разбор COUNT, SUM, AVG, MAX, MIN](https://www.youtube.com/watch?v=0SaZkLkLWfs) |
| SQL: SELECT, JOIN, GROUP BY — обзор | [MySQL Crash Course (рус)](https://www.youtube.com/watch?v=IK6e1SFCdow) |
| Объединение таблиц: INNER, LEFT, RIGHT, FULL JOIN | [Обзор всех типов JOIN](https://www.youtube.com/watch?v=Tc3SxeYxhAU) |
| Индексы PostgreSQL: типы и оптимизация | [Оптимизация PostgreSQL: индексы, EXPLAIN (1 ч)](https://www.youtube.com/watch?v=gA3A_epB3So) |
| EXPLAIN: как читать план запроса | [Разбор EXPLAIN ANALYZE с примерами](https://www.youtube.com/watch?v=drnDr8M-W5M) |

### Курсы — SQL и PostgreSQL

| Курс | Ссылка |
|---|---|
| Интерактивный тренажер по SQL | [Stepik](https://stepik.org/course/63054/promo) — бесплатный, практические задания в браузере, победитель EdCrunch Award |
| SQL для начинающих (+ оконные функции) | [Stepik](https://stepik.org/course/130488/promo) — от простых запросов до оконных функций, PostgreSQL |
| SQLBolt — интерактивные уроки SQL | [SQLBolt](https://sqlbolt.com/) — бесплатный, 18 уроков с упражнениями в браузере, JOIN и агрегация |
| SQL Academy — интерактивный курс | [SQL Academy](https://sql-academy.org/) — 70+ задач, от SELECT до JOIN и подзапросов, есть русская версия |
| SQL-EX — упражнения по SQL | [sql-ex.ru](https://sql-ex.ru/) — упражнения от SELECT до DML, рейтинг, PostgreSQL среди СУБД |
| Karpov Courses — Симулятор SQL | [Karpov Courses](https://karpov.courses/simulator-sql) — бесплатный тренажёр, JOIN, GROUP BY, оконные функции |
| LeetCode SQL 50 | [LeetCode](https://leetcode.com/studyplan/top-sql-50/) — 50 задач: SELECT, JOIN, GROUP BY, подзапросы |

### Compose + PostgreSQL: локальная среда из двух сервисов

| Тема | Источник |
|---|---|
| Compose — обзор и старт | [Docker Compose overview](https://docs.docker.com/compose/) + [Quickstart](https://docs.docker.com/get-started/docker-concepts/running-containers/multi-container-applications/) |
| Volumes — полный справочник | [Docker volumes](https://docs.docker.com/engine/storage/volumes/) |
| Workshop: named volume на примере БД | [Persist the DB](https://docs.docker.com/get-started/workshop/05_persisting_data/) — мостик от модуля 3 к PostgreSQL |
| Compose + PostgreSQL (официальный пример) | [PostgreSQL + Docker Compose](https://hub.docker.com/_/postgres) — раздел «How to use this image» с compose-примером |
| Compose — русский практический гайд | [Docker для новичков — Compose / Хабр](https://habr.com/ru/articles/804331/) |
| Compose — основы | [Основы Docker Compose / Skillbox Media](https://skillbox.ru/media/code/docker-compose/) |

### PostgreSQL: основы SQL и работа с БД

| Тема | Источник |
|---|---|
| PostgreSQL Tutorial | [PostgresPro: Учебное руководство](https://postgrespro.ru/docs/postgresql/current/tutorial) |
| PostgreSQL Exercises — практика SQL | [PostgreSQL Exercises](https://pgexercises.com/) |
| Metanit: PostgreSQL | [Metanit: PostgreSQL](https://metanit.com/sql/postgresql/) |
| `psql` — интерактивный клиент | [psql reference](https://www.postgresql.org/docs/current/app-psql.html) |
| IntelliJ Database tool | [JetBrains: Database management](https://www.jetbrains.com/help/idea/database-tool-window.html) |
| pgAdmin (рекомендуется в Docker Compose) | [pgAdmin Docker Image](https://hub.docker.com/r/dpage/pgadmin4/) — добавь как третий сервис в `docker compose`; [pgAdmin Docs](https://www.pgadmin.org/docs/) |
| DBeaver (дополнительный вариант) | [DBeaver](https://dbeaver.io/) — универсальный десктопный клиент, работает без IDE и Docker |

### SQL: агрегация и объединение (GROUP BY, HAVING, JOIN)

| Тема | Источник |
|---|---|
| GROUP BY и агрегатные функции | [PostgresPro: Агрегатные функции](https://postgrespro.ru/docs/postgresql/current/tutorial-agg) · [Metanit: Группировка](https://metanit.com/sql/postgresql/4.6.php) · [devKazakov: GROUP BY, HAVING](https://devkazakov.com/ru/blog/grupirovka-v-sql-group-by-having-agregatnye-funkcii/) |
| HAVING — фильтрация групп | [Microsoft Learn: Агрегатные функции и группирование](https://learn.microsoft.com/ru-ru/training/azure-databases/postgresql/basic-sql-aggregate-functions-grouping/) · [PostgresPro: Агрегатные функции](https://postgrespro.ru/docs/postgresql/current/tutorial-agg) |
| JOIN: INNER, LEFT, RIGHT, FULL | [PostgresPro: Соединение таблиц](https://postgrespro.ru/docs/postgresql/current/tutorial-join) · [Metanit: Соединение таблиц](https://metanit.com/sql/postgresql/6.2.php) |
| SQLBolt — JOIN (уроки 6–12) | [SQLBolt: Multi-table queries with JOINs](https://sqlbolt.com/lesson/select_queries_with_joins) — интерактивные упражнения |
| SQLZoo — JOIN и агрегация | [SQLZoo: The JOIN operation](https://sqlzoo.net/wiki/The_JOIN_operation) · [More JOIN operations](https://sqlzoo.net/wiki/More_JOIN_operations) — упражнения в браузере |
| sql-practice.com — GROUP BY и JOIN | [GROUP BY exercises](https://www.sql-practice.com/learn/query/group_by/) · [JOIN exercises](https://www.sql-practice.com/learn/query/join/) |
| HackerRank SQL — Aggregation и Join | [HackerRank SQL](https://www.hackerrank.com/domains/sql) — разделы Aggregation, Basic Join, Advanced Join |
| LearnSQL — GROUP BY practice | [10 GROUP BY Exercises](https://learnsql.com/blog/group-by-exercises/) — интерактивные задачи с решениями |

### JDBC: доступ к БД из Java

| Тема | Источник |
|---|---|
| JDBC Basics | [Oracle: JDBC Basics](https://docs.oracle.com/javase/tutorial/jdbc/basics/index.html) |
| PostgreSQL JDBC driver (pgJDBC) | [pgJDBC Documentation](https://jdbc.postgresql.org/documentation/) |
| Metanit: JDBC | [Metanit: JDBC](https://metanit.com/java/database/1.1.php) |
| `PreparedStatement` и SQL Injection | [Oracle: Prepared Statements](https://docs.oracle.com/javase/tutorial/jdbc/basics/prepared.html) · [Baeldung: SQL Injection и Java](https://www.baeldung.com/sql-injection) |

### Liquibase: версионирование схемы БД

| Тема | Источник |
|---|---|
| Liquibase Documentation | [Liquibase Docs](https://docs.liquibase.com/) |
| Liquibase + Gradle | [Liquibase Gradle Plugin](https://github.com/liquibase/liquibase-gradle-plugin) |
| Liquibase University (бесплатный курс) | [Liquibase Learn](https://learn.liquibase.com/) |
| Сравнение migration tools | [Сравнение Flyway и Liquibase / Хабр](https://habr.com/ru/companies/otus/articles/760088/) — контекст выбора |

### Testcontainers: интеграционные тесты с реальной БД

| Тема | Источник |
|---|---|
| Testcontainers for Java | [Testcontainers Java Docs](https://java.testcontainers.org/) |
| PostgreSQL module | [Testcontainers PostgreSQL module](https://java.testcontainers.org/modules/databases/postgres/) |
| Testcontainers Guides | [Testcontainers Guides](https://testcontainers.com/guides/) |
| Testcontainers на Habr | [Testcontainers / Хабр](https://habr.com/ru/search/?q=Testcontainers&target_type=posts&order=relevance) |

### ACID и транзакции

| Тема | Источник |
|---|---|
| ACID: что под капотом у транзакции | [Хабр (SimbirSoft)](https://habr.com/ru/company/simbirsoft/blog/572540/) — теория + изоляция + уровни |
| ACID и транзакции PostgreSQL с примерами | [Хабр](https://habr.com/ru/articles/843794/) — практические примеры в `psql`, dirty read / phantom read |
| JDBC-транзакции: от `setAutoCommit(false)` до Spring `@Transactional` | [Struchkov.dev](https://struchkov.dev/blog/ru/transaction-jdbc-and-spring-boot/) — мостик к модулю 5 |
| Уровни изоляции в PostgreSQL | [PostgresPro: Уровни изоляции транзакций](https://postgrespro.ru/docs/postgresql/current/transaction-iso) |
| Row-level locks: `SELECT FOR UPDATE` | [PostgresPro: Блокировки на уровне строк](https://postgrespro.ru/docs/postgresql/current/explicit-locking#LOCKING-ROWS) |
| `SELECT FOR UPDATE SKIP LOCKED` | [PostgresPro: SKIP LOCKED](https://postgrespro.ru/docs/postgresql/current/explicit-locking#LOCKING-ROWS) — мостик к модулю 6 (Outbox polling) |

### Индексы и EXPLAIN

| Тема | Источник |
|---|---|
| Типы индексов PostgreSQL: обзор | [PostgresPro: Типы индексов](https://postgrespro.ru/docs/postgresql/current/indexes-types) · [Tour of Postgres Index Types / Citus Data](https://www.citusdata.com/blog/2017/10/17/tour-of-postgres-index-types/) |
| B-tree | [PostgresPro: B-tree](https://postgrespro.ru/docs/postgresql/current/btree) |
| GIN | [PostgresPro: GIN](https://postgrespro.ru/docs/postgresql/current/gin) |
| GiST | [PostgresPro: GiST](https://postgrespro.ru/docs/postgresql/current/gist) |
| BRIN | [PostgresPro: BRIN](https://postgrespro.ru/docs/postgresql/current/brin) |
| Hash | [PostgresPro: Hash](https://postgrespro.ru/docs/postgresql/current/indexes-types) |
| Составные индексы (multicolumn) | [PostgresPro: Многоколоночные индексы](https://postgrespro.ru/docs/postgresql/current/indexes-multicolumn) |
| Частичные индексы (partial, с WHERE) | [PostgresPro: Частичные индексы](https://postgrespro.ru/docs/postgresql/current/indexes-partial) |
| Expression indexes | [PostgresPro: Индексы по выражениям](https://postgrespro.ru/docs/postgresql/current/indexes-expressional) |
| Index-Only scans | [PostgresPro: Index-Only Scans](https://postgrespro.ru/docs/postgresql/current/indexes-index-only-scans) |
| Уникальные индексы | [PostgresPro: Уникальные индексы](https://postgrespro.ru/docs/postgresql/current/indexes-unique) |
| Operator classes | [PostgresPro: Классы операторов](https://postgrespro.ru/docs/postgresql/current/indexes-opclass) |
| EXPLAIN и EXPLAIN ANALYZE | [PostgresPro: Использование EXPLAIN](https://postgrespro.ru/docs/postgresql/current/using-explain) · [Как читать EXPLAIN в PostgreSQL / Хабр](https://habr.com/ru/companies/postgrespro/articles/713176/) |
| depesz: Explaining the Unexplainable | [Part 2](https://www.depesz.com/2013/04/27/explaining-the-unexplainable/) |
| Визуализация: explain.depesz.com | [explain.depesz.com](https://explain.depesz.com/) |
| Визуализация: explain.dalibo.com | [explain.dalibo.com](https://explain.dalibo.com) |
| Егор Рогов: Индексы в PostgreSQL (10 частей) | [Часть 1 / Хабр](https://habr.com/ru/companies/postgrespro/articles/326096/) · [Части 2–10](https://habr.com/ru/companies/postgrespro/articles/326300/) |
| The Internals of PostgreSQL | [interdb.jp/pg](https://www.interdb.jp/pg/) |

### AI-инструменты для работы с БД

Общие AI-инструменты — в [README → AI-инструменты](../README.MD#ai-инструменты-и-плагины).

| Тема | Источник |
|---|---|
| Postgres MCP Pro — AI-агент для PostgreSQL | [crystaldba/postgres-mcp](https://github.com/crystaldba/postgres-mcp) — EXPLAIN-анализ, рекомендации по индексам, health checks |
| MCP Migration Advisor — проверка Liquibase-миграций | [dmitriusan/mcp-migration-advisor](https://github.com/dmitriusan/mcp-migration-advisor) — обнаружение конфликтов changeSet ID, проверка rollback |
| ChartDB — AI-powered ER-диаграммы | [chartdb.io](https://chartdb.io/) — визуализация схемы PostgreSQL, reverse engineering |
| Базовый MCP-плагин для PostgreSQL | [@modelcontextprotocol/server-postgres](https://github.com/modelcontextprotocol/servers/tree/main/src/postgres) — read-only доступ к PostgreSQL через MCP |
| CLAUDE.md — проектные инструкции | Файл с конвенциями проекта: стек, структура БД, антипаттерны, стиль — AI учитывает контекст при работе со схемой |

**Принцип:** AI помогает с EXPLAIN и схемой, результат верифицируется через benchmark или EXPLAIN ANALYZE.
