# PasarGuard Node Ansible Role

Роль Ansible для установки PasarGuard Node на Linux-сервер и автоматической регистрации узла в PasarGuard через Admin API.

Роль не использует внешний `pg-node.sh`. Все действия выполняются Ansible-задачами: установка Docker, генерация API key, генерация сертификата, создание `.env`, создание `docker-compose.yml`, запуск контейнера и запрос в PasarGuard API.

---

## Что делает роль

Роль выполняет следующие действия:

1. Устанавливает системные пакеты:
   - `curl`;
   - `openssl`;
   - `ca-certificates`;
   - `docker.io`;
   - `docker-compose-v2`.

2. Создаёт директории:

   ```text
   /opt/pg-node
   /var/lib/pg-node
   /var/lib/pg-node/certs
   ```

3. Генерирует или сохраняет существующий `API_KEY`.

4. Определяет белый IPv4 сервера.

5. Генерирует self-signed TLS-сертификат для ноды:

   ```text
   /var/lib/pg-node/certs/ssl_cert.pem
   /var/lib/pg-node/certs/ssl_key.pem
   ```

6. Добавляет в SAN сертификата:
   - `DNS:localhost`;
   - домены из `pg_node_cert_dns_names`;
   - `IP:127.0.0.1`;
   - белый IP сервера;
   - дополнительные IP из `pg_node_cert_extra_ip_sans`.

7. Создаёт файлы:

   ```text
   /opt/pg-node/docker-compose.yml
   /opt/pg-node/.env
   ```

8. Запускает контейнер:

   ```bash
   docker compose -f /opt/pg-node/docker-compose.yml -p pg-node up -d
   ```

9. Опционально получает access token в PasarGuard и отправляет запрос на создание узла.

10. Выводит в лог Ansible:
    - имя ноды;
    - адрес;
    - порт;
    - тип подключения;
    - API key;
    - PEM-сертификат.

---

## Требования

На управляющей машине:

- Ansible;
- SSH-доступ до сервера;
- пользователь с `sudo` или прямой `root`.

На целевом сервере:

- Debian/Ubuntu;
- доступ в интернет для установки пакетов и скачивания Docker image;
- доступ к PasarGuard API, если включена регистрация ноды;
- свободный порт ноды, по умолчанию `62050/tcp`.

---

## Структура роли

Корневая директория роли должна выглядеть так:

```text
PG-NODE/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── packages.yml
│   ├── facts.yml
│   ├── cert.yml
│   ├── compose.yml
│   ├── output.yml
│   └── register.yml
├── templates/
│   ├── docker-compose.yml.j2
│   ├── pg-node.env.j2
│   └── openssl-san.cnf.j2
└── README.md
```

Важно: директория должна называться именно `tasks`, а не `task`. Ansible ищет точку входа роли в файле:

```text
tasks/main.yml
```

---

## Быстрый старт

Пример структуры Ansible-проекта:

```text
ansible/
├── inventories/
│   └── pg-node.ini
├── group_vars/
│   └── pg_nodes.yml
├── playbooks/
│   └── install_pg_node.yml
└── roles/
    └── PG-NODE/
```

Пример inventory:

```ini
[pg_nodes]
de-node-1 ansible_host=192.168.1.1 ansible_user=root
```

Пример playbook:

```yaml
---
- name: Install PasarGuard node
  hosts: pg_nodes
  become: true
  gather_facts: true

  roles:
    - role: PG-NODE
```

Запуск:

```bash
ansible-playbook -i inventories/pg-node.ini playbooks/install_pg_node.yml
```

---

## Минимальный пример переменных

Файл:

```text
group_vars/pg_nodes.yml
```

```yaml
---
pg_node_name: "DE node"

pg_node_cert_dns_names:
  - "de-node.example.com"

pg_node_service_port: 62050
pg_node_connection_type: "grpc"

pasarguard_register_node: false

pg_node_print_sensitive: true
```

В этом режиме роль только установит ноду, поднимет контейнер и выведет в лог API key и сертификат. Ноду можно будет зарегистрировать в панели вручную.

---

## Пример с автоматической регистрацией в PasarGuard

```yaml
---
pg_node_name: "DE node"

pg_node_cert_dns_names:
  - "de-node.example.com"

pg_node_service_port: 62050
pg_node_connection_type: "grpc"

pg_node_core_config_id: 1
pg_node_keep_alive: 60
pg_node_usage_coefficient: 1

# Если пусто, роль отправит белый IP сервера.
# Можно указать IP или домен.
pg_node_register_address: ""

pasarguard_register_node: true

pasarguard_base_url: "https://testpg.example.org"
pasarguard_admin_username: ""
pasarguard_admin_password: ""
pasarguard_client_secret: ""
pasarguard_scope: "Authorized"

pasarguard_auth_token_path: "/api/admin/token"
pasarguard_node_register_path: "/api/node"

pasarguard_validate_certs: true

pg_node_print_sensitive: true
```

---

## Авторизация в PasarGuard

Роль использует рабочую схему авторизации PasarGuard Admin API.

Запрос токена:

```text
POST {{ pasarguard_base_url }}/api/admin/token
```

Авторизация выполняется через HTTP Basic Auth:

```text
username = PASARGUARD_ADMIN_USERNAME
password = PASARGUARD_CLIENT_SECRET
```

Тело запроса отправляется как `application/x-www-form-urlencoded`:

```text
grant_type=password
username=<PASARGUARD_ADMIN_USERNAME>
password=<PASARGUARD_ADMIN_PASSWORD>
scope=<PASARGUARD_SCOPE>
```

Эквивалент на Python:

```python
requests.post(
    f"{base}/api/admin/token",
    auth=(admin_username, client_secret),
    headers={
        "Content-Type": "application/x-www-form-urlencoded",
    },
    data={
        "grant_type": "password",
        "username": admin_username,
        "password": admin_password,
        "scope": scope,
    },
)
```

Важно: `PASARGUARD_CLIENT_SECRET` не передаётся в form body. Он используется как пароль в HTTP Basic Auth.

---

## Регистрация ноды в PasarGuard

После получения access token роль отправляет запрос на создание узла.

Endpoint по умолчанию:

```text
POST {{ pasarguard_base_url }}/api/node
```

Заголовки:

```text
Authorization: Bearer <access_token>
Content-Type: application/json
```

Payload:

```json
{
  "address": "192.168.1.1",
  "api_key": "valid uuid",
  "connection_type": "grpc",
  "core_config_id": 1,
  "keep_alive": 60,
  "name": "DE node",
  "port": 62050,
  "server_ca": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
  "usage_coefficient": 1
}
```

---

## Как переопределять поля payload

Все поля payload управляются переменными роли.

| Поле API | Переменная роли | Описание |
|---|---|---|
| `address` | `pg_node_register_address` | Адрес ноды в PasarGuard. Если пусто — белый IP сервера. |
| `api_key` | `pg_node_api_key_effective` | Итоговый API key. Берётся из `pg_node_api_key`, существующего `.env` или генерируется. |
| `connection_type` | `pg_node_connection_type` | `grpc` или `rest`. |
| `core_config_id` | `pg_node_core_config_id` | ID core config в PasarGuard. |
| `keep_alive` | `pg_node_keep_alive` | Keep-alive интервал. |
| `name` | `pg_node_name` | Имя ноды в PasarGuard. |
| `port` | `pg_node_service_port` | Порт ноды. |
| `server_ca` | `pg_node_cert_pem` | Сгенерированный сертификат. |
| `usage_coefficient` | `pg_node_usage_coefficient` | Коэффициент использования. |

Для ручной подмены сертификата в payload можно использовать:

```yaml
pg_node_server_ca_override: |
  -----BEGIN CERTIFICATE-----
  ...
  -----END CERTIFICATE-----
```

Для точечного переопределения любого поля итогового payload:

```yaml
pg_node_register_payload_override:
  address: "192.168.1.1"
  name: "DE node"
  keep_alive: 60
```

Не рекомендуется переопределять `api_key` через `pg_node_register_payload_override`, потому что ключ в `.env` ноды и ключ в PasarGuard должны совпадать. Для API key используй:

```yaml
pg_node_api_key: "6f3b9d2a-8e4c-4c91-9b72-1a0e9f6d4c2b"
```

---

## Логика API key

Роль определяет API key в таком порядке:

1. Если задана переменная `pg_node_api_key`, используется она.
2. Если переменная пустая, но в `/opt/pg-node/.env` уже есть валидный `API_KEY`, используется существующий.
3. Если ключа нет, роль генерирует новый UUID через:

   ```bash
   cat /proc/sys/kernel/random/uuid
   ```

Итоговый ключ записывается в:

```text
/opt/pg-node/.env
```

---

## Логика сертификата

Сертификат генерируется в файлы:

```text
/var/lib/pg-node/certs/ssl_cert.pem
/var/lib/pg-node/certs/ssl_key.pem
```

Команда OpenSSL использует:

```text
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:P-256
```

Срок действия по умолчанию:

```yaml
pg_node_cert_days: 3650
```

SAN формируется из:

```yaml
pg_node_cert_dns_names:
  - "de-node.example.com"

pg_node_cert_extra_ip_sans: []
```

Белый IP сервера добавляется автоматически.

Если нужно задать белый IP вручную:

```yaml
pg_node_public_ip: "1.2.3.4"
```

Если нужно пересоздать сертификат принудительно:

```bash
ansible-playbook -i inventories/pg-node.ini playbooks/install_pg_node.yml \
  -e pg_node_cert_force=true
```

---

## Docker Compose

Роль создаёт файл:

```text
/opt/pg-node/docker-compose.yml
```

Содержимое:

```yaml
services:
  node:
    container_name: node
    image: pasarguard/node:latest
    restart: always
    network_mode: host
    privileged: true
    cap_add:
      - NET_ADMIN
    env_file: .env
    volumes:
      - /var/lib/pg-node:/var/lib/pg-node
      - /etc/letsencrypt:/etc/letsencrypt:ro
```

Контейнер запускается так:

```bash
cd /opt/pg-node
docker compose up -d
```

---

## `.env` ноды

Роль создаёт файл:

```text
/opt/pg-node/.env
```

Пример:

```env
SERVICE_PORT = 62050
NODE_HOST = "0.0.0.0"

SSL_CERT_FILE = /var/lib/pg-node/certs/ssl_cert.pem
SSL_KEY_FILE = /var/lib/pg-node/certs/ssl_key.pem

API_KEY = 6f3b9d2a-8e4c-4c91-9b72-1a0e9f6d4c2b

SERVICE_PROTOCOL = grpc
```

---

## Основные переменные роли

| Переменная | Значение по умолчанию | Назначение |
|---|---:|---|
| `pg_node_app_name` | `pg-node` | Имя compose project. |
| `pg_node_app_dir` | `/opt/pg-node` | Директория compose и `.env`. |
| `pg_node_data_dir` | `/var/lib/pg-node` | Директория данных ноды. |
| `pg_node_certs_dir` | `/var/lib/pg-node/certs` | Директория сертификатов. |
| `pg_node_container_name` | `node` | Имя контейнера. |
| `pg_node_image` | `pasarguard/node:latest` | Docker image ноды. |
| `pg_node_service_port` | `62050` | Порт ноды. |
| `pg_node_host` | `0.0.0.0` | Адрес bind внутри ноды. |
| `pg_node_connection_type` | `grpc` | Тип соединения: `grpc` или `rest`. |
| `pg_node_api_key` | пусто | Явно заданный API key. |
| `pg_node_public_ip` | пусто | Явно заданный белый IP. |
| `pg_node_cert_dns_names` | `[]` | DNS SAN сертификата. |
| `pg_node_cert_extra_ip_sans` | `[]` | Дополнительные IP SAN. |
| `pg_node_cert_days` | `3650` | Срок действия сертификата. |
| `pg_node_cert_force` | `false` | Принудительно пересоздать сертификат. |
| `pg_node_print_sensitive` | `true` | Печатать API key и сертификат в лог. |
| `pg_node_core_config_id` | `1` | `core_config_id` для API регистрации. |
| `pg_node_keep_alive` | `60` | `keep_alive` для API регистрации. |
| `pg_node_usage_coefficient` | `1` | `usage_coefficient` для API регистрации. |
| `pg_node_register_address` | пусто | Адрес ноды для PasarGuard. |
| `pasarguard_register_node` | `false` | Включить регистрацию ноды. |
| `pasarguard_base_url` | пусто | Base URL PasarGuard. |
| `pasarguard_auth_token_path` | `/api/admin/token` | Endpoint получения токена. |
| `pasarguard_node_register_path` | `/api/node` | Endpoint создания ноды. |
| `pasarguard_validate_certs` | `true` | Проверять TLS-сертификат PasarGuard API. |
| `pasarguard_debug_http` | `false` | Показывать подробные ответы API. |

---

## Хранение секретов

Не храни `pasarguard_admin_password` и `pasarguard_client_secret` открытым текстом в Git.

Рекомендуемый вариант — Ansible Vault.

Пример отдельного файла:

```text
group_vars/pg_nodes/vault.yml
```

```yaml
pasarguard_admin_username: "admin@example.com"
pasarguard_admin_password: "strong-password"
pasarguard_client_secret: "secret"
```

Шифрование:

```bash
ansible-vault encrypt group_vars/pg_nodes/vault.yml
```

Запуск:

```bash
ansible-playbook -i inventories/pg-node.ini playbooks/install_pg_node.yml --ask-vault-pass
```

Или через vault password file:

```bash
ansible-playbook -i inventories/pg-node.ini playbooks/install_pg_node.yml \
  --vault-password-file ./vault-pass.txt
```

---

## Проверка после установки

Проверить контейнер:

```bash
docker ps --filter name=node
```

Проверить compose:

```bash
cd /opt/pg-node
docker compose ps
```

Посмотреть логи:

```bash
cd /opt/pg-node
docker compose logs -f
```

Проверить `.env`:

```bash
cat /opt/pg-node/.env
```

Проверить сертификат:

```bash
openssl x509 -in /var/lib/pg-node/certs/ssl_cert.pem -noout -subject -issuer -dates
openssl x509 -in /var/lib/pg-node/certs/ssl_cert.pem -noout -text | grep -A2 "Subject Alternative Name"
```

Вывести сертификат для ручной регистрации:

```bash
cat /var/lib/pg-node/certs/ssl_cert.pem
```

Проверить порт:

```bash
ss -tulpen | grep 62050
```

---

## Повторный запуск

Роль рассчитана на повторный запуск.

При повторном запуске:

- существующий API key сохраняется;
- директории не удаляются;
- сертификат не пересоздаётся без причины;
- `.env` и `docker-compose.yml` обновляются из шаблонов;
- контейнер перезапускается только при изменениях;
- регистрация в PasarGuard выполняется, если включено `pasarguard_register_node`.

---

## Отключение вывода чувствительных данных

После первичной установки лучше отключить вывод API key и сертификата:

```yaml
pg_node_print_sensitive: false
```

---

## Ручная регистрация без PasarGuard API

Отключи автоматическую регистрацию:

```yaml
pasarguard_register_node: false
pg_node_print_sensitive: true
```

После запуска роль выведет данные для ручного добавления:

```text
Name
Address
Port
Connection type
API_KEY
CERTIFICATE
```

Эти значения можно вставить в PasarGuard Panel вручную.

---

## Типовые проблемы

### Ansible не находит роль

Проверь путь:

```text
roles/PG-NODE/tasks/main.yml
```

Папка должна называться `tasks`.

---

### Не найден пакет docker-compose-v2

На некоторых системах пакет называется `docker-compose-plugin`.

Переопредели список пакетов:

```yaml
pg_node_required_packages:
  - curl
  - openssl
  - ca-certificates
  - docker.io
  - docker-compose-plugin
```

---

### Не определяется белый IP

Задай IP вручную:

```yaml
pg_node_public_ip: "1.2.3.4"
```

---

### Нужно зарегистрировать ноду по домену

Укажи домен как адрес регистрации:

```yaml
pg_node_register_address: "de-node.example.com"
```

Добавь этот же домен в сертификат:

```yaml
pg_node_cert_dns_names:
  - "de-node.example.com"
```

---

### Нужно пересоздать сертификат

```bash
ansible-playbook -i inventories/pg-node.ini playbooks/install_pg_node.yml \
  -e pg_node_cert_force=true
```

---

### Ошибка авторизации PasarGuard

Проверь переменные:

```yaml
pasarguard_base_url: "https://testpg.example.org"
pasarguard_admin_username: ""
pasarguard_admin_password: ""
pasarguard_client_secret: ""
pasarguard_scope: "Authorized"
pasarguard_auth_token_path: "/api/admin/token"
```

Также проверь, что PasarGuard API принимает Basic Auth:

```text
username = PASARGUARD_ADMIN_USERNAME
password = PASARGUARD_CLIENT_SECRET
```

---

### Ошибка 404 при регистрации ноды

Проверь путь создания узла:

```yaml
pasarguard_node_register_path: "/api/node"
```

Если в backend путь другой, нужно поменять только эту переменную.

---

## Итоговые файлы на сервере

После установки на сервере будут созданы:

```text
/opt/pg-node/docker-compose.yml
/opt/pg-node/.env
/var/lib/pg-node/certs/ssl_cert.pem
/var/lib/pg-node/certs/ssl_key.pem
/var/lib/pg-node/certs/openssl-san.cnf
```

Контейнер:

```text
node
```

Docker image:

```text
pasarguard/node:latest
```

Порт по умолчанию:

```text
62050
```

Тип подключения по умолчанию:

```text
grpc
```
