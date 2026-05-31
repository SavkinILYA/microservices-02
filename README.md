# Домашнее задание к занятию «Микросервисы: принципы»

---

## Задача 1: API Gateway

### Сравнительная таблица решений

| Решение | Маршрутизация | Аутентификация | HTTPS терминация | Простота настройки | Производительность | Лицензия |
|---|---|---|---|---|---|---|
| **NGINX** | `location/proxy_pass` | `auth_request` | `ssl_certificate` | Высокая | Очень высокая | Open Source |
| **Kong Gateway** | Routes + Services | плагины JWT/OAuth | + | Средняя | Высокая | Open Source / Enterprise |
| **Traefik** | dynamic config | ForwardAuth middleware | Let's Encrypt auto | Высокая | Высокая | Open Source |
| **HAProxy** | ACL rules | Частично (Lua / внешний) | + | Средняя | Очень высокая | Open Source / Enterprise |
| **AWS API Gateway** | + | Cognito / Lambda | + | Средняя | Высокая | Платная (SaaS) |
| **Envoy** | + | `ext_authz` | + | Низкая | Очень высокая | Open Source |

### Выбор: NGINX

**Обоснование:**

- **Маршрутизация** — реализуется через директивы `location` и `proxy_pass`, гибко и без лишних зависимостей.
- **Аутентификация** — директива `auth_request` позволяет вызвать внешний сервис проверки токена перед проксированием запроса.
- **HTTPS терминация** — нативная поддержка SSL/TLS через `ssl_certificate` / `ssl_certificate_key`.
- **Простота** — конфигурационный файл легко читать, версионировать в git и воспроизводить в любой среде без дополнительных баз данных (в отличие от Kong).
- **Производительность** — event-driven архитектура, минимальный оверхед на проксирование.

---

## Задача 2: Брокер сообщений

### Сравнительная таблица решений

| Решение | Кластеризация | Хранение на диске | Скорость | Форматы сообщений | Разграничение прав | Простота эксплуатации |
|---|---|---|---|---|---|---|
| **Kafka** | нативная | commit log | Очень высокая | Любые (binary/JSON/Avro) | ACL по топикам | Средняя |
| **RabbitMQ** | через плагин | durable queues | Высокая | AMQP, JSON, binary | vhost + permissions | Высокая |
| **NATS** | JetStream | JetStream | Очень высокая | Любые (binary) | Частично (accounts) | Высокая |
| **ActiveMQ** | + | + | Средняя | AMQP, MQTT, STOMP | + | Средняя |
| **Redis Streams** | Cluster | + | Высокая | Только строки/binary | Частично (ACL) | Высокая |
| **Pulsar** | нативная | + | Очень высокая | Любые | + | Низкая |

### Выбор: Kafka

**Обоснование по каждому требованию:**

- **Кластеризация** — нативная поддержка через KRaft (с Kafka 3.x без Zookeeper), отказоустойчивость из коробки.
- **Хранение на диске** — сообщения хранятся как commit log, retention настраивается по времени или размеру.
- **Высокая скорость** — sequential disk I/O + batching обеспечивает пропускную способность миллионов сообщений в секунду.
- **Форматы сообщений** — топики принимают любые байты; на практике используют JSON, Avro, Protobuf.
- **Разграничение прав** — ACL на уровне топиков: consumer-группа получает права только на нужный топик.
- **Простота эксплуатации** — широкое распространение в индустрии, богатая документация, много готовых примеров.

---

## Задача 3: API Gateway (необязательная)

### Схема маршрутизации

| Запрос клиента | Проверка токена | Проксируется в |
|---|---|---|
| `POST /v1/register` | Нет (анонимный) | `security POST /v1/user` |
| `POST /v1/token` | Нет (анонимный) | `security POST /v1/token` |
| `GET /v1/user` | Да (`auth_request`) | `security GET /v1/user` |
| `POST /v1/upload` | Да (`auth_request`) | `uploader POST /v1/upload` |
| `GET /v1/user/{image}` | Да (`auth_request`) | `minio GET /images/{image}` |

Проверка токена реализована через директиву NGINX `auth_request`: перед проксированием защищённого маршрута NGINX делает внутренний запрос к `security GET /v1/token/validation`. Если сервис возвращает `200` — запрос пропускается, иначе клиент получает `401`.

### nginx.conf

```nginx
server {
    listen 80;

    # Внутренний endpoint для проверки токена
    location = /auth {
        internal;
        proxy_pass http://security:3000/v1/token/validation;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
        proxy_set_header Authorization $http_authorization;
    }

    # POST /v1/register — анонимный, идёт в security POST /v1/user
    location = /v1/register {
        proxy_pass http://security:3000/v1/user;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # POST /v1/token — анонимный, идёт в security POST /v1/token
    location = /v1/token {
        proxy_pass http://security:3000/v1/token;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # GET /v1/user — с проверкой токена, идёт в security GET /v1/user
    location = /v1/user {
        auth_request /auth;
        proxy_pass http://security:3000/v1/user;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Authorization $http_authorization;
    }

    # POST /v1/upload — с проверкой токена, идёт в uploader POST /v1/upload
    location = /v1/upload {
        auth_request /auth;
        proxy_pass http://uploader:3000/v1/upload;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Authorization $http_authorization;
        client_max_body_size 50m;
    }

    # GET /v1/user/{image} — с проверкой токена, идёт в minio GET /images/{image}
    location ~ ^/v1/user/(.+)$ {
        auth_request /auth;
        proxy_pass http://minio:9000/images/$1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Authorization $http_authorization;
    }

    # Прямой доступ к файлам minio
    location /images/ {
        proxy_pass http://minio:9000/images/;
        proxy_set_header Host $host;
    }
}
```

### docker-compose.yml

```yaml
version: '3.8'

services:

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - security
      - uploader
      - minio
    networks:
      - internal

  security:
    image: ghcr.io/netology-code/devkub-homeworks/security:latest
    expose:
      - "3000"
    networks:
      - internal

  uploader:
    image: ghcr.io/netology-code/devkub-homeworks/uploader:latest
    expose:
      - "3000"
    environment:
      - MINIO_ENDPOINT=minio
      - MINIO_PORT=9000
      - MINIO_ACCESS_KEY=minioadmin
      - MINIO_SECRET_KEY=minioadmin
      - MINIO_BUCKET=images
    depends_on:
      - minio
    networks:
      - internal

  minio:
    image: minio/minio:latest
    command: server /data
    expose:
      - "9000"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    volumes:
      - minio-data:/data
    networks:
      - internal

  minio-init:
    image: minio/mc:latest
    depends_on:
      - minio
    entrypoint: >
      /bin/sh -c "
      sleep 5;
      mc alias set local http://minio:9000 minioadmin minioadmin;
      mc mb --ignore-existing local/images;
      mc anonymous set download local/images;
      exit 0;
      "
    networks:
      - internal

volumes:
  minio-data:

networks:
  internal:
    driver: bridge
```

### Проверка работы

```bash
# 1. Запуск стека
docker-compose up -d

# 2. Получение токена
curl -X POST -H 'Content-Type: application/json' \
  -d '{"login":"bob", "password":"qwe123"}' \
  http://localhost/token

# 3. Загрузка файла
curl -X POST \
  -H 'Authorization: Bearer <token>' \
  -H 'Content-Type: octet/stream' \
  --data-binary @yourfilename.jpg \
  http://localhost/upload

# 4. Получение файла
curl -X GET http://localhost/images/<uuid>.jpg
```
