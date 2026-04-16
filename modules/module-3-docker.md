# Модуль 3: Docker

## Содержание

- [Для кого этот модуль](#для-кого-этот-модуль)
- [Зачем этот модуль](#зачем-этот-модуль)
- [Что меняется по сравнению с модулем 2](#что-меняется-по-сравнению-с-модулем-2)
- [FR и NFR в этом модуле](#fr-и-nfr-в-этом-модуле)
- [Технологический стек и инструменты](#технологический-стек-и-инструменты)
- [Что строим / как проект эволюционирует](#что-строим--как-проект-эволюционирует)
- [Перед стартом](#перед-стартом)
- [Как проверить результат](#как-проверить-результат)
- [Публичные ресурсы](#публичные-ресурсы)

---

### Для кого этот модуль

**Ты прошёл модули 1-2** и продолжаешь развивать проект — не создаёшь новый, а контейнеризуешь существующее консольное приложение.

**Или ты не проходил модули 1-2**, потому что у тебя уже есть работающий Java-проект с Gradle-сборкой и тестами. В таком случае стартуй сразу отсюда — тебе понадобится JAR-артефакт, который можно упаковать в Docker-образ.

---

## Зачем этот модуль

В модулях 1-2 проект стал инженерно взрослее: осмысленная структура, тесты, проверки качества, более взрослая объектная модель. Но он всё ещё зависит от машины разработчика: версии Java, путей, настроек IDE.

Модуль 3 — первый модуль, где **функциональность проекта не меняется**. Приложение делает то же самое: создание мероприятия, регистрация, отмена, лист ожидания, отмена последнего действия. Но меняется **как оно запускается и доставляется** — это сфера нефункциональных требований. Тема критически важная, и лучше поработать с ней руками до того, как в модуле 4 добавится база данных и функциональность начнёт расти снова.

Проект становится **контейнеризуемым** и **операционно воспроизводимым** — одинаково собирается и запускается через Docker CLI, а не только через кнопку в IntelliJ.

Проект должен запускаться воспроизводимо — не только у разработчика, но и у пользователя. Docker — способ гарантировать это. Контейнеризация превращает «у меня работает» из отговорки в технический артефакт: образ собирается одной командой, запускается на любой машине, логи читаются стандартными инструментами.

Заодно проект переходит на **структурированное логирование** (SLF4J + Logback). Пока приложение запускалось из IDE, `System.out.println` было достаточно. В контейнере breakpoint не всегда поставишь. Remoute Debug пока не рассматриваем. Нужно уметь читать логи с уровнями и таймстемпами. Этот навык пригодится в каждом следующем модуле, а в модуле 4 (БД) станет критичным: диагностика SQL-запросов, миграций и проблем с соединениями без нормального логирования — практически невозможна.

Общая рамка по FR/NFR, роли ИИ и логика маршрута — в [корневом README](../README.MD).

---

## Что меняется по сравнению с модулем 2

| Аспект | Модуль 2 | Модуль 3 |
|--------|----------|----------|
| Запуск | `./gradlew run` или IDE | `docker run` / `docker compose up` |
| Диагностика | Консоль IDE | `docker logs`, `docker exec`, `docker inspect` |
| Окружение | Зависит от локальной Java | Не зависит от локальной Java |
| Данные | В памяти (или CSV-файлы ★) | Named volume для постоянных данных |
| Логирование | `System.out.println` | SLF4J + Logback, видимые через `docker logs` |
| Артефакт | JAR локально | Docker-образ в GHCR |

---

## FR и NFR в этом модуле

> Общая рамка: [README → Как работать с требованиями](../README.MD#как-работать-с-требованиями).

### Функциональные требования

**Новых FR нет.** Все функциональные требования наследуются из модулей 1-2 без изменений: создание мероприятия, регистрация, отмена, лист ожидания, отмена последнего действия — всё работает так же, но теперь внутри контейнера.

> Это первый модуль, где проект делает то же самое, но по-другому доставляется. Все требования ниже — нефункциональные.

### Нефункциональные требования

**Контейнеризация и доставка:**

- `docker build` собирает образ одной командой через собственный `Dockerfile`
- `docker compose config` валидирует конфигурацию
- Приложение запускается через Docker CLI (`docker run`, `docker compose up`)
- Образ опубликован в GHCR и доступен для скачивания

**Персистентность (named volume):**

- Named volume переживает `docker compose down` + `docker compose up`
- Если в модуле 2 было выполнено бонусное задание (★) по сохранению данных в CSV-файлы — named volume монтируется именно к директории с этими файлами, чтобы данные переживали пересоздание контейнера
- Если CSV-хранение не реализовано — volume добавляется как подготовка к модулю 4 (PostgreSQL): структура `docker-compose.yaml` уже будет готова к подключению базы

**Логирование (SLF4J + Logback):**

- Приложение логирует через SLF4J logger, а не через `System.out.println`
- Логи видны через `docker logs` и содержат уровень (INFO, WARN, ERROR)
- Конфигурация Logback вынесена в `logback.xml`

**AI SDLC:**

- AI-предложения по Docker-командам верифицируются через запуск в CLI
- Каждый слой Dockerfile объясним

> Logback вводится здесь как обязательный инженерный навык: в модуле 4 он понадобится для диагностики SQL-запросов, миграций и проблем с соединениями.

---

## Технологический стек и инструменты

| Категория | Что используем | Минимум для Done |
|-----------|----------------|------------------|
| Контейнерная платформа | Docker Desktop (или Docker Engine) | `docker version` работает |
| CLI | `docker` + `docker compose` plugin | Базовые команды выполняются руками |
| Java-артефакт | JAR из модулей 1-2 | Артефакт можно упаковать в образ |
| Конфигурация образа | `Dockerfile` + `.dockerignore` | Образ собирается и запускается |
| Оркестрация | `docker-compose.yaml` | Один сервис через `docker compose up` |
| Постоянные данные | Docker named volume | Volume виден через CLI |
| Registry | GitHub Container Registry (GHCR) | Образ запушен и доступен |
| Логирование | SLF4J + Logback | Приложение логирует через logger, а не `System.out` |
| AI | DeepSeek / Qwen Chat, Qwen Code CLI | Диагностика docker-логов через AI, верификация AI-команд через запуск |
| IDE | IntelliJ IDEA (или другая) | Подключается к Docker, но не заменяет CLI |

---

## Что строим / как проект эволюционирует

**Стартовая точка:** консольное Java-приложение из модуля 2, собирается через Gradle, запускается локально.

**Почему именно сейчас:** контейнеризация до добавления базы данных — это осознанный выбор. В модуле 4 compose-стек расширится сервисом PostgreSQL, и к этому моменту Docker CLI уже должен быть уверенным инструментом. В модуле 6 compose станет локальной инфраструктурой для нескольких микросервисов. Первый `Dockerfile` и первый `docker compose` здесь — фундамент для всей дальнейшей эволюции проекта.

**Что происходит в этом модуле:**

- Проект получает собственный `Dockerfile` и `.dockerignore`
- Локальный запуск перестаёт зависеть от локальной установки Java
- Приложение переходит на структурированное логирование через SLF4J + Logback — `System.out.println` заменяется на logger с уровнями
- Появляется первый `docker compose` с named volume — мостик к модулю 4
- Образ публикуется в GHCR

**К концу модуля:**

- Собираешь и запускаешь Java-приложение в контейнере
- Читаешь структурированные логи через `docker logs` — с уровнями, а не сплошным потоком `System.out`
- Заходишь внутрь контейнера для диагностики через `docker exec`
- Поднимаешь сервис через `docker compose`
- Видишь и проверяешь named volume
- Образ доступен в GHCR

**Финальный ориентир:**

Рабочий процесс выглядит так: собрать образ → запустить контейнер → поднять compose-стек с volume → опубликовать образ в GHCR. Все логи приложения видны через `docker logs` и содержат уровни Logback.

---

## Перед стартом

- Модуль 2 завершён
- Проект собирается через `./gradlew build`
- Понимаешь, что такое JAR и где лежит собранный артефакт
- SLF4J + Logback добавлены в зависимости Gradle (если не добавлены — это одна из первых задач модуля)
- Docker установлен и доступен из терминала
- Есть GitHub-аккаунт с правом записи в Packages (для GHCR-пуша понадобится Personal Access Token со scope `write:packages`)

---

## Как проверить результат

Модуль можно считать пройденным, если:

- `docker build -t party-app:local .` собирает образ
- `docker run --rm party-app:local` запускает приложение в контейнере
- `docker compose config` валидирует конфигурацию
- `docker compose up` поднимает сервис
- `docker volume ls` показывает named volume
- `docker logs` показывает структурированные логи с уровнями (INFO, WARN, ERROR), а не `System.out`-поток
- `docker push ghcr.io/<user>/party-app:latest` пушит образ
- `docker pull ghcr.io/<user>/party-app:latest` скачивает образ на другой машине

---

## Публичные ресурсы

> Если используешь ИИ для подсказок — проси его объяснять через аналогию с Java-сборкой: образ — это JAR, контейнер — это запущенный процесс, volume — это файл на диске, который переживает рестарт.

### Установка и первые команды

| Тема | Источник |
|---|---|
| Установка Docker Desktop | [Install Docker Desktop on Windows](https://docs.docker.com/desktop/setup/install/windows-install/) |
| Docker Desktop + WSL2 | [WSL 2 backend](https://docs.docker.com/desktop/features/wsl/) — если на Windows; [Microsoft Learn: Docker on WSL 2](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-containers) |
| Первые команды и шпаргалка | [Get Started](https://docs.docker.com/get-started/introduction/) + [Docker Cheatsheet (PDF)](https://docs.docker.com/get-started/docker_cheatsheet.pdf) |
| CLI-справочник | [Docker CLI reference](https://docs.docker.com/reference/cli/docker/) |

### Dockerfile + Java

| Тема | Источник |
|---|---|
| Контейнеризация Java | [Docker Java guide](https://docs.docker.com/guides/java/) + [Containerize a Java app](https://docs.docker.com/guides/java/containerize/) |
| Dockerfile — инструкции | [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) |
| Dockerfile — концепции | [Writing a Dockerfile](https://docs.docker.com/get-started/docker-concepts/building-images/writing-a-dockerfile/) |
| Java + Docker (Baeldung) | [Dockerizing a Java Application](https://www.baeldung.com/java-dockerize-app) |
| Docker + Java (Sematext) | [Docker Java Tutorial](https://sematext.com/blog/docker-java-tutorial/) |
| Multi-stage builds | [Multi-stage builds](https://docs.docker.com/get-started/docker-concepts/building-images/multi-stage-builds/) — для уменьшения размера образа |

### Docker Compose

| Тема | Источник |
|---|---|
| Compose — обзор и старт | [Docker Compose overview](https://docs.docker.com/compose/) + [Quickstart](https://docs.docker.com/compose/gettingstarted/) |
| Compose file reference | [compose.yaml reference](https://docs.docker.com/reference/compose-file/) |

### Volumes

| Тема | Источник |
|---|---|
| Named volumes — практика | [Persisting container data](https://docs.docker.com/get-started/docker-concepts/running-containers/persisting-container-data/) |
| Volumes — полный справочник | [Docker volumes](https://docs.docker.com/engine/storage/volumes/) |
| Workshop: named volume на примере БД | [Persist the DB](https://docs.docker.com/get-started/workshop/05_persisting_data/) — мостик к модулю 4 |

### GitHub Container Registry

| Тема | Источник |
|---|---|
| GHCR — официальный гайд | [Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) |
| Push через GitHub Actions | [Publishing Docker images](https://docs.github.com/en/actions/publishing-packages/publishing-docker-images) |

### Логирование (SLF4J + Logback)

| Тема | Источник |
|---|---|
| SLF4J + Logback — обзор | [SLF4J official manual](https://www.slf4j.org/manual.html) + [Logback architecture](https://logback.qos.ch/manual/architecture.html) |
| Logback configuration | [Logback configuration](https://logback.qos.ch/manual/configuration.html) |
| Логирование в Java (Baeldung) | [Introduction to SLF4J](https://www.baeldung.com/slf4j-with-log4j2-logback) |
| Логирование в Java (Хабр) | [Логирование в Java: что, зачем и как](https://habr.com/ru/articles/734242/) |

### Русскоязычные источники

| Тема | Источник |
|---|---|
| Docker для начинающих — обзор | [Docker для начинающих / Хабр (Netology)](https://habr.com/ru/companies/netologyru/articles/967546/) |
| Dockerfile — все инструкции | [Docker для новичков — Dockerfile / Хабр](https://habr.com/ru/articles/804325/) |
| Compose — практический гайд | [Docker для новичков — Compose / Хабр](https://habr.com/ru/articles/804331/) |
| Оптимизация Dockerfile | [Docker для новичков — Оптимизация / Хабр](https://habr.com/ru/articles/804333/) |
| Docker Compose для начинающих | [Руководство по Docker Compose / Хабр (ruvds)](https://habr.com/ru/companies/ruvds/articles/450312/) |
| Docker Compose — основы | [Основы Docker Compose / Skillbox Media](https://skillbox.ru/media/code/docker-compose/) |

### AI-инструменты

Общие AI-инструменты — в [README → AI-инструменты](../README.MD#ai-инструменты-и-плагины).

**Роль AI в этом модуле:** навигатор по Docker-концепциям и анализатор конфигурации (Dockerfile, compose.yaml, docker-логи).

**Принцип:** AI-предложения по Docker-командам проверяются через запуск. AI помогает понять, почему контейнер падает, но не заменяет ручную верификацию.

### YouTube

| Тема | Источник |
|---|---|
| Docker для Начинающих за 49 мин (2025) | [YouTube](https://www.youtube.com/watch?v=0oke9VfRt2M) |
| Docker — Полный курс (Stashchuk, 3 ч) | [YouTube](https://www.youtube.com/watch?v=_uZQtRyF6Eg) |
| Docker Tutorial for Beginners (TechWorld with Nana, плейлист ~4 ч) | [YouTube playlist](https://www.youtube.com/playlist?list=PLy7NrYWoggjzfAHlUusx2wuDwfCrmJYcs) |
| Docker Tutorial 2025 — Hands-On Labs (~2 ч) | [YouTube](https://www.youtube.com/watch?v=fTF-5lQmXHY) |

### Stepik — бесплатные курсы

| Курс | Ссылка |
|---|---|
| Знакомство с Docker (2-4 ч, 14 уроков) | [Stepik](https://stepik.org/course/253174/promo) |
| Docker для начинающих + практический опыт | [Stepik](https://stepik.org/course/123300/promo) |
| Docker (devupsen) — минимум за минимальное время | [Stepik](https://stepik.org/course/120182/promo) |

---

