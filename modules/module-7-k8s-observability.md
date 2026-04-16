# Модуль 7: Kubernetes и наблюдаемость

## Содержание

- [Для кого этот модуль](#для-кого-этот-модуль)
- [Зачем этот модуль](#зачем-этот-модуль)
- [Что меняется по сравнению с модулем 6](#что-меняется-по-сравнению-с-модулем-6)
- [FR и NFR](#fr-и-nfr)
- [Технологический стек и инструменты](#технологический-стек-и-инструменты)
- [Что строим / как проект эволюционирует](#что-строим--как-проект-эволюционирует)
- [Архитектура модуля 7: микросервисы в Kubernetes](#архитектура-модуля-7-микросервисы-в-kubernetes)
- [Перед стартом](#перед-стартом)
- [Как проверить результат](#как-проверить-результат)
- [Публичные ресурсы](#публичные-ресурсы)

---

## Для кого этот модуль

**Модуль 6 завершён** -- 4 бизнес-сервиса + API-шлюз развёрнуты через Docker Compose, DDD bounded contexts определены, каждый сервис владеет собственной базой данных, Feign-клиенты обеспечивают межсервисные вызовы, Retry + Circuit Breaker защищают от каскадных сбоев, секреты хранятся в Vault.

**Или вы не проходили модули 1--6**, но знакомы с Java, Spring Boot, Docker, базами данных и микросервисной архитектурой. Этот модуль -- инфраструктурный: он фокусируется на деплое и наблюдаемости, а не на дизайне сервисов. В таком случае стартуйте отсюда -- потребуется самостоятельно воспроизвести FR/NFR предыдущих модулей (или использовать любой микросервисный проект из 3+ сервисов).

---

## Зачем этот модуль

Модуль 6 дал микросервисную архитектуру: 4 бизнес-сервиса + API-шлюз, поднимаемые одной командой через Docker Compose. Но Docker Compose -- это инструмент для локальной разработки. Когда сервисов становится больше одного, появляются вопросы, которые Compose не решает: как обнаруживать сервисы без захардкоженных имён, как масштабировать отдельные сервисы независимо, как перезапускать только упавшие контейнеры без остановки всего стека, как видеть, что происходит между сервисами.

Kubernetes отвечает на эти вопросы: `Deployment` управляет жизненным циклом подов, `Service` даёт стабильное DNS-имя и балансировку, `Ingress` открывает внешний доступ, `ConfigMap` и `Secret` выносят конфигурацию из образов, `HPA` добавляет автоматическое масштабирование.

Сервисы переносятся в Kubernetes. Наблюдаемость превращается из "я смотрю логи" в "я понимаю, как система ведёт себя с точки зрения пользователя". SLI/SLO -- мост между техническим и бизнес-миром: "p95 latency < 200ms" = "пользователь не ждёт слишком долго". Observability -- это продукт, а не приятное дополнение.

Вместе с переносом в Kubernetes появляется наблюдаемость (observability) -- три столпа, без которых эксплуатация микросервисов превращается в гадание: **метрики** (Prometheus + Grafana), **трассировки** (Jaeger) и **логи** (EFK стек). Один HTTP-запрос к API-шлюзу проходит через 3--4 сервиса -- без distributed tracing невозможно понять, где задержка; без централизованного логирования -- невозможно собрать логи с десятков подов.

Общие правила roadmap, FR/NFR и принцип "один проект развивается от модуля к модулю" описаны в [корневом README roadmap](../README.MD).

**Результат модуля:** весь микросервисный стек из модуля 6 развёрнут в локальном Kubernetes-кластере через Helm umbrella chart, с health probes, resource limits, Prometheus + Grafana dashboards, Jaeger distributed tracing и централизованным логированием.

---

## Что меняется по сравнению с модулем 6

| Что было (M6 -- Docker Compose) | Что становится (M7 -- Kubernetes) |
|---|---|
| `docker compose up` | `helm install` / `kubectl apply` |
| Compose service names (`event-service:8080`) | K8s DNS (`event-service.party.svc.cluster.local:8080`) |
| Compose volumes для данных | ConfigMap / Secret / PersistentVolumeClaim |
| Docker Compose healthchecks | K8s liveness/readiness probes + Spring Boot Actuator |
| `docker compose logs` | EFK стек + Kibana (централизованное логирование) |
| Нет трассировки | Jaeger distributed tracing через цепочку сервисов |
| Нет метрик | Prometheus + Grafana dashboards |
| Keycloak в Docker Compose | Keycloak в K8s через Helm + NGINX Ingress TLS |
| Ручной scaling | HPA autoscaling по CPU/memory |
| Vault (dev mode) в Compose | K8s Secrets или Vault в K8s (trade-offs) |

---

## FR и NFR

> Общая рамка: [README -- Как работать с требованиями](../README.MD#как-работать-с-требованиями).

### Функциональные требования

**Kubernetes-деплой:**

- Все 4 бизнес-сервиса + API gateway из модуля 6 развёрнуты в локальном K8s-кластере (minikube)
- Каждый сервис имеет `Deployment`, `Service` и доступен через K8s DNS
- API gateway доступен извне через `Ingress`
- Конфигурация сервисов вынесена в `ConfigMap` и `Secret` (DB credentials, URL сервисов)
- Resource requests/limits указаны для каждого контейнера
- End-to-end сценарий: Ingress -- gateway -- event -- registration -- notification работает в K8s

**Управление жизненным циклом:**

- Health probes (liveness + readiness) настроены на каждом сервисе через Spring Boot Actuator — бизнес-мотивация: Kubernetes автоматически убирает из балансировки поды, которые не готовы обслуживать запросы, и перезапускает зависшие; пользователи не попадают на сломанные экземпляры
- Helm umbrella chart описывает деплой всего стека одной командой
- K8s RBAC настроен: ServiceAccount для каждого сервиса, минимально необходимые права (least privilege)

**Наблюдаемость (observability):**

- Prometheus собирает метрики с каждого сервиса через `/actuator/prometheus` — бизнес-мотивация: без метрик невозможно понять, как система ведёт себя под нагрузкой
- Grafana отображает дашборды с метриками Spring Boot микросервисов (HTTP-запросы, JVM, connection pools) — бизнес-мотивация: дашборд переводит технические метрики в язык, понятный стейкхолдерам
- Jaeger показывает distributed traces для HTTP-запросов через цепочку сервисов — бизнес-мотивация: один запрос проходит 3-4 сервиса; без трассировки невозможно найти, где задержка, влияющая на пользователя
- Для fan-out сценариев видно, какие downstream-ветки шли параллельно, какая из них стала самой медленной и не упёрлись ли сервисы в потоки, connection pool или сетевые лимиты

**SLI/SLO/SLA — метрики надёжности:**

- SLI (Service Level Indicator) измерим для каждого сервиса: availability (доля 2xx), latency (p50/p95/p99), error rate (доля 5xx) — бизнес-мотивация: SLI превращает "кажется, работает медленно" в измеримый факт
- SLO (Service Level Objective) задан как целевое значение: availability ≥ 99.5%, latency p95 ≤ 200ms для критичных сервисов — бизнес-мотивация: SLO — это обещание пользователю; если нарушено — стейкхолдер страдает
- Grafana дашборд отображает текущие SLI относительно SLO-таргетов: на каком percentile задержка превышает цель — бизнес-мотивация: если показатель падает, понятно, какой стейкхолдер страдает и как быстро нужно реагировать
- Error budget (допустимый объём ошибок за период) рассчитывается из SLO и виден на дашборде
- Alerting: Prometheus alert срабатывает, когда SLI приближается к границе SLO (например, error rate > 0.3% при SLO 0.5%)

**API Gateway в Kubernetes:**

- API-шлюз из модуля 6 развёрнут в K8s и доступен через NGINX Ingress
- Авторизация вынесена на уровень инфраструктуры: Keycloak (или Open Policy Agent) проверяет JWT-токен на Ingress/gateway, бизнес-сервисы получают уже валидированный токен
- Бизнес-сервисы могут читать claims из токена (роли, user ID), но не валидируют его повторно — это ответственность gateway-слоя
- Маршрутизация трафика управляется через Ingress rules и gateway configuration, а не через код бизнес-сервисов

### Нефункциональные требования

- Кластер поднимается воспроизводимо через minikube + Helm (воспроизводимость на чистой машине)
- Probes корректно отражают готовность сервисов (проверка зависимостей, а не просто HTTP 200)
- Метрики покрывают HTTP-запросы, JVM (включая состояние потоков: живые потоки, daemon/non-daemon), connection pools и бизнес-операции
- Distributed tracing позволяет увидеть полный путь запроса через gateway и бизнес-сервисы
- Requests/limits настроены так, чтобы сервисы не конкурировали за ресурсы
- Параллельные межсервисные сценарии наблюдаемы: по trace и метрикам видно, какие ветки шли одновременно, где образовалась очередь и какой ресурс стал узким местом
- SLI/SLO: Grafana дашборд показывает текущие SLI относительно SLO-таргетов для каждого сервиса; alert срабатывает при приближении к границе SLO
- Диагностика потоков доступна в K8s: при latency-спайке, который не объясняется CPU или memory, разработчик может получить состояние потоков пода и определить, чем они заняты — ожидают ли подключения к БД, заблокированы ли на I/O, или исчерпали пул
- ★ Профилирование в K8s доступно: разработчик может запустить запись событий (JFR) в поде и проанализировать результаты

**AI-инструменты и агентная работа:**

- K8s MCP: AI деплоит Helm charts, диагностирует поды, анализирует resource usage — AI выступает как K8s-оператор, а не заменяет понимание манифестов
- Grafana MCP: AI выполняет PromQL-запросы, строит SLI/SLO дашборды, помогает с настройкой alerting
- Jaeger MCP: AI ищет distributed traces по сервисам, находит медленные span-ы, прослеживает путь запроса через цепочку сервисов
- Elasticsearch MCP: AI анализирует логи, коррелирует с traces через KQL/ES|QL
- SLI/SLO дашборд: SLI-метрики видны относительно SLO-таргетов; если показатель падает, понятно, какой стейкхолдер страдает
- Сгенерированные AI манифесты и Helm values проверяются через деплой в minikube, проверку probes и запуск end-to-end сценария — AI не снимает ответственности за корректность конфигурации кластера

### Приоритизация тем

| Приоритет | Тема |
|---|---|
| Обязательно | minikube/kind, Deployment, Service, Ingress для всех сервисов из модуля 6 |
| Обязательно | Health Probes (liveness + readiness) через Actuator |
| Обязательно | ConfigMap + Secret для конфигурации микросервисов |
| Обязательно | Helm umbrella chart для multi-service деплоя |
| Обязательно | Prometheus + Grafana (kube-prometheus-stack) |
| Обязательно | Jaeger distributed tracing (all-in-one) |
| Обязательно | Resource Requests/Limits |
| Обязательно | SLI/SLO: метрики надёжности + Grafana дашборд + error budget |
| Обязательно | Наблюдаемость fan-out сценариев: параллельные вызовы к независимым сервисам видны в trace и метриках |
| Рекомендуется | Keycloak в K8s через Helm + NGINX Ingress TLS + Network Policies + K8s RBAC |
| Рекомендуется | HPA (Horizontal Pod Autoscaler) |
| Рекомендуется | GitOps-подход: манифесты и Helm values в Git |
| Рекомендуется | EFK/Kibana (централизованное логирование) |
| Опционально | CI/CD: GitHub Actions + setup-minikube |
| Опционально | Istio / Service Mesh (только на уровне концепции) |

### Что вне scope модуля

- Kafka и асинхронные события -- модуль 8
- ArgoCD / Flux / GitOps
- Jenkins и сложный multi-env pipeline
- Istio как обязательный блокер

---

## Технологический стек и инструменты

| Категория | Что используем | Минимум для Done |
|---|---|---|
| Локальный кластер | minikube (основной) / kind (для CI) | Кластер поднимается, сервисы деплоятся |
| Управление кластером | `kubectl` | Состояние, логи, события, проброс портов |
| Управление конфигурацией | Git + Helm values | Манифесты K8s и Helm values версонируются в Git |
| Деплой | Helm umbrella chart | Весь стек поднимается одной командой |
| Сервисное обнаружение | K8s DNS (CoreDNS) | Сервисы находят друг друга по DNS-имени |
| Конфигурация | ConfigMap + Secret | Настройки и секреты не вшиты в образ |
| Проверки состояния | liveness/readiness probes + Actuator | K8s понимает, когда сервис готов и когда его нужно перезапустить |
| Ресурсы | requests/limits | CPU и memory указаны для каждого контейнера |
| Метрики | kube-prometheus-stack (Prometheus + Grafana) | Метрики Spring Boot собираются и отображаются на дашбордах |
| SLI/SLO | Prometheus rules + Grafana dashboards | SLI-метрики видны относительно SLO-таргетов, error budget отслеживается |
| Трассировка | Jaeger all-in-one + OpenTelemetry | Distributed traces видны через цепочку сервисов |
| Логирование | EFK (Fluent Bit + Elasticsearch + Kibana) | Логи всех подов доступны в едином интерфейсе |
| Авторизация | Keycloak (JWT) + K8s RBAC + NGINX Ingress TLS | JWT-токены для API, RBAC для доступа к кластеру, TLS на Ingress |
| Автоскейлинг | HPA | Pod'ы масштабируются по нагрузке |
| AI | Qwen Code CLI + K8s MCP + Grafana MCP + Jaeger MCP + Elasticsearch MCP | AI-SRE: диагностика кластера, SLI/SLO запросы, корреляция логов и traces |
| Backend runtime | Spring Boot + Actuator + Micrometer | `/actuator/prometheus`, `/actuator/health/liveness`, `/actuator/health/readiness` |

---

## Что строим / как проект эволюционирует

**Стартовая точка:** 4 бизнес-сервиса + API gateway из модуля 6, поднимаемые через Docker Compose с отдельными PostgreSQL-базами, Feign-клиентами, Retry + Circuit Breaker, Vault и Keycloak.

### До -- после

| Слой / аспект | До (модуль 6) | После (модуль 7) |
|---|---|---|
| Среда запуска | Docker Compose | Kubernetes (minikube) |
| Обнаружение сервисов | Compose service names | K8s DNS (CoreDNS) |
| Деплой | `docker compose --profile full up` | `helm install` (umbrella chart) |
| Конфигурация | application.yml + Vault | ConfigMap + Secret + Helm values |
| Health checks | Compose healthchecks | K8s probes (liveness/readiness) |
| Метрики | Нет | Prometheus + Grafana dashboards |
| Трассировка | Нет | Jaeger distributed tracing |
| Логирование | `docker compose logs` | EFK стек + Kibana |
| Масштабирование | Ручное (replicas в compose) | HPA autoscaling |
| Сетевая безопасность | Docker Compose network | Network Policies |
| Авторизация | Keycloak в Docker Compose | Keycloak в K8s + Ingress TLS |

### К концу модуля система умеет

- разворачиваться в Kubernetes через Helm umbrella chart одной командой
- обнаруживать сервисы через K8s DNS вместо захардкоженных имён
- автоматически перезапускать упавшие поды через liveness probe
- исключать неготовые поды из балансировки через readiness probe
- собирать метрики HTTP-запросов, JVM и connection pools через Prometheus
- отображать метрики на Grafana dashboards
- показывать полный путь HTTP-запроса через цепочку сервисов в Jaeger
- показывать в trace параллельные вызовы к независимым сервисам и быстро находить самую медленную ветку
- агрегировать логи со всех подов в Kibana
- масштабировать поды автоматически через HPA
- ограничивать сетевой трафик между сервисами через Network Policies

---

## Архитектура модуля 7: микросервисы в Kubernetes

Микросервисный стек из модуля 6 переезжает из Docker Compose в Kubernetes. Каждый сервис получает набор K8s-объектов: `Deployment`, `Service`, `ConfigMap`, `Secret`, probes. Весь стек описывается через Helm umbrella chart.

### K8s-объекты для сервисов

| Сервис | Deployment | Service | Ingress | ConfigMap | Secret |
|---|---|---|---|---|---|
| event-service | replicas, probes, resources | ClusterIP, K8s DNS | -- | service URLs, feature flags | DB credentials |
| registration-service | replicas, probes, resources | ClusterIP, K8s DNS | -- | service URLs, timeouts | DB credentials |
| user-service | replicas, probes, resources | ClusterIP, K8s DNS | -- | service URLs | DB credentials |
| notification-service | replicas, probes, resources | ClusterIP, K8s DNS | -- | service URLs, SMTP config | DB credentials, API keys |
| api-gateway | replicas, probes, resources | ClusterIP, K8s DNS | NGINX Ingress | routes, downstream URLs | Keycloak client secret |

### Observability стек

| Компонент | Роль | Развёртывание |
|---|---|---|
| Prometheus | Сбор метрик с сервисов через `/actuator/prometheus` | kube-prometheus-stack Helm chart |
| Grafana | Визуализация метрик на дашбордах | kube-prometheus-stack Helm chart |
| Jaeger | Distributed tracing через цепочку сервисов | Jaeger all-in-one Helm chart |
| Fluent Bit | Сбор логов с узлов (DaemonSet) | Helm chart |
| Elasticsearch | Хранение и индексация логов | Helm chart |
| Kibana | Поиск и визуализация логов | Helm chart |

### Service discovery через K8s DNS

Docker Compose предоставляет имена сервисов автоматически (`http://event-service:8080`). В Kubernetes эту роль выполняет CoreDNS: каждый `Service` получает DNS-имя вида `service-name.namespace.svc.cluster.local`. Для вызова event-service из registration-service внутри кластера достаточно адреса `event-service:8080` -- если сервисы в одном namespace.

### Helm umbrella chart

Umbrella chart объединяет деплой всех сервисов и инфраструктурных компонентов в один Helm-релиз: общие `values.yaml` для конфигурации, отдельные subcharts для каждого сервиса, единый механизм обновлений.

### Параллелизм в scope модуля

Модуль 7 не вводит новый прикладной параллелизм: он требует, чтобы fan-out сценарии из модулей 6 и 8 были видны в trace, метриках и диагностике пода.

---

## Перед стартом

- модуль 6 завершён: 4 бизнес-сервиса + API gateway работают через Docker Compose;
- `./gradlew build` зелёный во всех сервисах;
- Docker и `docker compose` установлены;
- установлен `kubectl`;
- установлен `minikube` или `kind`;
- установлен `helm`;
- есть базовое понимание YAML, Docker-образов и переменных окружения.

---

## Как проверить результат

| Что проверяем | Как проверить | Что ожидаем |
|---|---|---|
| K8s-кластер | `minikube status` | кластер запущен |
| Helm-деплой | `helm install party ./chart` | все поды в состоянии `Running` и `Ready` |
| Сервисы в кластере | `kubectl get deployments,pods,services` | 4 бизнес-сервиса + gateway имеют Deployment и Service |
| K8s DNS | Вызов event-service из registration-service через K8s DNS-имя | межсервисный вызов отрабатывает |
| Ingress | HTTP-запрос через Ingress к gateway | запрос маршрутизируется к нужному сервису |
| End-to-end | Создание мероприятия -- запись участника -- уведомление | цепочка проходит через Ingress -- gateway -- event -- registration -- notification |
| Health probes | `kubectl describe pod` | liveness и readiness probes настроены, поды проходят проверки |
| ConfigMap/Secret | `kubectl get configmap,secret` | конфигурация вынесена, DB credentials не захардкожены |
| Resource limits | `kubectl describe pod` | requests и limits указаны для каждого контейнера |
| Prometheus | Prometheus UI: targets | каждый сервис виден как target, `/actuator/prometheus` доступен |
| Grafana | Grafana dashboard | метрики HTTP, JVM, connection pools отображаются |
| Jaeger | Jaeger UI: поиск trace | distributed trace показывает путь запроса через цепочку сервисов |
| Fan-out trace | Вызвать сценарий, где gateway или сервис обращается к 2 независимым downstream-сервисам | в Jaeger видно параллельные ветки и их длительность; самая медленная ветка определяется явно |
| Fan-out metrics | Нагрузить агрегирующий сценарий | Grafana показывает влияние fan-out на latency, thread metrics и connection pools |
| HPA (рекомендуется) | `kubectl get hpa` | autoscaler настроен и реагирует на нагрузку |
| Network Policies (рекомендуется) | `kubectl get networkpolicies` | трафик между сервисами ограничен явными правилами |
| Thread dump (под) | `kubectl exec <pod> -- jcmd 1 Thread.print` | состояние потоков получено из пода, видно, где потоки ожидают |
| JFR-профилирование (★) | Запустить JFR recording в поде через `jcmd` | профиль записан, анализируется локально |

---

## Публичные ресурсы

### Kubernetes: основы

| Тема | Источник |
|---|---|
| K8s: Deployments | [kubernetes.io -- Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) |
| K8s: Services | [kubernetes.io -- Services](https://kubernetes.io/docs/concepts/services-networking/service/) |
| K8s: DNS для сервисов | [kubernetes.io -- DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) |
| K8s: Ingress | [kubernetes.io -- Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) |
| Minikube: документация | [minikube.sigs.k8s.io](https://minikube.sigs.k8s.io/docs/start/) |
| Kind: документация | [kind.sigs.k8s.io](https://kind.sigs.k8s.io/docs/user/quick-start/) |
| Развёртывание микросервисов в K8s | [Хабр -- Учимся разворачивать микросервисы. Kubernetes](https://habr.com/ru/post/488796/) |
| Minikube: практические советы | [Хабр (VK) -- Как работать с Minikube](https://habr.com/ru/companies/vk/articles/648117/) |

### Helm

| Тема | Источник |
|---|---|
| Helm: документация | [helm.sh/docs](https://helm.sh/docs/) |
| Helm: Chart Template Guide | [helm.sh -- Chart Template Guide](https://helm.sh/docs/chart_template_guide/) |
| Основы Helm чартов | [Хабр -- Основы работы с Helm чартами](https://habr.com/ru/articles/547682/) |
| Helm для Kubernetes | [VK Cloud -- Helm для Kubernetes](https://cloud.vk.com/blog/helm-dlya-kubernetes-kak-uprostit-deploi) |
| Деплой микросервисов через Helm | [CNCF -- Deploying microservices with Helm](https://www.cncf.io/blog/2024/08/09/deploying-a-microservices-application-using-helm-on-kubernetes/) |

### ConfigMap, Secret, Probes, Resources

| Тема | Источник |
|---|---|
| ConfigMap | [kubernetes.io -- ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) |
| Secret | [kubernetes.io -- Secret](https://kubernetes.io/docs/concepts/configuration/secret/) |
| ConfigMap и Secrets: best practices | [Хабр (Beeline Cloud) -- ConfigMaps и Secrets](https://habr.com/ru/companies/beeline_cloud/articles/864222/) |
| Liveness, Readiness, Startup Probes | [kubernetes.io -- Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) |
| Spring Boot Probes | [Baeldung -- Liveness and Readiness Probes](https://www.baeldung.com/spring-liveness-readiness-probes) |
| Spring Blog -- Probes в Spring Boot | [spring.io -- Liveness and Readiness Probes](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot) |
| Resource Requests и Limits | [kubernetes.io -- Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) |
| Requests и Limits: подробно | [Хабр (Timeweb) -- Запросы и лимиты в K8s](https://habr.com/ru/companies/timeweb/articles/835068/) |
| K8s Best Practices | [VK Cloud -- Kubernetes Best Practices](https://cloud.vk.com/blog/kubernetes-best-practices/) |

### Observability: Prometheus + Grafana + Jaeger + EFK

| Тема | Источник |
|---|---|
| kube-prometheus-stack Helm chart | [GitHub -- prometheus-community/helm-charts](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) |
| Spring Boot Actuator Metrics | [spring.io -- Actuator Metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html) |
| Spring Boot мониторинг в K8s | [BellSoft -- Spring Boot мониторинг в K8s с Prometheus и Grafana](https://bell-sw.com/blog/spring-boot-monitoring-in-kubernetes-with-prometheus-and-grafana/) |
| Observability: три столпа | [Хабр (OTUS) -- Observability для микросервисов](https://habr.com/ru/companies/otus/articles/725000/) |
| Prometheus + Grafana в K8s | [Хабр (OTUS) -- Kubernetes Observability: Prometheus and Grafana](https://habr.com/ru/companies/otus/articles/719824/) |
| Jaeger: документация | [jaegertracing.io](https://www.jaegertracing.io/) |
| Distributed tracing с Jaeger | [Хабр (Slurm) -- Распределённая трассировка с Jaeger и OpenTelemetry](https://habr.com/ru/companies/slurm/articles/574322/) |
| EFK: логгинг в K8s | [Хабр (OTUS) -- Kubernetes Observability: логгинг с EFK](https://habr.com/ru/companies/otus/articles/721004/) |
| Логирование в Kubernetes | [Хабр (Southbridge) -- Логирование в Kubernetes](https://habr.com/ru/company/southbridge/blog/517636/) |
| JVM thread metrics в Micrometer | [Micrometer -- JVM Metrics](https://docs.micrometer.io/micrometer/reference/reference/jvm-metrics.html) — `jvm_threads_live`, `jvm_threads_states` |
| Thread dump: jcmd и jstack | [Baeldung -- Java Thread Dump](https://www.baeldung.com/java-thread-dump) — методы получения и анализа |

### Keycloak в K8s + Network Policies + HPA

| Тема | Источник |
|---|---|
| Keycloak Operator | [keycloak.org -- Basic Deployment](https://www.keycloak.org/operator/basic-deployment) |
| Keycloak и микросервисы | [Хабр -- KeyCloak и микросервисы](https://habr.com/ru/articles/720070/) |
| Keycloak в K8s | [Хабр (Касперский) -- Keycloak Standalone-HA в K8s](https://habr.com/ru/companies/kaspersky/articles/763790/) |
| Open Policy Agent + K8s | [OPA -- Kubernetes Admission Control](https://www.openpolicyagent.org/docs/latest/kubernetes-introduction/) |
| OPA Gatekeeper | [Gatekeeper -- Getting Started](https://open-policy-agent.github.io/gatekeeper/website/docs/) |
| cert-manager + Ingress TLS | [cert-manager.io -- Securing NGINX Ingress](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/) |
| Ingress-NGINX TLS | [kubernetes.github.io -- Ingress-NGINX TLS](https://kubernetes.github.io/ingress-nginx/user-guide/tls/) |
| Network Policies | [kubernetes.io -- Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) |
| Network Policies: практика | [Хабр (Beeline Cloud) -- Погружение в Network Policies](https://habr.com/ru/companies/beeline_cloud/articles/857972/) |
| HPA | [kubernetes.io -- Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) |
| Автоскейлинг HPA, VPA, CA | [Хабр (OTUS) -- Автоскейлинг в K8s](https://habr.com/ru/companies/otus/articles/818945/) |

### SLI/SLO/SLA

| Тема | Источник |
|---|---|
| Google SRE: SLI/SLO/SLA | [Google SRE Book -- Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) |
| SLI/SLO на практике | [Хабр (Авито) -- SLO: как мы начали измерять надёжность](https://habr.com/ru/companies/avito/articles/501864/) |
| SRE для разработчиков | [Хабр (Яндекс) -- Практика SRE](https://habr.com/ru/companies/yandex/articles/439546/) |
| Error budget и SLO | [Google SRE Workbook -- Error Budgets](https://sre.google/workbook/error-budget-policy/) |
| SLO с Prometheus и Grafana | [Grafana -- SLO dashboards with Prometheus](https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/slo-dashboards/) |
| Pyrra: SLO-monitoring для Prometheus | [GitHub -- pyrra-dev/pyrra](https://github.com/pyrra-dev/pyrra) |

### CI/CD (опционально)

| Тема | Источник |
|---|---|
| Minikube в GitHub Actions | [minikube.sigs.k8s.io -- GitHub Actions tutorial](https://minikube.sigs.k8s.io/docs/tutorials/setup_minikube_in_github_actions/) |
| CI/CD для самых маленьких | [Хабр -- CI/CD на GitHub Actions и GitLab CI](https://habr.com/ru/articles/914560/) |
| CI/CD с GitHub Actions | [Хабр (Флант) -- Настраиваем CI/CD с GitHub Actions](https://habr.com/ru/companies/flant/articles/803251/) |

### Istio / Service Mesh (опционально, только концепция)

| Тема | Источник |
|---|---|
| Istio: документация | [istio.io -- What is Istio](https://istio.io/latest/docs/concepts/what-is-istio/) |
| Service Mesh: знакомство с Istio | [Хабр (Slurm) -- Service mesh в K8s -- знакомство с Istio](https://habr.com/ru/companies/slurm/articles/701318/) |

### AI-инструменты для Kubernetes и Observability

| Тема | Источник |
|---|---|
| K8s MCP Server — kubectl + helm через AI: деплой, диагностика, resource analysis | [GitHub](https://github.com/alexei-led/k8s-mcp-server) |
| Grafana MCP — PromQL, dashboards, alerting через AI | [GitHub](https://github.com/grafana/mcp-grafana) |
| Grafana Tracing MCP — distributed traces через TraceQL | [Grafana Docs](https://grafana.com/docs/grafana-cloud/send-data/traces/mcp-server/) |
| Helm MCP — все 44 операции Helm CLI через AI | [PulseMCP](https://www.pulsemcp.com/servers/zekker6-helm) |
| Elasticsearch MCP — поиск и анализ логов через KQL/ESQL | [GitHub](https://github.com/cr7258/elasticsearch-mcp-server) |
| Jaeger MCP — поиск distributed traces по сервисам | [GitHub](https://github.com/serkan-ozal/jaeger-mcp-server) |
| Принцип: AI-SRE снижает cognitive load при диагностике K8s; верификация результата, а не микроменеджмент | — |
