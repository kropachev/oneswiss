# Пример добавления OneSwiss в существующую инфраструктуру Traefik (HTTPS)

- [Возможности и прелести Traefik](#возможности-и-прелести-traefik)
- [Особенности OneSwiss за Traefik](#особенности-oneswiss-за-traefik)
  - [WebSocket, MCP и большие .cf](#websocket-mcp-и-большие-cf)
  - [Ограничение wildcard-сертификатов](#ограничение-wildcard-сертификатов)
- [Подготовка Traefik](#подготовка-traefik)
  - [ProxyAddress и IP Traefik](#proxyaddress-и-ip-traefik)
  - [Увеличение таймаутов для загрузки больших файлов](#увеличение-таймаутов-для-загрузки-больших-файлов)
  - [Применение изменений Traefik](#применение-изменений-traefik)
- [Развертывание OneSwiss](#развертывание-oneswiss)
  - [DNS](#dns)
  - [Подготовка конфигурационных файлов](#подготовка-конфигурационных-файлов)
  - [Запуск и проверка](#запуск-и-проверка)
  - [Настройка агентов](#настройка-агентов)
  - [Сервис регистрации ошибок 1С](#сервис-регистрации-ошибок-1с)

## Возможности и прелести Traefik

Traefik умеет автоматически обнаруживать Docker-контейнеры на хосте, где он работает, и получать для них TLS-сертификаты. Чтобы опубликовать новый сервис, достаточно подключить его контейнеры к общей сети Traefik и добавить в compose-файл метки с правилами маршрутизации. Traefik увидит контейнеры, создаст маршруты и запросит необходимые сертификаты. 

Но с OneSwiss все будет не так просто. Эта инструкция показывает, как добавить OneSwiss к уже развернутой инфраструктуре Traefik учитывая особенности заголовков и передачи больших файлов (cf erp). Для простоты будем использовать стандартный `docker-compose.yml` из корня проекта, а все настройки, связанные с прокси, добавим в отдельный файл `docker-compose.traefik.yml`.  
Дополнительный файл содержит следующие изменения:

- убирает прямую публикацию портов `7002` и `3000`;
- подключает `server` и `web` к внешней сети Traefik;
- добавляет роутеры, middleware и внутренние порты сервисов;
- передает серверу адрес доверенного прокси через `ProxyAddress`.

Оба compose-файла применяются одной командой, поэтому копировать и отдельно поддерживать полный `docker-compose.yml` не требуется.

Веб-панель и сервер публикуются на двух отдельных поддоменах:

- `oneswiss.example.com` - веб-панель (`web:3000`);
- `api.oneswiss.example.com` - сервер (`server:7002`).

Снаружи используются стандартные порты Traefik: **80 для HTTP** (перенаправление на HTTPS и, при необходимости, ACME HTTP-01) и **443 для HTTPS**. Порты контейнеров `3000` и `7002` доступны Traefik внутри Docker-сети, на хосте не публикуются.

В инструкции используются два каталога на Docker-хосте:

- **Каталог OneSwiss** - корень проекта с исходным `docker-compose.yml`. Здесь по порядку создаются `.env` и `docker-compose.traefik.yml`.
- **Каталог Traefik** - каталог уже работающего compose-проекта Traefik. Для увеличении таймаута в этом каталоге также создается  `dynamic/oneswiss-timeouts.yml`.

## Особенности OneSwiss за Traefik

### WebSocket, MCP и большие .cf

Для WebSocket и потоковых ответов дополнительные метки Traefik не нужны:

- **WebSocket / SignalR** - Traefik автоматически обрабатывает переключение протокола. Хабы `/agentsHub`, `/taskLogHub`, `/updatesHub` и подключение агентов через `/ws/agents` работают через общий API-роутер.
- **MCP на `/mcp`** - используется транспорт Streamable HTTP, который может передавать ответы потоком через SSE. Traefik не буферизует такие ответы, если к роутеру не подключен buffering middleware.
- **Загрузка `.cf`** - без buffering middleware Traefik не задает ограничение размера тела запроса. OneSwiss также настроен на прием больших файлов. Особенности настройки таймаутов рассмотрены в отдельном разделе.

### Ограничение wildcard-сертификатов

В этой схеме внешний TCP-роутер выбирает увеличенный таймаут по SNI домена API. Один wildcard- или SAN-сертификат не должен одновременно покрывать домен API и соседние сайты на том же IP, которые остаются на обычном entrypoint.

Например, сертификат `*.example.com` покрывает и `api-oneswiss.example.com`, и `grafana.example.com`. Браузер может открыть HTTP/2-соединение к Grafana и повторно использовать его для API без нового TLS-соединения. Traefik в этом случае не получит новый SNI и не направит запрос на entrypoint с увеличенным таймаутом.

В примерах используется `api.oneswiss.example.com`: сертификат `*.example.com` это имя не покрывает, поэтому для API нужен отдельный сертификат. Если инфраструктура может предоставить только общий wildcard, для API потребуется отдельный IP. Еще один вариант - отключить HTTP/2 у всех сайтов на этом IP, сертификаты которых покрывают домен API; это затронет и остальные сервисы.

## Подготовка Traefik

В существующем Traefik должны быть настроены Docker provider, внешний HTTPS entrypoint на 443, ACME-резолвер и доступ к общей Docker-сети. HTTP на 80 должен перенаправляться на HTTPS. Ниже предполагаются имена entrypoints `web`/`websecure`.

### ProxyAddress и IP Traefik

OneSwiss ожидает, что IP доверенного прокси будет явно указан в параметре `ProxyAddress`. Traefik в Docker-сети по умолчанию получает динамический IP. Он может измениться при пересоздании контейнера, в том числе после обновления образа, поэтому Docker не рекомендует полагаться на сохранение динамического адреса.

IP нужен в одном месте - в `.env` как `TRAEFIK_IP=172.28.0.10`. Оттуда он подставится и в переменную окружения `ProxyAddress` сервера, и в docker-метку с описанием middleware `oneswiss-ms-proxy`, которая выставляет заголовок `X-MS-Proxy`.

**Простой способ - взять текущий IP Traefik**

В терминале Docker-хоста, из любого каталога, смотрим текущий IP Traefik:

```bash
docker network inspect traefik-public \
  --format '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
```

Вместо `traefik-public` нужно подставить имя внешней сети. Команда выводит адрес вместе с маской подсети, например `172.18.0.4/16`. В `.env` указываем только часть до `/`: `TRAEFIK_IP=172.18.0.4`. 

> 💡 При пересоздании Traefik нужно проверять его IP. Если адрес изменился, обновлять `TRAEFIK_IP` в `.env` каталога OneSwiss и повторять команды запуска. Новое значение будут попадать и в `ProxyAddress`, и в метку middleware.

**Нормальный способ - закрепить IP явно**

`traefik-public` в примерах ниже - имя внешней Docker-сети, общей для Traefik
и публикуемых через него контейнеров. Используем фактическое имя сети из
`services.traefik.networks` в compose-файле Traefik.

Чтобы гарантировать неизменность адреса, Traefik можно привязать к свободному IP из подсети существующей внешней сети. В терминале Docker-хоста, из любого каталога, смотрим подсеть:

```bash
docker network inspect traefik-public \
  --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}'
```

Выбираем в этой подсети свободный адрес, который не используется другими контейнерами. В существующем `docker-compose.yml` **в каталоге Traefik** дополняем `services.traefik.networks` и описание сети следующим фрагментом, сохраняя остальные настройки файла:

```yaml
services:
  traefik:
    networks:
      traefik-public:
        ipv4_address: 172.28.0.10

networks:
  traefik-public:
    name: traefik-public
    external: true
```

Закрепленный IP применим вместе с остальными изменениями Traefik. Общую
Docker-сеть пересоздавать не нужно.

> 💡 в примере из README OneSwiss (`X-Ms-Proxy: 10.10.0.1`) заголовок выставляет корпоративный edge-прокси. В нашем случае Traefik - сам этот прокси, поэтому в `X-MS-Proxy` пишется **его же** IP. Если перед Traefik стоит еще один фронт (LB, cloudflare tunnel), значение может отличаться - такие вещи нужно будет согласовать с сетевиками.

### Увеличение таймаутов для загрузки больших файлов

В Traefik входящий таймаут задается на entrypoint, а не на отдельном HTTP-роутере. Поэтому добавим внутренний entrypoint и направим на него только домен API OneSwiss.

Снаружи останутся **80/443**. Порт `8443` будет слушать только loopback внутри контейнера Traefik.

**Настройка `traefik.yml`**

Для добавления entrypoint придется скорректировать статические настройки в `traefik.yml` в каталоге Traefik.

Роутеры без явно указанного entrypoint могут автоматически подключиться к новому entrypoint OneSwiss. Чтобы этого не произошло, нужно добавить `asDefault` к существующим HTTP/TCP entrypointsв файле `traefik.yml`, или убедиться что уже добавлено.

У нового `websecure-extended-timeout` указываем `asDefault: false`. Этот параметр исключает его из автоматического выбора.

В `traefik.yml` дополняем разделы `entryPoints` и`providers` по примеру ниже. Существующие адреса `:80`/`:443`, редиректы и остальные настройки сохраняем. Если File provider для динамических настроек уже добавлен, раздел `providers.file` оставляем как есть.

```yaml
entryPoints:
  web:
    # Добавленный параметр, если asDefault: true ранее нигде не был задан.
    asDefault: true
  websecure:
    # Добавленный параметр при том же условии.
    asDefault: true
  # Добавленный раздел: отдельный entrypoint для долгих загрузок OneSwiss.
  websecure-extended-timeout:
    address: "127.0.0.1:8443"
    asDefault: false
    proxyProtocol:
      trustedIPs:
        - "127.0.0.1/32"
    transport:
      respondingTimeouts:
        readTimeout: "1h"
        writeTimeout: "0s"
        idleTimeout: "180s"

providers:
  # Добавляем этот раздел, только если File provider еще не настроен.
  file:
    directory: /etc/traefik/dynamic
    watch: true
```

`1h` - пример лимита на прием всего запроса. Выбираем его по размеру файла и скорости загрузки с запасом.  
`writeTimeout: 0s` и `idleTimeout: 180s` стандартные значения на новом entrypoint.  
Таймауты общих `web` и `websecure` не меняем.

Если File provider для динамических настроек уже добавлен в `traefik.yml`, сохраняем его текущие `directory` или `filename` и подключенный том.  
Если он еще не настроен, можно использовать путь `/etc/traefik/dynamic` и добавить в `docker-compose.yml` Traefik подключение каталога к существующему списку `services.traefik.volumes`:

```yaml
services:
  traefik:
    volumes:
      # Остальные существующие подключения сохраняем.
      - ./dynamic:/etc/traefik/dynamic:ro
```

Изменения `traefik.yml` и `volumes` требуют пересоздания контейнера Traefik.

### Применение изменений Traefik

Этот шаг выполняем, только если меняли файлы **в каталоге Traefik**. В терминале Docker-хоста и переходим в каталог Traefik, где находится его `docker-compose.yml`. Команды ниже предполагают имя сервиса `traefik`, при другом имени используем его. Если проект запускается с дополнительными compose-файлами, сохраняем все его обычные параметры `-f`.

Проверяем compose-файл Traefik:
```bash
docker compose -f docker-compose.yml config --quiet
```

Если ответ пустой, значит ошибок нет. Пересоздаем только его контейнер, это применит как изменения compose-файла, так и статической конфигурации:

```bash
docker compose -f docker-compose.yml up -d --no-deps --force-recreate traefik
```

## Развертывание OneSwiss

### DNS

В настройка DNS локальной сети или своего домена нужно создать две A-записи, обе указывают на IP хоста, где работает Traefik:

```
oneswiss.example.com       A   <IP хоста с Traefik>
api.oneswiss.example.com   A   <IP хоста с Traefik>
```

Traefik выпустит для каждого имени свой сертификат через настроенный ACME-резолвер.

### Подготовка конфигурационных файлов

**Создаем каталог OneSwiss**

Создаем на Docker-хосте отдельный каталог установки, например `~/oneswiss` и переходим в него:
```bash
mkdir -p ~/oneswiss
cd ~/oneswiss
```

Копируем в этот каталог `docker-compose.yml` и `.env.example` из корня одной
и той же версии OneSwiss. Все следующие команды раздела выполняем из этого
каталога.

**Генерация секретов**

`DB_PASSWORD` и `JWT_SIGNING_KEY` - длинные случайные строки. Для `JWT_SIGNING_KEY` требуется не менее 32 символов. Генерируем значения однократно следующими командами в терминале Linux-хоста, из любого каталога. Полученные значения сохраняем для `.env`, который создадим следующим шагом.

```bash
# DB_PASSWORD (32 символа base64, 24 байта энтропии)
head -c 24 /dev/urandom | base64

# JWT_SIGNING_KEY (44 символа base64, 32 байта энтропии)
head -c 32 /dev/urandom | base64
```

**Переменные `.env`**

В **каталоге OneSwiss**, рядом с исходным `docker-compose.yml`, создаем `.env` на основе `.env.example` и добавляем параметры для Traefik. Ниже приведен полный текст `.env`, нужно заменить домены, параметры инфраструктуры и секреты на свои.

Hostname хранится один раз: `PUBLIC_BACKEND_URL` и `PUBLIC_UI_URL` собираются
из `PUBLIC_BACKEND_HOST` и `PUBLIC_UI_HOST`. Эти же hostname используются в
правилах `Host(...)` Traefik.

```env
IMAGE_REGISTRY=
IMAGE_TAG=

# Порты, публикуемые на хосте (слева от ":" в docker-compose.yml). Меняйте, если они
# пересекаются с другими контейнерами/сервисами на этом хосте. Порт внутри контейнера
# не меняется.
# Для Traefik: override убирает публикацию этих портов; оставляем стандартные значения.
SERVER_PORT=7002
WEB_PORT=3000

# Публичный адрес бэкенда, по которому он доступен из БРАУЗЕРА пользователя (не имя сервиса
# в Docker-сети). Если меняете SERVER_PORT, приведите порт здесь в соответствие.
# Для Traefik: HTTPS на 443; SERVER_PORT на этот URL не влияет, порт не добавляем.
# PUBLIC_BACKEND_HOST - домен для правила Host(...) Traefik и сборки URL.
PUBLIC_BACKEND_HOST=api.oneswiss.example.com
PUBLIC_BACKEND_URL=https://${PUBLIC_BACKEND_HOST}

# Публичный адрес веб-панели (используется бэкендом для CORS). Если меняете WEB_PORT,
# приведите порт здесь в соответствие.
# Для Traefik: HTTPS на 443; WEB_PORT на этот URL не влияет, порт не добавляем.
# PUBLIC_UI_HOST - домен для правила Host(...) Traefik и сборки URL.
PUBLIC_UI_HOST=oneswiss.example.com
PUBLIC_UI_URL=https://${PUBLIC_UI_HOST}

# Имя внешней Docker-сети, к которой подключен Traefik.
TRAEFIK_NETWORK=traefik-public

# Имя HTTPS entrypoint в статической конфигурации Traefik.
TRAEFIK_ENTRYPOINT=websecure

# Отдельный entrypoint с увеличенным таймаутом для API OneSwiss.
TRAEFIK_API_ENTRYPOINT=websecure-extended-timeout

# Имя ACME certificate resolver в конфигурации Traefik.
TRAEFIK_CERT_RESOLVER=default

# IP контейнера Traefik во внешней сети, без маски подсети.
TRAEFIK_IP=172.28.0.10

# Секреты.
DB_PASSWORD=change-me
JWT_SIGNING_KEY=change-me-to-a-long-random-secret-at-least-32-chars
```

Docker Compose поддерживает подстановку переменных внутри значений `.env`, поэтому итоговыми значениями будут, например,  
`PUBLIC_BACKEND_URL=https://api.oneswiss.example.com` и  
`PUBLIC_UI_URL=https://oneswiss.example.com`.

Если OneSwiss разворачивается впервые, `DB_PASSWORD` будет применен при создании базы данных. Дополнительные действия потребуются при повторном развертывании с уже существующим томом `db_data`, переменная `POSTGRES_PASSWORD` не изменяет пароль в ранее созданной базе. В этом случае нужно указать в `DB_PASSWORD` действующий пароль либо сначала изменить пароль пользователя в PostgreSQL.

**Создаем `docker-compose.traefik.yml`**

В **каталоге OneSwiss** создаем `docker-compose.traefik.yml` рядом со стандартным
`docker-compose.yml` и уже созданным `.env`. Ниже приведен полный текст нового
файла. Дополнительный файл применяется поверх основного и
содержит только изменения, необходимые для публикации через Traefik.

```yaml
services:
  db:
    container_name: oneswiss-db
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  server:
    container_name: oneswiss-server
    ports: !reset []
    environment:
      ConnectionStrings__Default: "Host=db;Port=5432;Database=oneswiss;Username=oneswiss;Password=${DB_PASSWORD}"
      Auth__Jwt__SigningKey: ${JWT_SIGNING_KEY}
      ProxyAddress: ${TRAEFIK_IP}
    networks:
      - default
      - traefik-public
    volumes:
      - server_data:/app/Data
      - data_protection_keys:/root/.aspnet/DataProtection-Keys
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=${TRAEFIK_NETWORK}"
      # Middleware: добавить X-MS-Proxy к каждому запросу к API.
      - "traefik.http.middlewares.oneswiss-ms-proxy.headers.customrequestheaders.X-MS-Proxy=${TRAEFIK_IP}"
      - "traefik.http.routers.oneswiss-api.rule=Host(`${PUBLIC_BACKEND_HOST}`)"
      - "traefik.http.routers.oneswiss-api.entrypoints=${TRAEFIK_API_ENTRYPOINT}"
      - "traefik.http.routers.oneswiss-api.tls.certresolver=${TRAEFIK_CERT_RESOLVER}"
      - "traefik.http.routers.oneswiss-api.middlewares=oneswiss-ms-proxy@docker"
      - "traefik.http.routers.oneswiss-api.service=oneswiss-api"
      - "traefik.http.services.oneswiss-api.loadbalancer.server.port=7002"

  web:
    container_name: oneswiss-web
    ports: !reset []
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=${TRAEFIK_NETWORK}"
      - "traefik.http.routers.oneswiss-web.rule=Host(`${PUBLIC_UI_HOST}`)"
      - "traefik.http.routers.oneswiss-web.entrypoints=${TRAEFIK_ENTRYPOINT}"
      - "traefik.http.routers.oneswiss-web.tls.certresolver=${TRAEFIK_CERT_RESOLVER}"
      - "traefik.http.routers.oneswiss-web.service=oneswiss-web"
      - "traefik.http.services.oneswiss-web.loadbalancer.server.port=3000"

networks:
  default:
  traefik-public:
    name: ${TRAEFIK_NETWORK}
    external: true

volumes:
  data_protection_keys:
```

Все параметры этого override, зависящие от инфраструктуры, задаются в `.env`.
Оба compose-файла можно использовать без локальных правок.

**На что обратить внимание**

- `ports: !reset []` удаляет прямую публикацию портов, заданную в стандартном `docker-compose.yml`. После объединения файлов сервер и веб-панель доступны снаружи только через Traefik.

- `ProxyAddress: ${TRAEFIK_IP}` указывает серверу IP доверенного прокси. Без этой настройки сервер не будет использовать переданные Traefik заголовки `X-Forwarded-For` и `X-Forwarded-Proto`. Сами эти заголовки Traefik добавляет автоматически.

- Middleware `oneswiss-ms-proxy` добавляет нестандартный заголовок `X-MS-Proxy`, который требуется OneSwiss. Суффикс `@docker` в `oneswiss-ms-proxy@docker` означает, что middleware объявлен через Docker labels. Он подключен только к API-роутеру: веб-панели этот заголовок не требуется.

- `traefik.docker.network=${TRAEFIK_NETWORK}` сеть, через которую Traefik обращается к контейнеру. Для `server` это важно, поскольку он подключен сразу к двум сетям, внутренней сети с PostgreSQL и внешней сети Traefik.

- `loadbalancer.server.port` задает внутренний порт контейнера: `7002` для API и
`3000` для веб-панели. Порты указаны явно, чтобы маршрутизация не зависела от
директив `EXPOSE` внутри Docker-образов.

- Volume `data_protection_keys` сохраняет ключи ASP.NET Data Protection при
  обновлении или пересоздании контейнера сервера. Без постоянного volume могут
  перестать расшифровываться ранее выданные authentication cookies, а
  незавершенный вход через OIDC/SSO может завершиться ошибкой.

API-роутер не содержит `PathPrefix`, поэтому передает серверу все пути, включая
API, WebSocket, SignalR, MCP, статические данные и прокси хранилищ конфигураций.

Traefik обнаружит контейнеры после их подключения к общей сети и начнет выпуск
сертификатов. Если маршруты не появились или HTTPS не заработал, проверьте логи
Traefik.

**Динамические настройки traefik `dynamic/oneswiss-timeouts.yml`**

Создаем `oneswiss-timeouts.yml` в каталоге динамических настроек Traefik.
Если при подготовке Traefik был указан `/etc/traefik/dynamic`, создаем файл
`dynamic/oneswiss-timeouts.yml` в каталоге Traefik.

```yaml
tcp:
  routers:
    oneswiss-api-passthrough:
      entryPoints:
        - websecure
      rule: "HostSNI(`api.oneswiss.example.com`)"
      service: oneswiss-loopback
      tls:
        passthrough: true
  services:
    oneswiss-loopback:
      loadBalancer:
        serversTransport: oneswiss-loopback-proxy-v2
        servers:
          - address: "127.0.0.1:8443"
  serversTransports:
    oneswiss-loopback-proxy-v2:
      proxyProtocol:
        version: 2
```

**На что обратить внимание**

- **`entryPoints`.** Указываем внешний entrypoint, который принимает HTTPS на порту 443: значение `TRAEFIK_ENTRYPOINT` из `.env` OneSwiss. В примере это `websecure`. TCP-роутер передаст соединение с него на внутренний `websecure-extended-timeout`, поэтому здесь это имя не указываем.
- **`rule`.** В `HostSNI` вручную указываем значение `PUBLIC_BACKEND_HOST` из `.env` OneSwiss. В приведенном примере это `api.oneswiss.example.com`.
- **`tls.passthrough`.** Оставляем значение `true`, чтобы внешний TCP-роутер передал TLS-соединение на внутренний entrypoint.
- **`oneswiss-loopback-proxy-v2`.** Это имя TCP-транспорта, объявленного ниже в `serversTransports`. Сервис `oneswiss-loopback` использует его для передачи соединения на `127.0.0.1:8443` вместе с заголовком PROXY protocol v2. Имя можно изменить, но тогда такое же значение нужно указать в `serversTransport` сервиса.
- **`proxyProtocol.version`.** Оставляем значение `2`. Через PROXY protocol внешний TCP-роутер передает внутреннему entrypoint исходный IP клиента.

### Запуск и проверка

Теперь **в терминале Docker-хоста переходим в каталог OneSwiss**, где находятся
его `.env`, `docker-compose.yml` и `docker-compose.traefik.yml`.
Проверяем итоговую конфигурацию следующей командой:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.traefik.yml \
  config --quiet
```

Команда проверяет синтаксис, подстановку переменных и объединение compose-файлов.
При успешной проверке она не выводит сообщений. Это не проверка конфигурации
самого Traefik: ошибки его entrypoints, маршрутов и транспортов проверяются
отдельно в журнале и dashboard.

Если ошибок нет, **из того же каталога OneSwiss** запускаем приложение:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.traefik.yml \
  up -d
```
> 💡 Оба параметра `-f` следует использовать и при последующих командах `pull`, `logs`,
`ps` и `down`.

Если первичная установка завершилась неудачно и ее нужно начать заново,
останавливаем сервисы и удаляем контейнеры вместе с томами OneSwiss:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.traefik.yml \
  down -v
```

Параметр `-v` удалит том PostgreSQL `db_data` вместе со всеми данными. Эту
команду используем только для очистки неудачной установки. Внешняя сеть
Traefik и его контейнер удалены не будут.

После запуска веб-панели Traefik в разделе HTTP-Routers появится строка с сервисом oneswiss-web. К сожалению Traefik не показывает статус выпуска сертификата, поэтому придется подождать некоторое время, пока сервис станет доступен без предупреждений безопасности.

В dashboard Traefik проверяем созданные маршруты:

1. Открываем раздел **TCP - Routers**, находим
   `oneswiss-api-passthrough@file` и открываем его карточку. В поле
   **EntryPoints** должен быть `websecure`, в поле **Service** -
   `oneswiss-loopback`, а в правиле - домен API. Переходим по ссылке сервиса:
   в его карточке должно отображаться полное имя `oneswiss-loopback@file`.
2. Открываем раздел **HTTP - Routers**, находим `oneswiss-api@docker` и
   открываем его карточку. В поле **EntryPoints** должен быть
   `websecure-extended-timeout`, а в поле **Service** -
   `oneswiss-api`. Переходим по ссылке сервиса: в его карточке должно
   отображаться полное имя `oneswiss-api@docker`.
3. Возвращаемся к общему списку **HTTP - Routers** и смотрим колонку
   **EntryPoints**. Значение `websecure-extended-timeout` должно быть только в
   строке `oneswiss-api@docker`. У остальных роутеров в этой колонке должны
   остаться прежние entrypoint, например `websecure`.

Затем проверяем загрузку `.cf` длительностью больше 60 секунд, скачивание, WebSocket и MCP, если он включен. В запросах к OneSwiss должны сохраняться IP клиента и схема `https`. Веб-панель и соседние сервисы должны работать через прежние entrypoints со стандартными таймаутами.

**В терминале, из каталога OneSwiss**, проверяем, что порты `7002` и `3000` не опубликованы на хосте:
```bash
docker compose -f docker-compose.yml -f docker-compose.traefik.yml ps
```

### Настройка агентов

Агент - это отдельное приложение, устанавливаемое **на серверах 1С**. У него свой файл конфигурации `appsettings.json`, лежащий рядом с исполняемым файлом.

После перехода сервера на HTTPS в этот файл нужно подставить публичные адреса именно **HTTPS/WSS**:

```json
{
  "Server": "wss://api.oneswiss.example.com",
  "Auth": {
    "TokensEndpoint": "https://api.oneswiss.example.com/api/auth/token"
  }
}
```

Полный набор полей (`InstanceName`, `PlatformPaths`, `Auth.ClientId`/`ClientSecret` и т.д.) описан в основном README.

### Сервис регистрации ошибок 1С

```
https://api.oneswiss.example.com/api/ErrorLoggingService
```

Шаблон `{URL}/api/ErrorLoggingService` - из README.
