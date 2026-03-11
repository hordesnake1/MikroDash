# MikroDash: установка с нуля на R5S + команды для MikroTik hEX S

## Назначение

Этот документ описывает полный путь установки `MikroDash`:

- `R5S` выступает Docker-хостом для контейнера
- `MikroTik hEX S` (`RB760iGS`) выступает источником данных через RouterOS API
- панель открывается с `R5S`, а не с самого MikroTik

Подходит для сценария, где `hEX S` не поддерживает контейнеры или контейнеры на роутере использовать не хочется.

## Схема

```text
Браузер оператора -> R5S:3081 -> MikroDash -> RouterOS API (8728) -> hEX S
```

## Предварительные данные

Перед началом подготовь:

- IP `R5S` в LAN: `192.168.1.72`
- IP `hEX S` в LAN: `192.168.1.1`
- WAN-интерфейс для графиков MikroDash: например `pppoe-out1`
- логин/пароль для HTTP Basic Auth в MikroDash
- отдельный пароль для read-only API пользователя на MikroTik

## Часть 1. Установка с нуля на R5S

### 1. Установить Docker и Compose

На `R5S` должен быть установлен Docker Engine и plugin `docker compose`.

Проверка:

```bash
docker --version
docker compose version
```

### 2. Клонировать ваш форк

```bash
mkdir -p /opt
cd /opt
git clone git@github.com:hordesnake1/MikroDash.git mikrodash
cd /opt/mikrodash
git checkout main
```

### 3. Подготовить production env

```bash
cd /opt/mikrodash
cp deploy/r5s/.env.example deploy/r5s/.env
chmod 600 deploy/r5s/.env
```

Открой файл:

```bash
vi deploy/r5s/.env
```

Заполни минимум такие значения:

```env
BASIC_AUTH_USER=admin
BASIC_AUTH_PASS=CHANGE_ME_STRONG
ROUTER_HOST=192.168.1.1
ROUTER_PORT=8728
ROUTER_TLS=false
ROUTER_USER=mikrodash
ROUTER_PASS=CHANGE_ME_ROUTER_PASSWORD
DEFAULT_IF=pppoe-out1
```

Примечания:

- `ROUTER_HOST` — IP `hEX S`
- `ROUTER_USER` / `ROUTER_PASS` — read-only API user, который будет создан на MikroTik ниже
- `DEFAULT_IF` — интерфейс WAN, который нужен на главном графике
- `BASIC_AUTH_PASS` должен быть не тестовым

### 4. Собрать и запустить контейнер

```bash
cd /opt/mikrodash/deploy/r5s
docker compose up -d --build
```

### 5. Проверить старт контейнера

```bash
docker compose ps
docker compose logs --tail=100 mikrodash
curl -i http://127.0.0.1:3081/healthz
```

Ожидаемое поведение:

- контейнер в состоянии `running`
- после успешного подключения к RouterOS `/healthz` возвращает `HTTP/1.1 200 OK`
- в логах нет бесконечного reconnect loop

### 6. Открыть панель

В браузере:

```text
http://<R5S_IP>:3081
```

После этого должен появиться запрос Basic Auth.

## Часть 2. Команды для MikroTik hEX S

Ниже команды подобраны под предоставленный конфиг `RouterOS 7.20.7` на `RB760iGS`.

### Важно перед применением

- вносить из LAN
- не терять fallback-доступ через WinBox/MAC-WinBox
- сначала добавить `allow`, потом `drop`
- в твоем конфиге LAN не режется общим `input drop`, поэтому сервис `api` нужно ограничить отдельно

### 1. Backup и быстрая проверка

```routeros
/export file=pre-mikrodash-api
/ip service print detail
/ip firewall filter print detail
```

### 2. Создать read-only группу и пользователя для MikroDash

```routeros
/user group add name=mikrodash policy=read,api,!local,!telnet,!ssh,!ftp,!reboot,!write,!policy,!test,!winbox,!web,!sniff,!sensitive,!romon,!rest-api
/user add name=mikrodash group=mikrodash password=CHANGE_ME_ROUTER_PASSWORD
```

### 3. Включить RouterOS API только для R5S

```routeros
/ip service set api disabled=no port=8728 address=192.168.1.72/32
```

### 4. Добавить address-list для хоста R5S

```routeros
/ip firewall address-list add list=mikrodash_hosts address=192.168.1.72 comment="R5S MikroDash host"
```

### 5. Разрешить API с R5S и запретить остальным

```routeros
/ip firewall filter add chain=input action=accept protocol=tcp dst-port=8728 src-address-list=mikrodash_hosts comment="Allow MikroDash API from R5S"
/ip firewall filter add chain=input action=drop protocol=tcp dst-port=8728 comment="Drop MikroDash API from others"
```

## Часть 3. Проверка после применения на MikroTik

### Проверить сервис и пользователя

```routeros
/user group print detail where name="mikrodash"
/user print detail where name="mikrodash"
/ip service print detail where name="api"
/ip firewall address-list print where list="mikrodash_hosts"
/ip firewall filter print detail where comment~"MikroDash API"
```

Ожидаемое состояние:

- есть группа `mikrodash`
- есть пользователь `mikrodash`
- сервис `api` включен, `port=8728`, `address=192.168.1.72/32`
- в `mikrodash_hosts` только `192.168.1.72`
- есть 2 `input` правила для `8728`

### Проверка с R5S

```bash
nc -vz 192.168.1.1 8728
```

Если порт доступен, запускай/проверяй контейнер:

```bash
cd /opt/mikrodash/deploy/r5s
docker compose logs --tail=100 mikrodash
curl -i http://127.0.0.1:3081/healthz
```

### Проверка counters на MikroTik

```routeros
/ip firewall filter print stats where comment~"MikroDash API"
```

Если MikroDash подключился, у allow-правила появятся счетчики.

## Часть 4. Smoke-проверка

После старта контейнера:

```bash
cd /opt/mikrodash/deploy/r5s
docker compose ps
docker compose logs --tail=200 mikrodash
curl -i http://127.0.0.1:3081/healthz
```

Проверить в UI:

- открывается Dashboard
- проходит Basic Auth
- есть интерфейсы и неотрицательные скорости
- открываются DHCP, Logs, VPN
- нет пустого белого экрана и явных ошибок

Проверка restart:

```bash
docker compose restart mikrodash
sleep 10
curl -i http://127.0.0.1:3081/healthz
```

## Часть 5. Обновление после новых PR

Когда в ваш `main` попадают новые фиксы:

```bash
ssh root@<R5S_IP>
cd /opt/mikrodash
git pull origin main
cd deploy/r5s
docker compose up -d --build
docker compose logs --tail=100 mikrodash
curl -i http://127.0.0.1:3081/healthz
```

## Часть 6. Rollback

### Rollback на R5S

```bash
cd /opt/mikrodash
git log --oneline -n 5
git checkout <PREVIOUS_GOOD_COMMIT>
cd deploy/r5s
docker compose up -d --build
```

### Rollback на MikroTik

```routeros
/ip firewall filter remove [find where comment="Allow MikroDash API from R5S"]
/ip firewall filter remove [find where comment="Drop MikroDash API from others"]
/ip firewall address-list remove [find where list="mikrodash_hosts"]
/ip service set api disabled=yes address=""
/user remove [find where name="mikrodash"]
/user group remove [find where name="mikrodash"]
```

## Часть 7. Полезные замечания

- Для данного `hEX S` лучше держать MikroDash вне роутера — на `R5S`
- Для production лучше не использовать `latest`, а деплоить только проверенный `main`
- Если потом будет reverse proxy, оставляй Basic Auth включенным и в MikroDash тоже
- Отдельно стоит позже проверить, нужны ли тебе `auto-media-sharing` и `auto-smb-sharing` в `/disk settings`
