# Roadmap Даниила Емельянова: Путь к Middle Java-разработчику за 4–6 месяцев
## Version 1.0

**Занятость:** 15–20 часов в неделю  
[ROADMAP LINK](https://whimsical.com/java-roadmap-LWtzDp4HnE8hZ3u3VdEYMK)  
![JAVA_ROADMAP.png](static%2FJAVA_ROADMAP.png)
 ---

## Содержание
- [Roadmap Даниила Емельянова: Путь к Middle Java-разработчику за 4–6 месяцев](#roadmap-даниила-емельянова-путь-к-middle-java-разработчику-за-4–6-месяцев)
  - [Содержание](#содержание)
  - [Введение](#введение)
  - [Как читать роудмап](#как-читать-роудмап)
  - [Месяц 1: Infrastructure Base](#месяц-1-infrastructure-base)
  - [Месяц 1: Core](#месяц-1-core)
  - [Месяц 2: Advanced](#месяц-2-advanced)
  - [Месяц 3: Database](#месяц-3-database)
  - [Месяц 4: Enterprise](#месяц-4-enterprise)
  - [Месяц 5: CI/CD](#месяц-5-cicd)
  - [Месяц 5-6: Cloud, микросервисы и Job Interview](#месяц-5-6-cloud-микросервисы-и-job-interview)
  - [Заключение](#заключение)
  - [Контакты:](#контакты)

*(Если времени меньше четырёх месяцев, некоторые блоки можно учить параллельно.)*
 
---

## Введение
Данный роудмап показывает путь от базового Java-разработчика к уровню **Middle**, включая релевантный стек самых популярных на данный момент технологий для работы в большинстве современных компаний. Роудмап рассчитан как на полных новичков, так и на тех, кто уже имеет базовые знания в Java. В том числе — на разработчиков, которые в данный момент испытывают сложности с поиском работы. Разделы начиная с Enterprice помогут освоить необходимые харды по стеку с микросервисами. Раздел Карьерного трека — получить необходимые софты и отточить навык прохождения интервью. Карьерный трек можно начинать паралельно с изучением уровня Enterprice. По роудмапу планируются обновления как в части самой карты, так и описания блоков с сылками.

## Как читать роудмап
На схеме представлены блоки, которые нужно изучить. В каждом блоке — задания, которые нужно выполнить. Пунктирными линиями обозначены темы, которые можно изучать только теоретически, не практикуя их.  
Цифрами обозначены как рекомендуемый порядок, так и затраты в месяцах, начиная с первого блока.  
Ссылки на материалы для изучения, примеры домашних заданий представлены в каждом блоке. Все материалы на момент публикации бесплатные. Большинство материалов доступно на английском языке. Если есть сложности с английским, можно использовать браузер со встроенным переводчиком, а также AI в качестве помощника для разжёвывания материала.

**Схема включает**:

- **Infrastructure Base**: IDE, Git, Build Tools, AI, Agile/Scrum основы
- **Core**: базовый синтаксис, коллекции, ввод-вывод, исключения, TDD, Checkstyle
- **Advanced**: ООП, расширенные коллекции, Java Streams, сериализация, GC, JVM, паттерны, SOLID, многопоточность
- **Database**: SQL (DDL, DML, JOIN), ORM (JPA, Hibernate), Liquibase, NoSQL
- **Enterprise**: Spring (Boot, WebFlux, GraphQL, Security, Data JPA), AOP, Logging, Camunda BPMN, CI/CD (GitLab CI, Jenkins)
- **Cloud**: Docker, Docker Compose, Kubernetes, микросервисы (паттерны: circuit breaker, saga, 2 Phase Commit), Keycloak, Kafka, ELK, Spring Cloud (Config/Stream/Feign), ActiveMQ
- **Career track / Job Interview**: Советы по составлению резюме, разбор частых вопросов

**Рекомендуемые книги**:
- *Head First Java*, *Head First OOP*, *Head First SQL*, *Head First Servlets & JSP*  
  для начала и закрепления базовых знаний, а также больше практики с Java Core, OOP, базами данных, веб-приложениями.

И **доп. материалы** по Spring от Евгения Борисова (подкасты и серии «потрошитель»/«построитель»), чтобы глубже понять, как Spring устроен внутри и чем отличается Spring Core от Spring Boot.
 
---

## Месяц 1: Infrastructure Base

| **Блок**         | **Описание и цели**                                                                                                                                                           | **Ссылки и скачивание**                                                                                                                                                                                  |
 |------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **IDE**          | Попробовать запустить HelloWorld в онлайн тренажёре. Установить и освоить базовую среду разработки: IntelliJ IDEA (Community) / VSCode. Узнать основы дебага, автоподсказок.                                         | – [IntelliJ IDEA Community Download](https://www.jetbrains.com/idea/download/) <br> – [VSCode (Java Ext)](https://code.visualstudio.com/docs/languages/java) <br> – [Online IDE w3schools](https://www.w3schools.com/java/java_compiler.asp) |
| **Git**          | Система контроля версий, GitHub/GitLab, ветки, мерджи, пул-реквесты. Цель — хранить код и работать в команде.                                                                                                         | – [Git (Official)](https://git-scm.com/) <br> – [Pro Git Book](https://git-scm.com/book/en/v2)                                                                                                                                               |
| **Build Tools**  | Maven, Gradle (Ant — реже). Настроить сборку/зависимости, запуск проектов.                                                                                                                                           | – [Maven (Official)](https://maven.apache.org/) <br> – [Gradle (Official)](https://gradle.org/)                                                                                                                                                |
| **AI Tools**     | ChatGPT, Cursor AI, GitHub Copilot — помощники при написании кода, рефакторинге, генерировании тестов.                                                                                                               | – [Cursor AI](https://www.cursor.so/) <br> – [GitHub Copilot](https://github.com/features/copilot) <br> – [ChatGPT](https://chat.openai.com/)                                                                                                 |
| **Agile (Scrum)**| Базовые понятия: спринты, бэклог, командная работа. Разобраться, как ведётся итеративная разработка в реальных проектах.                                                                                             | – [Scrum Guide](https://scrumguides.org/)                                                                                                                                                                                                    |

**Примеры домашних заданий**:
1. Инициализировать локальный Git-репозиторий, запушить на GitHub, оформить pull request.
2. Создать «Hello, World!» на Maven/Gradle, собрать проект, проверить логи.
3. Изучить горячие клавиши и дебаг в IntelliJ или VsCode.
4. (Для практики Agile) Попробовать разбить задание на несколько «задач-историй» и решить их итеративно.

 ---

## Месяц 1: Core

| **Блок**                 | **Описание и цели**                                                                                                                                                                       | **Документация**                                                                                                                                                                  |
 |--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Basic Syntax**         | Переменные, операторы, типы данных, простые конструкции (if/else, switch). Цель — уметь писать базовый код на Java.                                                                                                               | – [Oracle Java Tutorial (Basics)](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/) <br> – *Head First Java* (для лёгкого старта с примерами кода)                                                                   |
| **Data Types, Variables**| Примитивы (int, boolean, etc.) и ссылки (String, массивы), область видимости                                                                                                                                                     | – [Data Types (Oracle)](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html)                                                                                                                              |
| **Conditionals, Loops**  | Управляющие конструкции: `for`, `while`, `do-while`, `switch`, `if/else`                                                                                                                                                         | – [Java Flow Control](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/flow.html)                                                                                                                                     |
| **Functions & Methods**  | Объявление, параметры, возвращаемые значения, перегрузка                                                                                                                                                                        | – [Java Methods (W3Schools)](https://www.w3schools.com/java/java_methods.asp)                                                                                                                                                  |
| **Exception Handling**   | try/catch/throw, checked/unchecked исключения                                                                                                                                                                                   | – [Java Exceptions (Oracle)](https://docs.oracle.com/javase/tutorial/essential/exceptions/)                                                                                                                                    |
| **Collections Base**     | ArrayList, LinkedList (минимум). Понимание, как хранить наборы данных.                                                                                                                                                          | – [Collections (Oracle)](https://docs.oracle.com/javase/tutorial/collections/)                                                                                                                                                 |
| **I/O Streams**          | Ввод-вывод (FileInputStream, FileOutputStream, BufferedReader). Уметь читать/записывать файлы.                                                                                                                                  | – [Java I/O (Baeldung)](https://www.baeldung.com/java-io)                                                                                                                                                                      |
| **TDD (база)**           | Начальная практика тестирования: JUnit, написание простых тестов.                                                                                                                                                               | – [JUnit 5 Docs](https://junit.org/junit5/docs/current/user-guide/)                                                                                                                                                            |
| **Checkstyle**           | Инструмент статического анализа кода, проверка на стиль/ошибки.                                                                                                                                                                 | – [Checkstyle (GitHub)](https://github.com/checkstyle/checkstyle)                                                                                                                                                             |

**Примеры домашних заданий**:
1. **Консольные игры** (выбрать 1–2): «Угадай число», «Крестики-нолики», «Виселица» (Hangman).
2. Применить TDD на паре методов (например, простая калькуляция).
3. Считывать/записывать данные из файла (тесты на проверку корректного чтения).
4. Применить **Checkstyle** к проекту, исправить замечания по стилю.

 ---

## Месяц 2: Advanced

| **Блок**                    | **Описание и цели**                                                                                                                             | **Документация**                                                                                                                                                                                                           |
 |-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **OOP**                     | Классы, объекты, инкапсуляция, наследование, интерфейсы, полиморфизм. Уметь структурировать код, создавать более сложные объекты.               | – [Java OOP (Oracle)](https://docs.oracle.com/javase/tutorial/java/IandI/) <br> – *Head First OOP* (доп. книга с примерами и объяснениями ООП)                                                                             |
| **Collection Advanced**     | HashMap, Set, Queue, PriorityQueue и т.д. Понимать сложности операций, внутренние структуры.                                                    | – [Collections (Oracle)](https://docs.oracle.com/javase/tutorial/collections/)                                                                                                                                             |
| **Java Streams**            | Stream API (map/filter/reduce), лямбда-выражения, функциональный стиль.                                                                         | – [Java Streams (Baeldung)](https://www.baeldung.com/java-8-streams-introduction)                                                                                                                                          |
| **Serialization**           | Serializable, JSON (Jackson/Gson). Понимать, как сохранять/восстанавливать объекты.                                                             | – [Serialization (Oracle)](https://docs.oracle.com/javase/8/docs/platform/serialization/)                                                                                                                                  |
| **GC** (Garbage Collection) | Алгоритмы GC (Serial, Parallel, G1, ZGC). Понимать работу сборки мусора.                                                                        | – [GC Tuning (Oracle)](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/)                                                                                                                                |
| **JVM**                     | Байткод, ClassLoader, JIT-компиляция, понимание, как работает JVM.                                                                              | – [Inside the JVM (Baeldung)](https://www.baeldung.com/jvm)                                                                                                                                                                |
| **Design Patterns**         | Singleton, Factory, Strategy, Observer, Builder. Уметь применять при решении типовых задач.                                                     | – [GoF Patterns (Refactoring.Guru)](https://refactoring.guru/design-patterns)                                                                                                                                              |
| **SOLID**                   | Знать принципы: Single Responsibility, Open-Closed, Liskov, Interface Segregation, Dependency Inversion. Уметь рассказывать, когда применяются. | – [SOLID (Baeldung)](https://www.baeldung.com/solid-principles)   </br> – [Bob Martin: Clean Code](https://www.ozon.ru/product/chistyy-kod-sozdanie-analiz-i-refaktoring-biblioteka-programmista-martin-robert-211433291/) |
| **MultiThreading Base**     | Thread, Runnable, ExecutorService, синхронизация (synchronized, Lock). Базовые навыки написания многопоточного кода.  Completable Future.       | – [Java Concurrency (Oracle)](https://docs.oracle.com/javase/tutorial/essential/concurrency/) </br> - [Guide To CompletableFuture](https://www.baeldung.com/java-completablefuture)   </br>[Java Concurrency in Practice](https://jcip.net/)                            |

**Примеры домашних заданий**:
1. **Простая практика многопоточности** (необязательный проект):
  - Создать несколько потоков, использующих один общий ресурс (список/очередь), и продумать синхронизацию.
  - Можно ограничиться короткими примерами, например запуск нескольких потоков, которые пишут/читают данные из списка.
2. **Задачи на Streams**: сортировка списка объектов, группировка, подсчёт (использовать фильтры, маппинг).
3. **Реализовать 2–3 паттерна** (Decorator, Singleton, Factory, Strategy) в мини-проекте:
«магазин» с разными способами оплаты (Strategy) + одиночный конфиг (Singleton) для хранения констант магазина из файла `app.properties`.
4. **(Для SOLID)** Показать, как можно нарушить или соблюдать принципы в коде (пример мини-приложения, где сначала пишем «нечисто», а затем рефакторим).

> **Примечание**: Многопоточность может усложнить код начинающим, поэтому достаточно ознакомиться с основными принципами (`Thread`, `Runnable`, `synchronized`), без сложных проектов. По желанию — углубиться (CompletableFuture, ExecutorService) как опциональная часть. Для интервью будет достоточно рассказать про CompletableFuture и проблемы дедлоки, гонки потоков.





 ---

## Месяц 3: Database

| **Блок**                  | **Описание и цели**                                                                                                                                   | **Документация**                                                                                                                                                              |
 |---------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Persistence (JDBC / ORM)** | Уметь сохранять данные из приложения в базу: JDBC для низкоуровневого доступа, ORM (JPA, Hibernate) для маппинга объектов.                                                               | – [Hibernate Docs](https://hibernate.org/orm/documentation/) <br> – [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)                                                               |
| **Relational Database**   | SQL (DDL, DML, JOIN & Constraints, Transactions & ACID). PostgreSQL/MySQL.                                                                                                                  | – [SQL (W3Schools)](https://www.w3schools.com/sql/) <br> – [PostgreSQL Docs](https://www.postgresql.org/docs/) <br> – *Head First SQL* (для освоения базового SQL в наглядной форме)                                      |
| **Checkstyle / Code Quality** | Продолжать держать проект в чистоте, проверять код на стиль, формат, простые ошибки.                                                                                                    | – [Checkstyle (GitHub)](https://github.com/checkstyle/checkstyle)                                                                                                                                                         |
| **DML / DDL Basics** (он же Data Manipulation / Definition)| Создание таблиц, заполнение данными, SELECT, UPDATE, DELETE, ALTER, DROP.                                                                                                             | – [Head First SQL](https://docs.google.com/file/d/0BzjGbQIp2abGcW1sYjZvUG9GZmc/edit?pli=1&resourcekey=0-uZYQar6EgXfOUqGL0ZnX9A) (наглядное объяснение)                                                                                                            |
| **JOIN & Constraints**    | INNER, LEFT, RIGHT, FOREIGN KEY, PRIMARY KEY, UNIQUE.                                                                                                                                         | – [SQL JOIN (W3Schools)](https://www.w3schools.com/sql/sql_join.asp)                                                                                                                                                      |
| **Transactions & ACID**   | COMMIT, ROLLBACK, уровни изоляции.                                                                                                                                                            | – [SQL Transactions (Oracle)](https://docs.oracle.com/cd/B19306_01/server.102/b14220/transact.htm)                                                                                                                        |
| **Migration (Liquibase)** | Инструмент для версионирования схемы (Liquibase, Flyway). Удобно для командной разработки.                                                                                                   | – [Liquibase (Official)](https://www.liquibase.org/) <br> – [Flyway (Official)](https://flywaydb.org/)                                                                                                                    |
| **NoSQL** (MongoDB)       | Документо-ориентированные БД (MongoDB), ключ-значение (Redis), по желанию.                                                                                                                   | – [MongoDB Docs](https://docs.mongodb.com/)                                                                                                                                                                              |

**Примеры домашних заданий**:
1. **SQL-задачи**: создать пару таблиц (User, Order), сделать SELECT, UPDATE, DELETE, JOIN.
2. **ORM**: записать и прочитать сущности (Hibernate). Связать 1–M, M–M.
3. **Liquibase**: написать пару миграций (CREATE TABLE, ALTER TABLE).
4. (Дополнительно) написать **Telegram-бот**, который хранит данные о пользователях/заказах в PostgreSQL (JDBC или JPA).
5. (Дополнительно) **Расчёт показателей** для «онлайн-биржи»: хранить заявки на покупку/продажу, делать SQL-запрос для расчёта объёма сделок.

 ---

## Месяц 4: Enterprise

| **Блок**                              | **Описание и цели**                                                                                                                                                                   | **Документация**                                                                                                                                                                                    |
 |---------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Spring Context**     | Механизм управления жизненным циклом бинов в Spring. Понимание того, как работает `ApplicationContext`, какие есть виды контекстов (например, `AnnotationConfigApplicationContext`). Умение пользоваться `@Configuration`, `@Bean`, `@ComponentScan`. | – [Spring Context (Official)](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-introduction) <br> – [AnnotationConfigApplicationContext](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-definition-class) |
| **Spring DI (Dependency Injection)** | Понимание принципов Inversion of Control (IoC), создание и связывание бинов, аннотации `@Component`, `@Autowired`, `@Configuration`. Отдельно разобраться, чем `@Bean` отличается от `@Component`. | – [Spring Framework (Official)](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html) <br> – [Spring IoC Container & DI](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#spring-core) |
| **Spring** (Boot, Web, WebFlux, GraphQL, AOP, Context) | Контейнер IoC, автоконфигурация, REST-контроллеры, реактивный стек (WebFlux), Spring GraphQL (опционально), AOP, жизненный цикл бинов. Понять, чем отличается Spring Core от Spring Boot и как они дополняют друг друга.       | – [Spring Framework (Official)](https://docs.spring.io/spring-framework/docs/current/reference/html/) <br> – [Spring Boot (Official)](https://docs.spring.io/spring-boot/docs/current/reference/html/) <br> – [Евгений Борисов: «Spring Потрошитель», «Spring Построитель»](https://www.youtube.com/watch?v=BmBr5diz8WA) (плейлисты, подкасты)     |
| **Spring Security**                   | Аутентификация, авторизация (Basic, JWT, Keycloak). Роли/права, тестирование безопасности.                                                                                                                                    | – [Spring Security Reference](https://docs.spring.io/spring-security/site/docs/current/reference/html5/)  |
| **Spring Data JPA**                  | Репозитории, аннотации (например, `@Repository`), интеграция с Hibernate.                                                                                                                                                     | – [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)                                                                                                                                   |
| **Spring Test**                       | Тестирование Spring-приложений (MockMvc, @SpringBootTest).                                                                                                                                                                    | – [Testing in Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)                                                                                                  |
| **Logging** (SLF4J, Logback)          | Логгирование, формат логов, уровни (INFO, WARN, ERROR).                                                                                                                                                                       | – [SLF4J (Official)](http://www.slf4j.org/) <br> – [Logback (Official)](http://logback.qos.ch/)                                                                                                                            |
| **Camunda BPMN**                      | Моделирование бизнес-процессов (BPMN 2.0), интеграция со Spring, запуск workflow.                                                                                                                                             | – [Camunda Docs](https://docs.camunda.org/)                                                                                                                                                                                |
| **CI/CD** (GitLab CI, Jenkins)        | Инструменты для сборки/тестов/деплоя. Можно связать Spring-приложение, запускать тесты при каждом пуше.                                                                                                                        | – [GitLab CI Docs](https://docs.gitlab.com/ee/ci/) <br> – [Jenkins (Official)](https://www.jenkins.io/)                                                                                                                                         |

В Spring-фреймворке ключевым понятием является **IoC-контейнер** (Inversion of Control Container). Он создаёт объекты (бины) и управляет их жизненным циклом, внедряя зависимости (Dependency Injection). Основные способы настройки DI:
- Аннотации `@Component`, `@Service`, `@Repository` и т.д. + `@Autowired` для внедрения зависимостей.
- Java-based конфигурация: `@Configuration`, методы с `@Bean`.
- XML-конфигурация (реже используется в современных проектах).

**Примеры домашних заданий**:
1. научиться отделять создание объектов (бинов) от их логики и связывать их через контейнер, а не вручную, уменьшая связанность кода.
2. **CRM-система (монолит)**: Spring Boot + Data JPA + Security. CRUD по клиентам/заказам.
3. **Добавить BPMN** (Camunda) в CRM-проект: модель процесса (approve/reject заказа).
4. **Настроить CI** (GitLab CI/Jenkins): сборка, тесты, результаты отчётов.
5. (Опционально) Подключить **GraphQL** (Spring GraphQL) для некоторых запросов.

 ---
### Месяц 5 CI/CD

| **Блок**                    | **Описание и цели**                                                                                                                   | **Документация**                                                                                                                             |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Jenkins**               | Традиционный сервер для автоматизации сборок, запуск тестов, интеграции и деплоя. Создание pipeline, настройка job’ов через web-интерфейс или Jenkinsfile.                                                                 | – [Jenkins (Official)](https://www.jenkins.io/) <br> – [Jenkins Getting Started](https://www.jenkins.io/doc/pipeline/tour/getting-started/)                                                                                               |
| **GitLab CI**             | Встроенный механизм CI/CD в GitLab. Описание pipeline в `.gitlab-ci.yml`. Запуск тестов, сборка Docker-образов, публикация артефактов.                                                                                    | – [GitLab CI Docs](https://docs.gitlab.com/ee/ci/)                                                                                                                                                 |
| **Code Review**           | Процесс командного ревью кода (Merge Request / Pull Request). Анализ изменений, комментарии к коду, одобрение от коллег.                                                                                                 | – [Pull Request (GitHub Docs)](https://docs.github.com/en/pull-requests) <br> – [GitLab Merge Requests](https://docs.gitlab.com/ee/user/project/merge_requests/)                                                                          |
| **Code Coverage**         | Автоматическая проверка покрытого кода тестами (например, JaCoCo, Cobertura). Сколько % строк/ветвей покрыто тестами.                                                                                                     | – [JaCoCo (GitHub)](https://github.com/jacoco/jacoco)                                                                                                                                               |
| **Code Quality**          | Анализ статического качества кода (SpotBugs, SonarQube, PMD). Поиск потенциальных багов, несоответствий стилю, бесполезных кусков кода.                                                                                    | – [SonarQube](https://www.sonarqube.org/) <br> – [SpotBugs (GitHub)](https://github.com/spotbugs/spotbugs)                                                                                          |
| **Helm** (Chart)          | Пакетный менеджер для Kubernetes. Сбор манифестов (`Deployment`, `Service`, `ConfigMap`) в один «Chart» и удобный деплой.                                                                                               | – [Helm (Official)](https://helm.sh/docs/)                                                                                                                                                          |
| **WireMock** (при желании)| Локальная заглушка (mock) внешних API. Интеграционное тестирование без реальных внешних сервисов.                                                                                                                         | – [WireMock (Official)](https://wiremock.org/)                                                                                                                                                      |
| **SonarQube** (снова)     | Централизованный инструмент Code Quality, поддерживает плагины для Jenkins/GitLab.                                                                                                                                         | – [SonarQube (Official)](https://www.sonarqube.org/)                                                                                                                                                |

**Краткое описание**:
- **CI** (Continuous Integration) означает, что каждое изменение кода (commit/pull request) автоматически собирается и тестируется в общей ветке, чтобы сразу выявлять проблемы.
- **CD** (Continuous Delivery/Deployment) расширяет идею CI, добавляя автоматический деплой в тестовые или даже прод-среды при успехе сборки и тестов.

Минимальная реализация для учебного проекта:
1. **Pipeline**: шаги (build → test → build Docker image).
2. **Результаты тестов**: Unit-тесты (JUnit), Integration-тесты (при необходимости).
3. (Опционально) Автоматический деплой в локальный Kubernetes (minikube) или удалённый сервер.

Задача — уметь прописать `.gitlab-ci.yml` или GitHub Actions workflow (или конфиг для Jenkins), настроить сборку и проверить, что тесты проходят успешно.


## Месяц 5-6: Cloud, микросервисы и Job Interview

| **Блок**                   | **Описание и цели**                                                                                                                                                                                                                                                                                                                                                                                                                       | **Документация**                                                                                                                                                                                                                                |
 |----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Docker, Docker Compose** | Контейнеризация (создание Dockerfile, docker-compose для приложения + БД). Уметь упаковывать и переносить приложения, собирать стенды локально.                                                                                                                                                                                                                                                                                           | – [Docker Docs](https://docs.docker.com/)                                                                                                                                                                                                              |
| **Kubernetes**             | Оркестрация контейнеров: Pods, Deployment, Service, ConfigMap, Secrets. Пробный деплой в minikube. Понять принципы работы кластера.                                                                                                                                                                                                                                                                                                       | – [Kubernetes Docs](https://kubernetes.io/docs/home/)                                                                                                                                                                                                  |
| **Microservices**          | Разделение монолита, Spring Cloud (Config, Eureka, Gateway), Micropatterns (circuit breaker, retry, saga, 2 Phase Commit).                                                                                                                                                                                                                                                                                                                | – [Spring Cloud (Official)](https://spring.io/projects/spring-cloud) <br> – [Microservices.io Patterns](https://microservices.io/patterns/)                                                                                                            |
| **Cloud** (Keycloak, Kafka, ELK, ActiveMQ, Spring Cloud Stream, OpenFeign, Spring Cloud Config) | Инструменты для аутентификации (Keycloak), асинхронной коммуникации (Kafka, ActiveMQ), логирования/мониторинга (ELK), обмена сообщениями, сервис-конфигурации (Config, OpenFeign).                                                                                                                                                                                                                                                        | – [Keycloak Docs](https://www.keycloak.org/documentation) <br> – [Apache Kafka](https://kafka.apache.org/documentation/) <br> – [ELK Stack](https://www.elastic.co/guide/index.html) <br> – [ActiveMQ Docs](https://activemq.apache.org/)              |
| **Code Review / Code Quality** | Обзор кода: SonarQube, SpotBugs, Code Coverage. Цель — поддерживать высокий уровень качества.                                                                                                                                                                                                                                                                                                                                             | – [SonarQube](https://www.sonarqube.org/) <br> – [SpotBugs (GitHub)](https://github.com/spotbugs/spotbugs)                                                                                                                                             |
| **Career Track (CV/Resume, Job Interview)** | Подготовка к собеседованию: пройтись по Java Core, Spring, SQL, Docker/K8s, Keycloak, BPMN, микросервисы. Составить резюме, потренироваться на примерах вопросов, рассмотреть разные варианты трудоустройства.  Записывать каждый скрининг, интервью (захватом экрана, диктофоном). Анализировать вопросы и разбирать свои ответы. Сравнивать их с записями (мок и реальные интервью других людей). Повторять, пока не получишь оффер. | – [Top Spring Interview Questions](https://www.baeldung.com/spring-interview-questions) <br> – [Java Interview Q&A (GeeksForGeeks)](https://www.geeksforgeeks.org/java-interview-questions/)  </br> – [Recording screen cast (OBS)](https://obsproject.com/) |

**Примеры домашних заданий**:
1. **Разбить CRM на микросервисы**: Spring Cloud (Eureka, Feign, Gateway, Config Server). Общение через REST/Kafka.
2. **Контейнеризация**: `docker-compose.yml`, задеплоить в Kubernetes (minikube).
3. **Keycloak**: настроить OAuth2, проверить вход в микросервис через Keycloak.
4. **ELK**: минимум подключить Elasticsearch, смотреть, как логи пишутся в Kibana.
5. **Interview-репетиция**: список ~50 вопросов, пройтись устно (Java Core, Spring, SQL, Docker, Kubernetes, etc.).
6. (Опционально) **Telegram-бот**: связать с микросервисом, добавить Keycloak.
7. (Опционально) **Онлайн биржа**: храним заявки в Kafka, рассчитываем курс, смотрим логи в Kibana.

 ---

## Заключение
Пройдя все эти блоки, вы осваиваете **основы** (Core) и **продвинутые темы** (Advanced), учитесь работать с **SQL** и ORM (JPA/Hibernate), осваиваете **Spring-экосистему** (Boot, Security, Data, AOP, WebFlux, GraphQL), знакомитесь с **Camunda BPMN** для бизнес-процессов, учитесь **контейнеризации** (Docker, Compose) и **оркестрации** (Kubernetes), внедряете **Keycloak** для аутентификации, используете **Kafka** и **ActiveMQ** для асинхронного взаимодействия, а также изучаете **микросервисные паттерны**.

В конце вы готовитесь к **Job Interview** — составляете резюме, отвечаете на частые вопросы (Java, Spring, микросервисы, CI/CD). Таким образом, через **4–6 месяцев** при занятости 15–20 часов в неделю можно добиться уровня, близкого к **Middle Java Developer**, имея в портфолио финальный проект и набор мини-проектов.
 
---

## Контакты
- Телеграм-канал: [@mentee_power](https://t.me/mentee_power)
- Личный телеграм: *@daniil_it_engineer*
