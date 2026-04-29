## 1. Назначение релиза

**Тип релиза:** ручной релиз на виртуальную машину  
**Репозиторий:** https://github.com/IoTDataPlatform/constructor

Платформа предназначена для быстрого разворачивания потоковой системы обработки данных из готовых инфраструктурных компонентов. Разработчик продукта берёт готовую платформу, выбирает нужные технологии через теги конструктора и заменяет только бизнес-логику: источники данных, обработчики потоков, схемы сообщений, SQL-запросы, правила агрегации и сценарии выдачи данных наружу.

В базовой конфигурации платформа собирается как Docker Compose-проект и может включать Kafka, Kafka Connect, Schema Registry, MQTT, PostgreSQL, Iceberg/S3, Trino, Flink, Kafka Streams и Redis. Конструктор генерирует нужный состав файлов по выбранным тегам, после чего система запускается на виртуальной машине.

---

## 2. Участники релиза

С точки зрения пользователей платформы:
| Сторона | Зона ответственности |
|---|---|---|
| Разработка | Склонировать репозиторий, выбрать необходимые компоненты платформы, реализовать бизнес логику аналитики, сконфигурировать (Java-)коннекторы, Docker Compose |
| Бизнес | Обозначение целей продукта, согласование состава MVP и финального продукта |
| Эксплуатация / DevOps | Подготовка виртуальной машины, переменные окружения, TLS, сетевые правила, запуск и мониторинг контейнеров |

Для демонстрации планируется релиз проекта-примера "датчиков погоды" (генерируется по выбранным тегам).
---

## 3. Цели релиза

### 3.1. Продуктовые цели

1. Дать разработчикам готовый шаблон развертывания потоковой платформы.
2. Предоставить воспроизводимый сценарий запуска на виртуальной машине через Docker Compose.
3. Подготовить релиз не как набор исходников, а как продукт: документация, инструкция запуска, security checklist, tutorial, готовая сборка и release-артефакты.

### 3.2. Функциональные цели

- генерация Docker Compose-конфигурации по тегам конструктора;
- запуск базовой платформы на VM;
- поддержка источника данных через MQTT или Kafka Connect Source Connector;
- доставка событий в Kafka;
- хранение данных в PostgreSQL и/или Iceberg/S3;
- потоковая обработка через Kafka Streams и/или Flink;
- выдача обработанных данных через Redis, PostgreSQL или Trino;
- пример бизнес-логики на синтетических данных датчиков температуры;
- инструкция, как заменить примерную бизнес-логику на бизнес-логику пользователя.

### 3.3. Нефункциональные цели

- воспроизводимый запуск из чистого клона репозитория;
- запуск без ручной установки Kafka, Flink, PostgreSQL и других сервисов на хост;
- минимизация открытых наружу портов;
- отсутствие отладочных UI-контейнеров в production-профиле;
- конфигурация секретов через `.env` или Docker secrets, а не через исходный код;
- документированный план устранения известных уязвимостей.

---

## 4. Состав релиза

### 4.1. Основные компоненты

| Компонент | Назначение | Статус в релизе |
|---|---|---|
| `bricks/filter_tags.py` | Генератор конфигурации по тегам | Есть |
| `bricks/Dockerfile` | Docker-образ конструктора на Python | Есть |
| `docker-compose.yml` | Итоговая конфигурация платформы после генерации | Генерируется |
| Kafka | Брокер событий | Есть |
| Kafka Connect | Интеграция источников и приёмников данных | Есть |
| Schema Registry | Хранение схем сообщений | Есть |
| MQTT | Приём данных от устройств / эмулятор устройств | Есть как тег |
| PostgreSQL | Реляционное хранение и serving-слой | Есть как тег |
| Iceberg + S3/MinIO | Lakehouse-хранение | Есть как тег |
| Trino | Федеративные SQL-запросы | Есть как тег |
| Flink | Потоковая обработка | Есть как тег |
| Redis | In-memory serving/cache | Есть как тег |
| Kafka UI / MQTT Web / другие UI | Отладочные интерфейсы | Только для dev/demo, не для production |

### 4.2. Конфигурации релиза

#### Demo-конфигурация

Используется для демонстрации и проверки функциональности:

```bash
(cd bricks && docker compose run --rm --user "$(id -u):$(id -g)" lg --tags MQTT,POSTGRES,FLINK,REDIS)
docker compose up -d --build
```

#### Production-like конфигурация для VM

Используется для релиза на виртуальной машине. Отличается от demo-конфигурации тем, что:

- наружу публикуется только входная точка продукта;
- Kafka UI, MQTT Web, MinIO Console и другие отладочные UI не запускаются;
- пароли и ключи задаются через `.env` / secrets;
- внешние соединения переводятся на TLS;
- контейнеры подключаются к внутренним Docker-сетям без лишнего `ports`.

Рекомендуемая команда генерации для production-like стенда:

```bash
(cd bricks && docker compose run --rm --user "$(id -u):$(id -g)" lg --tags MQTT,POSTGRES,FLINK,REDIS)
```

После генерации необходимо применить production override:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

Файл `docker-compose.prod.yml` должен быть подготовлен до финального релиза.

---

## 5. Инструкция по развёртыванию на виртуальной машине

### 5.1. Требования к VM

Минимальная конфигурация для демонстрационного стенда:

- Ubuntu Server 22.04/24.04 LTS или совместимый Linux-дистрибутив;
- 4 CPU;
- 8 GB RAM минимум, 16 GB RAM рекомендуется для Flink + Kafka + PostgreSQL + Redis;
- 40 GB свободного диска минимум;
- Docker Engine;
- Docker Compose Plugin;
- Git;
- Git LFS;
- доступ по SSH для DevOps-ответственного.

### 5.2. Подготовка VM

```bash
sudo apt update
sudo apt install -y git git-lfs ca-certificates curl

git lfs install
```

Установка Docker выполняется по официальной инструкции Docker для выбранного дистрибутива. После установки проверить:

```bash
docker --version
docker compose version
```

### 5.3. Клонирование проекта

```bash
git clone https://github.com/IoTDataPlatform/constructor.git
cd constructor
```

### 5.4. Настройка переменных окружения

Создать файл `.env` в корне проекта. Пример:

```env
POSTGRES_DB=weather
POSTGRES_USER=platform_app
POSTGRES_PASSWORD=<strong-random-password>
REDIS_PASSWORD=<strong-random-password>
MINIO_ROOT_USER=<strong-random-user>
MINIO_ROOT_PASSWORD=<strong-random-password>
KAFKA_CLUSTER_ID=<generated-cluster-id>
```

Требования:

- не использовать простые пароли;
- не коммитить `.env` в репозиторий;
- добавить `.env` в `.gitignore`;
- для финального стенда заменить `.env` на Docker secrets или защищённое хранилище секретов.

### 5.5. Генерация состава платформы

Для демонстрационного сценария:

```bash
(cd bricks && docker compose run --rm --user "$(id -u):$(id -g)" lg --tags MQTT,POSTGRES,FLINK,REDIS)
```

Для сценария с Kafka Connect Source вместо MQTT:

```bash
(cd bricks && docker compose run --rm --user "$(id -u):$(id -g)" lg --tags SOURCE,POSTGRES,FLINK,REDIS)
```

Для lakehouse-сценария:

```bash
(cd bricks && docker compose run --rm --user "$(id -u):$(id -g)" lg --tags SOURCE,ICEBERG,TRINO,FLINK)
```

### 5.6. Запуск

```bash
docker compose up -d --build
```

Для production-like запуска:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build
```

### 5.7. Проверка состояния контейнеров

```bash
docker compose ps
docker compose logs --tail=100 kafka-connect
docker compose logs --tail=100 flink-jobmanager
docker compose logs --tail=100 postgres
docker compose logs --tail=100 redis
```

Все основные контейнеры должны быть в состоянии `running` или `healthy`. One-shot init-контейнеры могут завершиться со статусом `exited 0`.

### 5.8. Остановка стенда

```bash
docker compose down
```

Полная очистка данных для повторного запуска:

```bash
docker compose down -v
git clean -fd
```

Команду `docker compose down -v` использовать только на demo/dev-стенде, потому что она удаляет volumes с данными.

---

## 6. Как разработчику заменить бизнес-логику

Платформа разделяет инфраструктурные компоненты и прикладную логику. Для адаптации под новый продукт разработчик меняет только те части, которые относятся к предметной области.

| Что менять | Где менять | Пример |
|---|---|---|
| Источник данных | MQTT producer или Kafka Connect Source Connector | Заменить генератор температуры на чтение телеметрии устройств |
| Схему события | Avro/JSON Schema-конфигурации и модели | Заменить поля `sensorId`, `temperatureF` на поля своей предметной области |
| Правила обработки | Kafka Streams/Flink job | Добавить фильтрацию, enrichment, оконные агрегации, расчёт метрик |
| Хранилище | PostgreSQL init SQL, Iceberg table config | Создать таблицы под нужные события и агрегаты |
| Serving-слой | Redis/PostgreSQL/Trino | Подготовить API/SQL-запросы/кэш для внешних систем |
| Набор технологий | `--tags` конструктора | Выбрать `MQTT,POSTGRES,FLINK,REDIS` или другой набор |

Ожидаемый результат: разработчик не переписывает инфраструктуру, а добавляет бизнес-логику поверх готового data stack.

---

## 7. Туториал / контрольный сценарий использования

### 7.1. Цель сценария

Проверить, что платформа принимает события от эмулятора устройств, доставляет их в Kafka, обрабатывает поток и сохраняет/выдаёт результат через выбранный storage/serving-компонент.

### 7.2. Сценарий: MQTT → Kafka/Flink → PostgreSQL/Redis

1. Склонировать репозиторий:

```bash
git clone https://github.com/IoTDataPlatform/constructor.git
cd constructor
```

2. Сгенерировать конфигурацию:

```bash
(cd bricks && docker compose run --rm --user "$(id -u):$(id -g)" lg --tags MQTT,POSTGRES,FLINK,REDIS)
```

3. Запустить платформу:

```bash
docker compose up -d --build
```

4. Проверить контейнеры:

```bash
docker compose ps
```

5. Проверить поступление данных в Kafka:

```bash
docker compose logs --tail=100 kafka-connect
```

6. Проверить запись данных в PostgreSQL:

```bash
docker compose exec postgres psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -c 'select * from earth_temp order by created_at desc limit 10;'
```

7. Проверить результат обработки в Redis, если Redis включён в сценарий:

```bash
docker compose exec redis redis-cli --scan
```

8. Ожидаемый результат:

- контейнеры платформы запущены;
- события от эмулятора появляются в Kafka;
- данные сохраняются в PostgreSQL;
- агрегаты/последние значения доступны в Redis или другом serving-слое;
- пользователь может заменить примерную логику генерации температуры на свою бизнес-логику.

### 7.3. Критерии успешного прохождения туториала

| Проверка | Ожидаемый результат |
|---|---|
| `docker compose ps` | Основные сервисы запущены |
| Логи Kafka Connect / MQTT | Нет бесконечных ошибок подключения |
| PostgreSQL-запрос | Возвращает последние записи |
| Redis scan/get | Возвращает ключи или агрегированные значения |
| Повторный запуск | Система поднимается из чистого клона по инструкции |

---

## 8. Готовая сборка и релизные артефакты

К релизу подготовлены следующие артефакты:

| Артефакт | Формат | Где находится | Статус |
|---|---|---|---|
| Исходный код конструктора | GitHub repository | `IoTDataPlatform/constructor` | Есть |
| Dockerfile конструктора | `bricks/Dockerfile` | Репозиторий | Есть |
| Архив файлов конструктора | `bricks/fs.tar.gz` через Git LFS | Репозиторий | Есть |
| Инструкция запуска | `README.md` | Репозиторий | Есть |
| План релиза | `RELEASE.md` | Корень репозитория | Этот файл |
| Production override | `docker-compose.prod.yml` | Корень репозитория | Есть (нужно опубликовать) |
| `.env` | `.env.example` | Корень репозитория | Есть (нужно опубликовать) |
| Security checklist | раздел 9 этого файла | в этом файле | Есть |

---

## 9. Безопасность

Безопасность входит в план релиза и проверяется до передачи продукта в эксплуатацию.

### 9.1. Основные угрозы

| ID | Угроза | Риск | Где проявляется | План устранения | Приоритет |
|---|---|---|---|---|---|---|
| SEC-01 | Публикация внутренних портов наружу | Доступ к Kafka, PostgreSQL, Redis, MinIO, Flink, Kafka Connect из интернета | `ports` в Docker Compose | Убрать лишние `ports`, оставить только необходимые entrypoint-порты; использовать внутренние Docker networks | Critical |
| SEC-02 | Отладочные UI в production | Неавторизованное управление Kafka, MQTT, MinIO, Flink или Connect | Kafka UI, MQTT Web, MinIO Console, Flink UI | Не запускать UI-контейнеры в production; вынести их в dev/demo override | Critical |
| SEC-03 | Слабые пароли по умолчанию | Компрометация БД, Redis, MinIO | `.env`, docker-compose environment | Заменить дефолтные пароли на случайные; добавить `.env.example` без секретов | Critical |
| SEC-04 | Секреты в репозитории или образах | Утечка доступов | Dockerfile, compose, init scripts | Не хранить секреты в Git; использовать `.env`, Docker secrets или секреты CI/CD | Critical |
| SEC-05 | Отсутствие TLS для удалённых сервисов | Перехват данных и credentials | Kafka, MQTT, PostgreSQL, Redis, S3/MinIO, Trino | Все удалённые соединения переводить на TLS; plaintext разрешён только внутри закрытой локальной сети VM/demo | High |
| SEC-06 | Запуск контейнеров от root | Повышение последствий при компрометации контейнера | Docker images | По возможности задать non-root user; проверить права volumes | High |
| SEC-07 | Неприкреплённые версии образов | Невоспроизводимые сборки и риск supply chain | `image: latest` | Зафиксировать версии образов; не использовать `latest` в release/prod | High |
| SEC-08 | Отсутствие healthcheck/лимитов ресурсов | Сложная диагностика и DoS внутри VM | Docker Compose services | Добавить healthcheck, restart policy, CPU/memory limits | Medium |
| SEC-09 | Открытый Kafka Connect REST API | Возможность создать вредоносный connector | Kafka Connect | Не публиковать REST API наружу; доступ только из internal network; для внешнего доступа — reverse proxy + auth + TLS | Critical |
| SEC-10 | Неограниченные volumes и логи | Переполнение диска VM | Kafka logs, PostgreSQL data, Docker logs | Настроить retention, log rotation, мониторинг свободного места | Medium |
| SEC-11 | Отсутствие резервного копирования | Потеря данных при сбое VM | PostgreSQL, object storage, volumes | Описать backup/restore для volumes и БД | Medium |
| SEC-12 | Отсутствие разделения dev/prod | Dev-инструменты попадают в релиз | Compose-файлы и теги | Разделить `docker-compose.dev.yml` и `docker-compose.prod.yml` | High |

### 9.2. Требования к сетевой безопасности

1. Контейнеры Kafka, PostgreSQL, Redis, Kafka Connect, Schema Registry, Flink, MinIO и Trino не должны быть доступны из внешней сети напрямую.
2. Все сервисы должны взаимодействовать через внутреннюю Docker-сеть.
3. Наружу публикуется только минимально необходимый API продукта или reverse proxy.
4. Kafka UI, MQTT Web, MinIO Console, Flink Dashboard и другие UI для отладки не входят в production-профиль.
5. Внешние подключения к удалённым сервисам должны использовать TLS.
6. Для MQTT использовать `mqtts`, для Kafka — SSL/SASL_SSL, для PostgreSQL — SSL mode, для Redis — TLS или закрытый сетевой контур, для S3/MinIO — HTTPS.
7. Firewall VM должен запрещать входящие подключения ко всем портам, кроме SSH и публичного entrypoint продукта.
8. SSH-доступ к VM должен быть только по ключам, без парольного входа.

### 9.3. Требования к секретам

- Все пароли, access keys, tokens и private keys хранятся вне репозитория.
- В Git хранится только `.env.example` с пустыми или демонстрационными значениями.
- Перед релизом выполнить проверку:

```bash
git grep -nE 'password|secret|token|access_key|private_key|AKIA|BEGIN RSA|BEGIN PRIVATE'
```

- Для production использовать Docker secrets или секреты инфраструктуры.
- Ротация секретов выполняется перед финальной передачей стенда в эксплуатацию.

### 9.4. Минимальный production override

До финального релиза нужно подготовить `docker-compose.prod.yml`, который:

- удаляет публикацию лишних портов;
- отключает UI/debug-контейнеры;
- задаёт restart policy;
- включает healthcheck;
- задаёт resource limits;
- подключает secrets;
- использует TLS-конфигурацию для внешних подключений.

---

## 10. План тестирования и приёмки

### 10.1. Smoke-тесты

| Тест | Команда / действие | Ожидаемый результат |
|---|---|---|
| Генерация конфигурации | `lg --tags MQTT,POSTGRES,FLINK,REDIS` | Создан `docker-compose.yml` и нужные директории |
| Запуск платформы | `docker compose up -d --build` | Контейнеры запущены |
| Проверка логов | `docker compose logs --tail=100` | Нет циклических ошибок |
| Проверка данных в PostgreSQL | `select * from earth_temp limit 10;` | Есть записи |
| Проверка Redis | `redis-cli --scan` | Есть ключи или кэшированные значения |
| Повторный запуск | `docker compose down && docker compose up -d` | Сервисы восстанавливаются |

### 10.2. Security-тесты

| Тест | Как проверить | Ожидаемый результат |
|---|---|---|
| Проверка открытых портов VM | `ss -tulpn` / `nmap` снаружи | Открыты только разрешённые порты |
| Проверка секретов в Git | `git grep` | Нет реальных секретов |
| Проверка debug-контейнеров | `docker compose ps` | В production нет Kafka UI, MQTT Web и других debug UI |
| Проверка TLS | Подключение к внешним сервисам | Используется TLS |
| Проверка дефолтных паролей | Ревью `.env` и compose | Нет слабых паролей |

### 10.3. Критерии приёмки релиза

Релиз считается готовым, если:

- проект разворачивается на VM по инструкции из этого файла;
- есть воспроизводимый tutorial с ожидаемым результатом;
- есть готовый Docker-образ или release archive;
- выполнены критичные пункты безопасности `SEC-01` — `SEC-05`, `SEC-09`;
- debug UI отсутствуют в production-профиле;
- внешние соединения используют TLS или закрытый сетевой контур.

---

## 11. План работ и сроки

| Дата | Работа | Ответственный | Результат | Статус |
|---|---|---|---|---|
| 30.04.2026 | Подготовить первичный `RELEASE.md` | Разработка | План релиза в репозитории | Готово |
| 30.04.2026 | Заполнить участников релиза и зоны ответственности | Разработка + куратор + DevOps | Таблица участников согласована | Готово |
| 04.05.2026 | Подготовить `.env.example` и обновить `.gitignore` | Разработка | Секреты вынесены из compose | Готово (необходимо опубликовать .env.example, docker-compose.prod.yml) |
| 04.05.2026 | Подготовить `docker-compose.prod.yml` | DevOps + разработка | Production-like профиль без debug UI | Готово (необходимо опубликовать .env.example, docker-compose.prod.yml) |
| 10.05.2026 | Добавить healthcheck/restart policy для сервисов | Разработка + DevOps | Контейнеры диагностируются и перезапускаются | Готово (необходимо опубликовать .env.example, docker-compose.prod.yml) |
| 12.05.2026 | Развернуть release candidate на VM | DevOps | Стенд поднят на VM | Запланировано |
| 12.05.2026 | Выполнить smoke-тесты и security-тесты | Разработка + DevOps | Протокол проверки | Запланировано |
| 13.05.2026 | Согласовать релиз с куратором | Куратор + разработка | Релиз принят к финальной демонстрации | Запланировано |
| Конец мая 2026 | Финальная защита продукта | Вся команда | Демонстрация продукта, документации, сборки, безопасности и кейса использования | Запланировано |

---

## 12. Ссылки

- Статья MVP: https://habr.com/ru/articles/1022970/
- Репозиторий конструктора: https://github.com/IoTDataPlatform/constructor
