# PasarGuard Node Ansible Role

Роль Ansible для установки PasarGuard Node на Linux-сервер и автоматической регистрации узла в PasarGuard через Admin API.

Роль не использует внешний `pg-node.sh` для установки. Все действия выполняются Ansible-задачами: установка Docker, генерация API key, генерация сертификата, создание `.env`, создание `docker-compose.yml`, запуск контейнера и запрос в PasarGuard API.

Роль поддерживает два режима развёртывания (`pg_node_deploy_mode`):

- `docker` (по умолчанию) — нода запускается контейнером через `docker compose`;
- `systemd` — нативная сборка бинарника `pasarguard-node` из исходников `PasarGuard/node` и управление через systemd unit `/etc/systemd/system/pg-node.service`, без Docker. Соответствует разделу доки [«Настройка как системный сервис (ручная установка)»](https://docs.pasarguard.org/ru/node/installation/).

Сертификат, `.env`, логика API key, установка geo-файлов (`geoip.dat` / `geosite.dat`) и регистрация в PasarGuard общие для обоих режимов.

Дополнительно роль ставит официальный скрипт управления нодой (`pg-node.sh` из репозитория `PasarGuard/scripts`) в `/usr/local/bin/pg-node`, чтобы дальше управлять нодой командами `pg-node status`, `pg-node logs`, `pg-node restart` и т.д. Скрипт использует те же пути, что и роль, поэтому работает с тем же `docker-compose.yml`, `.env` и сертификатом. Отключается через `pg_node_install_cli: false`.

---

## Что делает роль

Ниже описан режим `docker` (по умолчанию). Про режим `systemd` см. раздел [«Режим systemd (нативная установка как сервис)»](#режим-systemd-нативная-установка-как-сервис).

Роль выполняет следующие действия:

1. Устанавливает системные пакеты:
   - base: `curl`, `openssl`, `ca-certificates`, `unzip`, `tar` (`pg_node_base_packages`);
   - Docker: `docker.io` + `docker-compose-v2` (Ubuntu) либо `docker-ce` + `docker-compose-plugin` (Docker CE), см. `pg_node_docker_install_mode`; в systemd-режиме Docker не ставится.

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

9. Скачивает гео-файлы (`geoip.dat` / `geosite.dat`), выставляет `XRAY_ASSETS_PATH` (отключается `pg_node_install_geodat: false`).

10. Ставит `node-serviced` (+ systemd unit `pg-node-service`) на `API_PORT` для команд из панели, и `yq` для `pg-node core-update` (отключается `pg_node_install_serviced: false` / `pg_node_api_port: ""`).

11. Ставит скрипт управления нодой в `/usr/local/bin/pg-node` (docker-режим; `pg_node_install_cli: false`).

12. Опционально (`pasarguard_register_node: true`) получает access token и **создаёт или обновляет** узел: ищет ноду по имени в `GET /api/nodes` → нет: `POST /api/node`, есть: `PUT /api/node/{id}`.

13. Выводит в лог Ansible имя/адрес/порт/`api_port`/тип подключения/`keep_alive`/режим, API key и PEM-сертификат.

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
- свободные и открытые во внешний файрвол/security group порты ноды:
  - `62050/tcp` — `SERVICE_PORT`, основной канал панель ↔ нода;
  - `62051/tcp` — `API_PORT`, по нему панель шлёт ноде команды (update core, restart) и делает connectivity check. Без открытого `62051` в панели будет `503` на `core_update` и т.п.

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
│   ├── build.yml        # systemd-режим: Go toolchain + build-пакеты
│   ├── facts.yml
│   ├── cert.yml
│   ├── geodat.yml       # geoip.dat / geosite.dat, оба режима
│   ├── compose.yml      # docker-режим
│   ├── service.yml      # systemd-режим: сборка бинарника + unit
│   ├── cli.yml          # docker-режим
│   ├── serviced.yml     # node-serviced (API_PORT), оба режима
│   ├── output.yml
│   └── register.yml
├── templates/
│   ├── docker-compose.yml.j2
│   ├── pg-node.service.j2
│   ├── pg-node-service.service.j2   # unit для node-serviced
│   ├── pg-node.env.j2
│   └── openssl-san.cnf.j2
├── molecule/
│   ├── shared/          # общие converge.yml + verify.yml
│   ├── default/         # docker-in-docker, для CI
│   └── delegated/       # прогон на живом хосте
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

pasarguard_base_url: "https://example.com"
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

После получения access token роль **создаёт или обновляет** узел (см. раздел
[«Создание / обновление ноды»](#создание--обновление-ноды)):

- нет ноды с именем `pg_node_name` → `POST /api/node`;
- есть → `PUT /api/node/{id}`.

Заголовки:

```text
Authorization: Bearer <access_token>
Content-Type: application/json
```

Полный payload (POST). При PUT из него исключаются `pg_node_update_exclude_fields`
(по умолчанию `api_key`, `address`, `server_ca`):

```json
{
  "address": "192.168.1.1",
  "api_key": "valid uuid",
  "connection_type": "grpc",
  "core_config_id": 1,
  "keep_alive": 60,
  "name": "DE node",
  "port": 62050,
  "api_port": 62051,
  "server_ca": "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----",
  "usage_coefficient": 1,
  "data_limit": 0,
  "default_timeout": 10,
  "internal_timeout": 15,
  "proxy_url": null
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
| `core_config_id` | `pg_node_core_config_id` / `pg_node_core_config_name` | ID ядра, либо имя — роль резолвит имя в id через `GET /api/cores/simple?all=true` (имя в приоритете; дубль имени → падение). |
| `keep_alive` | `pg_node_keep_alive` | Keep-alive интервал. |
| `data_limit` | `pg_node_data_limit_gb` | Лимит данных. Задаётся в ГБ, роль переводит в байты. `0` = без лимита. |
| `default_timeout` | `pg_node_default_timeout` | «Тайм-аут по умолчанию», секунды (3..60). |
| `internal_timeout` | `pg_node_internal_timeout` | «Внутренний тайм-аут», секунды (3..60). |
| `proxy_url` | `pg_node_proxy_url` | «URL прокси», напр. `socks5://127.0.0.1:1080`. Отправляется всегда: пусто → `null` (панель очищает поле). |
| `api_port` | `pg_node_api_port` | `API_PORT` / порт `node-serviced`. Пусто → не отправляется. |
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

## Создание / обновление ноды

При `pasarguard_register_node: true` роль получает список нод (`GET /api/nodes`)
и ищет ноду по её имени (`pg_node_name`):

- **не нашлась** → `POST /api/node` — создаёт (полный payload);
- **нашлась** → `PUT /api/node/{id}` — перезаписывает поля из host_vars
  (имя, ядро, порты, `keep_alive`, тайм-ауты, лимит, прокси, `usage_coefficient`),
  **кроме** `pg_node_update_exclude_fields` (по умолчанию `api_key`, `address`,
  `server_ca` — остаются в панели как есть);
- **несколько нод с этим именем** → падение (удали дубли в панели).

То есть повторный прогон приводит поля ноды к тому, что в git/host_vars.
Смотри в логе задачу **`Print PasarGuard result`**: `Status: updated,
node_id=86, http_status=200`.

### Ядро по имени или id

```yaml
pg_node_core_config_id: 58            # числовой id
# или
pg_node_core_config_name: "core-name" # имя -> роль резолвит id через GET /api/cores/simple?all=true
```

Имя в приоритете. Дубль имени (или не найдено) → падение с подсказкой.

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

## Режим systemd (нативная установка как сервис)

Включается так:

```yaml
pg_node_deploy_mode: systemd
```

В этом режиме роль:

1. Ставит base-пакеты и build-пакеты (`git`, `make`, `gcc`, `pkg-config`). Docker не ставится.
2. Ставит Go toolchain в `/usr/local/go`, если в системе нет Go нужной версии (`PasarGuard/node` требует Go >= 1.25). Если подходящий Go уже есть, шаг пропускается.
3. Клонирует `https://github.com/PasarGuard/node.git` в `/opt/pg-node/node`.
4. Собирает бинарник: `make deps`, `make`. Результат копируется в `/opt/pg-node/pasarguard-node`.
5. Ставит Xray core через `PasarGuard/scripts/install_core.sh` (в `/usr/local/bin/xray`). Отключается через `pg_node_install_xray: false`.
6. Генерирует тот же сертификат, что и в docker-режиме (`/var/lib/pg-node/certs/`).
7. Рендерит `/opt/pg-node/.env` (тот же шаблон, что и для контейнера).
8. Создаёт unit `/etc/systemd/system/pg-node.service` c `WorkingDirectory=/opt/pg-node` и `ExecStart=/opt/pg-node/pasarguard-node`. Бинарник читает `.env` из рабочего каталога.
9. Делает `daemon-reload`, `enable`, `start` и проверяет, что сервис `active`.
10. Опционально регистрирует ноду в PasarGuard (та же логика, что и в docker-режиме).

Управление после установки:

```bash
systemctl status pg-node
systemctl restart pg-node
journalctl -u pg-node -f
```

Пересборка бинарника принудительно:

```bash
ansible-playbook -i inventories/pg-node.ini playbooks/install_pg_node.yml \
  -e pg_node_deploy_mode=systemd -e pg_node_build_force=true
```

По умолчанию бинарник пересобирается только если его нет или обновились исходники (`pg_node_src_version`).

Замечания:

- Скрипт управления `pg-node` (`tasks/cli.yml`) в этом режиме не ставится — он обёртка над `docker compose`.
- Сборка требует интернет-доступа к `go.dev`, `proxy.golang.org` и GitHub, а также ~2 GB RAM.
- Открыть порты ноды: `sudo ufw allow 62050/tcp && sudo ufw allow 62051/tcp` (роль firewall не трогает).

### Переменные режима systemd

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `pg_node_deploy_mode` | `docker` | `docker` или `systemd`. |
| `pg_node_service_name` | `pg-node` | Имя systemd unit (без `.service`). |
| `pg_node_service_user` | `root` | Пользователь сервиса. |
| `pg_node_service_group` | `root` | Группа сервиса. |
| `pg_node_src_repo` | `https://github.com/PasarGuard/node.git` | Репозиторий исходников. |
| `pg_node_src_version` | `main` | Ветка/тег/коммит исходников. |
| `pg_node_src_dir` | `/opt/pg-node/node` | Каталог исходников и сборки. |
| `pg_node_bin_path` | `/opt/pg-node/pasarguard-node` | Куда ставится собранный бинарник. |
| `pg_node_build_force` | `false` | Принудительно пересобрать бинарник. |
| `pg_node_install_xray` | `true` | Ставить Xray core. |
| `pg_node_xray_executable_path` | `/usr/local/bin/xray` | Путь к бинарнику xray в unit. |
| `pg_node_xray_assets_path` | `/usr/local/share/xray` | Путь к ассетам xray в unit. |
| `pg_node_build_packages` | `[git, make, gcc, pkg-config]` | Пакеты для сборки. |
| `pg_node_go_install` | `true` | Ставить Go toolchain при необходимости. |
| `pg_node_go_version` | `1.25.1` | Версия Go для установки. |
| `pg_node_go_min_version` | `1.25.0` | Минимальная приемлемая версия Go. |
| `pg_node_go_dir` | `/usr/local/go` | Куда ставится Go. |

---

## Geodata (geoip.dat / geosite.dat)

Роль сама скачивает гео-файлы Xray и кладёт их в `pg_node_geodat_dir`
(по умолчанию `/var/lib/pg-node/xray-assets`). Работает в обоих режимах:

- **systemd** — файлы лежат в `pg_node_geodat_dir`, туда же указывает
  `XRAY_ASSETS_PATH` в unit и в `.env`.
- **docker** — файлы лежат на хосте в `pg_node_geodat_dir`. Если каталог внутри
  `pg_node_data_dir` (по умолчанию так), он уже проброшен в контейнер; иначе роль
  добавляет отдельный `:ro` volume. В `.env` выставляется `XRAY_ASSETS_PATH`,
  что переопределяет гео-файлы, вшитые в образ `pasarguard/node`.

При каждом запуске роль скачивает свежие файлы. Нода перезапускается только если
содержимое файла реально изменилось.

По умолчанию берутся upstream-сборки v2fly (те же данные, что XTLS кладёт в
релизы Xray-core):

```yaml
pg_node_geodat_files:
  - name: geoip.dat
    url: "https://github.com/v2fly/geoip/releases/latest/download/geoip.dat"
  - name: geosite.dat
    url: "https://github.com/v2fly/domain-list-community/releases/latest/download/dlc.dat"
```

Пример с альтернативными правилами (Loyalsoldier + русские правила):

```yaml
pg_node_geodat_files:
  - name: geoip.dat
    url: "https://github.com/runetfreedom/russia-v2ray-rules-dat/releases/latest/download/geoip.dat"
  - name: geosite.dat
    url: "https://github.com/runetfreedom/russia-v2ray-rules-dat/releases/latest/download/geosite.dat"
  - name: geoip-loyalsoldier.dat
    url: "https://github.com/Loyalsoldier/v2ray-rules-dat/releases/latest/download/geoip.dat"
```

Можно задать проверку контрольной суммы на файл:

```yaml
pg_node_geodat_files:
  - name: geoip.dat
    url: "https://github.com/v2fly/geoip/releases/latest/download/geoip.dat"
    checksum: "sha256:https://github.com/v2fly/geoip/releases/latest/download/geoip.dat.sha256sum"
```

Отключить установку гео-файлов полностью:

```yaml
pg_node_install_geodat: false
```

В docker-режиме это вернёт ноду на гео-файлы из образа, в systemd-режиме останутся
файлы, которые положил `install_core.sh` в `/usr/local/share/xray`.

### Переменные geodata

| Переменная | Значение по умолчанию | Назначение |
|---|---|---|
| `pg_node_install_geodat` | `true` | Ставить/обновлять гео-файлы. |
| `pg_node_geodat_dir` | `/var/lib/pg-node/xray-assets` | Каталог с гео-файлами. |
| `pg_node_geodat_set_assets_path` | `true` | Выставлять `XRAY_ASSETS_PATH` в `.env` и unit. |
| `pg_node_geodat_files` | v2fly geoip + geosite | Список `{name, url, checksum?}`. |

---

## `.env` ноды

Роль создаёт файл:

```text
/opt/pg-node/.env
```

Пример:

```env
SERVICE_PORT = 62050
API_PORT = 62051
NODE_HOST = "0.0.0.0"

SSL_CERT_FILE = /var/lib/pg-node/certs/ssl_cert.pem
SSL_KEY_FILE = /var/lib/pg-node/certs/ssl_key.pem

API_KEY = 6f3b9d2a-8e4c-4c91-9b72-1a0e9f6d4c2b

SERVICE_PROTOCOL = grpc
```

`API_PORT` роль пишет из `pg_node_api_port` (по умолчанию `62051`). Пусто
(`pg_node_api_port: ""`) — строка `API_PORT` в `.env` не пишется и `node-serviced`
не ставится.

При включённой регистрации (`pasarguard_register_node: true`) роль отправляет
`api_port` и в payload создания/обновления ноды.

---

## node-serviced (management API на `API_PORT`)

Сама нода (`pasarguard/node` в docker-режиме и нативный бинарник в systemd-режиме)
читает **только `SERVICE_PORT`** и открывает один порт. Команды из панели
(update core, restart) и connectivity check идут на **`API_PORT` (62051)**, а его
обслуживает **отдельный бинарник `node-serviced`** — это и есть «сервис», про
который спрашивает `pg-node.sh service-install`.

Роль ставит его в обоих режимах (`pg_node_install_serviced: true`, если
`pg_node_api_port` не пуст):

1. скачивает `node-serviced` из релизов `PasarGuard/node-serviced` под нужную
   архитектуру → `/usr/local/bin/{{ pg_node_app_name }}-service`;
2. рендерит systemd unit `/etc/systemd/system/{{ pg_node_app_name }}-service.service`
   (`ExecStart` = бинарник, `Environment=ENV_FILE=/opt/pg-node/.env`);
3. `daemon-reload`, `enable --now`, проверяет `systemctl is-active`.

Управление:
```bash
systemctl status pg-node-service
journalctl -u pg-node-service -f
```

Без `node-serviced` (или при закрытом во внешний файрвол `62051`) панель отдаёт
`503` на `core_update`. `node-serviced` при `core_update` дёргает
`pg-node core-update`, которому нужен `yq` (mikefarah) и `unzip` — роль ставит их
сама (`pg_node_install_yq: true`, `unzip`/`tar` в base-пакетах). Без `yq` в логах
`journalctl -u pg-node-service` будет `yq: command not found` (exit 127), а панель
покажет `404`.

| Переменная | По умолчанию | Назначение |
|---|---|---|
| `pg_node_install_serviced` | `true` | Ставить `node-serviced`. |
| `pg_node_serviced_repo` | `PasarGuard/node-serviced` | Репозиторий релизов. |
| `pg_node_serviced_version` | `latest` | `latest` или точный тег (`v0.1.4`). |
| `pg_node_serviced_service_name` | `{{ pg_node_app_name }}-service` | Имя systemd unit. |
| `pg_node_serviced_bin_path` | `/usr/local/bin/{{ pg_node_app_name }}-service` | Путь к бинарнику. |
| `pg_node_serviced_update` | `true` | Перекачивать бинарник каждый запуск. |

---

## Основные переменные роли

| Переменная | Значение по умолчанию | Назначение |
|---|---:|---|
| `pg_node_deploy_mode` | `docker` | Режим развёртывания: `docker` или `systemd`. |
| `pg_node_install_geodat` | `true` | Скачивать и обновлять `geoip.dat` / `geosite.dat`. |
| `pg_node_geodat_dir` | `/var/lib/pg-node/xray-assets` | Каталог гео-файлов (`XRAY_ASSETS_PATH`). |
| `pg_node_app_name` | `pg-node` | Имя compose project. |
| `pg_node_app_dir` | `/opt/pg-node` | Директория compose и `.env`. |
| `pg_node_data_dir` | `/var/lib/pg-node` | Директория данных ноды. |
| `pg_node_certs_dir` | `/var/lib/pg-node/certs` | Директория сертификатов. |
| `pg_node_container_name` | `node` | Имя контейнера. |
| `pg_node_image` | `pasarguard/node:latest` | Docker image ноды. |
| `pg_node_service_port` | `62050` | Порт ноды (`SERVICE_PORT`). |
| `pg_node_api_port` | `62051` | Порт `node-serviced` (`API_PORT`) для команд из панели. Пусто — не ставить. |
| `pg_node_install_serviced` | `true` | Ставить `node-serviced` (management API на `API_PORT`). |
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
| `pg_node_core_config_name` | пусто | Имя ядра вместо id (резолвится через `GET /api/cores/simple?all=true`). |
| `pg_node_keep_alive` | `60` | `keep_alive` для API регистрации. |
| `pg_node_usage_coefficient` | `1` | `usage_coefficient` для API регистрации. |
| `pg_node_data_limit_gb` | `0` | Лимит данных ноды в ГБ (0 = без лимита). |
| `pg_node_default_timeout` | `10` | «Тайм-аут по умолчанию», сек (3..60). |
| `pg_node_internal_timeout` | `15` | «Внутренний тайм-аут», сек (3..60). |
| `pg_node_proxy_url` | пусто | «URL прокси» ноды; пусто → `null`. |
| `pg_node_update_exclude_fields` | `[api_key, address, server_ca]` | Поля, не отправляемые при PUT-обновлении. |
| `pg_node_install_yq` | `true` | Ставить `yq` (mikefarah) — нужен `pg-node core-update`. |
| `pg_node_register_address` | пусто | Адрес ноды для PasarGuard. |
| `pasarguard_register_node` | `false` | Включить регистрацию (create/update) ноды. |
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
- гео-файлы перекачиваются, нода перезапускается только при реальном изменении содержимого;
- если `pasarguard_register_node: true` — нода в панели **приводится к host_vars**: найдена по имени → `PUT`, не найдена → `POST` (поля `pg_node_update_exclude_fields` при PUT не трогаются).

---

## Тесты (Molecule)

В `molecule/` два сценария с общими `converge.yml` / `verify.yml` (`molecule/shared/`):

| Сценарий | Драйвер | Где выполняется |
|---|---|---|
| `default` | docker | одноразовый привилегированный контейнер `geerlingguy/docker-ubuntu2404-ansible` (docker-in-docker), для CI |
| `delegated` | default | реальный хост (по умолчанию `deploy@192.168.0.2`) |

Оба покрывают `pg_node_deploy_mode=docker` и `pg_node_deploy_mode=systemd`
(через `PG_NODE_DEPLOY_MODE=systemd`).

```bash
pip install "molecule>=6" "molecule-plugins[docker]" ansible-lint

# контейнерный, полный цикл
molecule test -s default

# на живом хосте (роль применяется по-настоящему, автоснос не делается)
molecule converge    -s delegated
molecule idempotence -s delegated
molecule verify      -s delegated
molecule destroy     -s delegated          # снести ноду с хоста

# другой хост:
PG_NODE_TEST_HOST=10.0.0.5 PG_NODE_TEST_USER=root molecule converge -s delegated
```

Подробнее — `molecule/README.md`.

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
pasarguard_base_url: "https://example.com"
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

После установки на сервере будут созданы (docker-режим, всё включено):

```text
/opt/pg-node/docker-compose.yml
/opt/pg-node/.env
/var/lib/pg-node/certs/ssl_cert.pem
/var/lib/pg-node/certs/ssl_key.pem
/var/lib/pg-node/certs/openssl-san.cnf
/var/lib/pg-node/xray-assets/geoip.dat
/var/lib/pg-node/xray-assets/geosite.dat
/usr/local/bin/pg-node                      # CLI (pg_node_install_cli)
/usr/local/bin/pg-node-service              # node-serviced (pg_node_install_serviced)
/usr/local/bin/yq                           # pg_node_install_yq
/etc/systemd/system/pg-node-service.service
```

systemd-режим — вместо контейнера: `/opt/pg-node/pasarguard-node`,
`/opt/pg-node/node/` (исходники), `/etc/systemd/system/pg-node.service`.

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
