# 🌐 Plyglo — VM #1: EDGE

Репозиторий инфраструктуры **VM #1 (EDGE)** проекта Plyglo.

Здесь живут:
- **Caddy** — reverse proxy, TLS (Let's Encrypt), маршрутизация по доменам
- **site** — Nuxt 3 (SSR + BFF на Nitro), обслуживает `plyglo.com` и `api.plyglo.com`
- **app** — Expo web (SPA), обслуживает `study.plyglo.com`
- **Агенты мониторинга** — node-exporter, cAdvisor, promtail

VM #1 — **единственная публичная VM**. Только она слушает 80/443 и имеет публичный IP.

---

## 1. Роль VM в общей архитектуре

```
┌──────────────────────────────────────────────────────────────────┐
│                        ИНТЕРНЕТ                                  │
└────────────────────────────┬─────────────────────────────────────┘
                             │ :80 / :443
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  VM #1 — EDGE  (этот репозиторий)                                │
│                                                                  │
│  Caddy ─┬──► site:3000        (plyglo.com)                       │
│         ├──► site:3000        (api.plyglo.com)                   │
│         ├──► app:3000         (study.plyglo.com)                 │
│         └──► 10.10.2.12:9000  (auth.plyglo.com → Authentik)      │
│                                                                  │
│  site (Nuxt BFF) ─┬──► auth.plyglo.com  (OIDC)                   │
│                   ├──► 10.10.2.11:6379  (Redis: сессии + CSRF)   │
│                   └──► 10.10.3.x        (микросервисы VM #3)     │
│                                                                  │
│  app (Expo) ──► статика (SPA)                                    │
│                                                                  │
│  node-exporter, cAdvisor, promtail                               │
└────────────────────────────┬─────────────────────────────────────┘
                             │ vmbr1
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌───────────────┐   ┌─────────────────┐   ┌──────────────────┐
│ VM #2 AUTH    │   │ VM #3 BUSINESS  │   │ VM #4 MONITORING │
│ Authentik,    │   │ courses (Go),   │   │ Prometheus,      │
│ PG, Redis     │   │ progress (Go),  │   │ Grafana, Loki,   │
│               │   │ users (Go), PG  │   │ Alertmanager     │
└───────────────┘   └─────────────────┘   └──────────────────┘
```

**Ключевое:** `site` — один контейнер, два домена. Caddy роутит оба по `Host` на `site:3000`. Nuxt сам разбирается: SSR для `plyglo.com`, BFF-роуты (`/auth/*`, `/me`, `/api/*`) для `api.plyglo.com`.

---

## 2. Домены и маршрутизация

| Домен | Бэкенд | Что отдаёт | Публично |
|---|---|---|---|
| `plyglo.com` | `site:3000` | Nuxt SSR — страницы сайта | ✅ |
| `api.plyglo.com` | `site:3000` | Nuxt Nitro — BFF + прокси API | ✅ |
| `study.plyglo.com` | `app:3000` | Expo web (SPA) | ✅ |
| `auth.plyglo.com` | `10.10.2.12:9000` | Authentik (IdP) | ✅ |
| `grafana.plyglo.com` | `10.10.4.10:3000` | Grafana (опционально) | опционально |

**Что где обрабатывается в `site`:**
- `/` и другие страницы → `app/pages/*.vue` (SSR)
- `/auth/login`, `/auth/callback`, `/auth/logout` → `server/routes/auth/*`
- `/me` → `server/routes/me.get.ts` (возвращает `user` + `csrf_token`)
- `/api/*` → `server/routes/api/[...path].ts` (прокси на VM #3)

---

## 3. Что где живёт на VM #1

| Компонент | Контейнер | Порт | Кто ходит |
|---|---|---|---|
| Caddy (http) | `caddy` | `:80` | интернет |
| Caddy (https) | `caddy` | `:443`, `:443/udp` | интернет |
| Caddy admin | `caddy` | `:2019` | localhost (внутри контейнера) |
| Caddy metrics | `caddy` | `:9180` | Prometheus (VM#4) |
| site (Nuxt) | `site` | `:3000` | Caddy (edge network) |
| app (Expo) | `app` | `:3000` | Caddy (edge network) |
| node-exporter | `node-exporter` | `:9100` (host) | Prometheus (VM#4) |
| cAdvisor | `cadvisor` | `10.10.1.10:8080` | Prometheus (VM#4) |
| promtail | `promtail` | — | Loki (VM#4) |

---

## 4. Сеть

| Интерфейс | Назначение | Адрес |
|---|---|---|
| `vmbr0` | внешний, интернет | публичный IP |
| `vmbr1` | внутренний, между VM | `10.10.1.10` |

**Docker-сети внутри VM #1:**

| Сеть | Флаг | Кто в ней |
|---|---|---|
| `edge` | обычный bridge | caddy, site, app, cadvisor, promtail |
| `internal` | `internal: true` (нет выхода наружу) | site, app |

- `edge` — для трафика Caddy ↔ site/app, и для Prometheus (VM#4) → cAdvisor.
- `internal` — для внутренних вызовов site↔app (на будущее), не даёт случайно «сходить в интернет».

**Важно:** `promtail` и `cadvisor` сидят в `edge`, **не** в `internal`. Иначе:
- promtail не достучится до Loki (`10.10.4.10:3100`) — `internal: true` блокирует исходящие.
- Prometheus (VM#4) не достучится до cAdvisor (`10.10.1.10:8080`).

---

## 5. Файлы в репозитории

```
/srv/plyglo-edge/
├── docker-compose.yml
├── Caddyfile
├── .env.example
├── .gitignore
├── promtail-config.yml
├── README.md          ← этот файл
├── caddy/
│   ├── data/          (создаётся автоматически, НЕ коммитится — сертификаты ACME)
│   └── config/        (создаётся автоматически, НЕ коммитится)
└── static/            (опционально — если используешь Releases+Volume для Expo)
```

---

## 6. Запуск

### 6.1. Первый запуск

```bash
# 1. Клонировать репозиторий
cd /srv
git clone <repo-url> plyglo-edge
cd plyglo-edge

# 2. Создать .env из шаблона
cp .env.example .env

# 3. Заполнить секреты
nano .env

# 4. Ограничить права
chmod 600 .env

# 5. Залогиниться в GHCR (для приватных образов)
echo "<PAT с правом read:packages>" | docker login ghcr.io -u imirjar --password-stdin

# 6. Запустить
docker compose pull
docker compose up -d

# 7. Проверить
docker compose ps
docker compose logs -f caddy
```

### 6.2. Проверка после запуска

```bash
# 1. Caddy загрузил конфиг без ошибок
docker compose logs caddy | grep -i error

# 2. site отвечает
curl -sI http://localhost:3000/ | head -1
# → HTTP/1.1 200 OK (или 302, если нужен логин)

# 3. app отвечает
curl -sI http://localhost:3000/ --resolve study.plyglo.com:3000:127.0.0.1 | head -1

# 4. TLS работает
curl -sI https://plyglo.com/ | head -1
# → HTTP/2 200

# 5. OIDC discovery доступен через Caddy
curl -s https://auth.plyglo.com/application/o/plyglo-api/.well-known/openid-configuration | jq .issuer
# → "https://auth.plyglo.com/application/o/plyglo-api/"

# 6. site видит Redis (VM#2)
docker exec site sh -c 'nc -zv 10.10.2.11 6379'
# → open (или "connected")

# 7. site видит Authentik
docker exec site sh -c 'wget -qO- https://auth.plyglo.com/-/health/live/'
```

### 6.3. Порядок запуска VM

**VM #1 стартует последней.** В Proxmox (VM → Options → Start/Shutdown order):

| VM | Start order | Start delay |
|---|---|---|
| VM #2 (AUTH) | **1** | 30s |
| VM #3 (BUSINESS) | 2 | 20s |
| VM #4 (MONITORING) | 3 | 10s |
| **VM #1 (EDGE)** | **4** | — |

Причина: `site` при старте делает OIDC discovery в Authentik (VM#2) и коннектится к Redis (VM#2). Если VM#2 не готова — `site` уйдёт в crash-loop (переживёт благодаря `restart: unless-stopped`, но некрасиво).

**Устойчивость к недоступности VM#2:**
- `restart: unless-stopped` в compose — бесконечные ретраи.
- OIDC discovery в Nuxt — **ленивый**, при первом запросе, а не на старте.
- Redis-клиент — с `retryStrategy`, не падает при недоступности.

---

## 7. Переменные окружения (`.env`)

| Переменная | Обязательна | Описание |
|---|---|---|
| `TZ` | нет (default `UTC`) | Часовой пояс контейнеров |
| `NUXT_OIDC_ISSUER` | да | `https://auth.plyglo.com/application/o/plyglo-api/` |
| `NUXT_OIDC_CLIENT_ID` | да | из Authentik (см. раздел 10) |
| `NUXT_OIDC_CLIENT_SECRET` | да | из Authentik |
| `NUXT_REDIS_URL` | да | `redis://:PASSWORD@10.10.2.11:6379` |
| `NUXT_SESSION_SECRET` | да | секрет для подписи временных кук (`openssl rand -hex 32`) |
| `NUXT_ENCRYPTION_KEY` | да | AES-GCM ключ для refresh_token (`openssl rand -hex 32`) |
| `NUXT_PUBLIC_API_BASE` | да | `https://api.plyglo.com` — для фронта |

Полный шаблон — в `.env.example`.

---

## 8. Caddyfile — ключевые моменты

### 8.1. Структура

```caddy
{
    email admin@plyglo.com
}

(security_headers) {
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}

plyglo.com {
    import security_headers
    encode gzip zstd
    reverse_proxy site:3000
}

api.plyglo.com {
    import security_headers
    encode gzip zstd
    reverse_proxy site:3000
}

study.plyglo.com {
    import security_headers
    encode gzip zstd
    reverse_proxy app:3000
}

auth.plyglo.com {
    import security_headers
    encode gzip zstd

    @metrics path /-/metrics/*
    respond @metrics 404

    reverse_proxy http://authentik.internal.lan:9000
}
```

### 8.2. Что критично

- **`api.plyglo.com` → `site:3000`**, не `{$API_UPSTREAM}`. Если использовать env-переменную, её надо явно задать в `environment` caddy в compose, иначе конфиг сломается.
- **`/-/metrics/*` на `auth.plyglo.com` заблокирован** (`respond @metrics 404`) — эндпоинт не должен быть публичным (CVE, связанные с брутфорсом `SECRET_KEY`).
- **`security_headers`** — общий сниппет, применяется ко всем доменам.
- **`encode gzip zstd`** — сжатие для статики и HTML.
- **`rate_limit`** — **не встроен в Caddy**. Требует кастомного образа с плагином `github.com/mholt/caddy-ratelimit` (собирается через `xcaddy`). Пока отложено; защита от брутфорса — на стороне Authentik (`Reputation Policy`).

### 8.3. Как обновлять Caddyfile

```bash
nano Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
# или
docker compose restart caddy
```

`caddy reload` — без даунтайма.

---

## 9. Как `site` (Nuxt BFF) общается с другими VM

### 9.1. OIDC (Authentik, VM#2)

Используется `openid-client` (или `nuxt-oidc-auth`).

**Настройки:**
```typescript
// server/utils/oidc.ts
import * as client from 'openid-client'

let config: client.Configuration | null = null

export async function getOidcConfig() {
  if (config) return config
  config = await client.discovery(
    new URL(process.env.NUXT_OIDC_ISSUER!),   // https://auth.plyglo.com/application/o/plyglo-api/
    process.env.NUXT_OIDC_CLIENT_ID!,
    process.env.NUXT_OIDC_CLIENT_SECRET!,
  )
  return config
}
```

**Ленивая инициализация** — discovery при первом запросе, не на старте. Если VM#2 недоступна — контейнер не упадёт, только `/auth/login` вернёт `500`.

**Валидация `id_token`:** `openid-client` автоматически проверяет подпись через JWKS (`.../jwks/`), `iss`, `aud`, `exp`.

### 9.2. Redis (VM#2)

```typescript
// server/utils/redis.ts
import Redis from 'ioredis'

export const redis = new Redis(process.env.NUXT_REDIS_URL!, {
  retryStrategy: (times) => Math.min(times * 50, 2000),
  maxRetriesPerRequest: null,
})
```

**URL:** `redis://:PASSWORD@10.10.2.11:6379`

**Ключи:**
```
session:{session_id}              → hash (user_id, roles, refresh_token, ...)
user_sessions:{user_id}           → set of session_ids
csrf:{session_id}                 → строка
```

### 9.3. Микросервисы (VM#3)

Проксирование через `server/routes/api/[...path].ts`:

```typescript
// server/routes/api/[...path].ts
export default defineEventHandler(async (event) => {
  const session = await requireSession(event)   // из middleware
  const path = getRouterParam(event, 'path')

  return proxyRequest(event, `http://courses.internal.lan:8001/${path}`, {
    headers: {
      'X-User-ID': session.user_id,
      'X-User-Roles': session.roles.join(','),
    },
  })
})
```

**Важно:** микросервисы доверяют `X-User-*` заголовкам. Они не должны быть подделаны снаружи — поэтому `api.plyglo.com` идёт только через Nuxt BFF, который сам выставляет эти заголовки, а не пробрасывает клиентские.

---

## 10. OIDC-приложение в Authentik (настройка, справка)

Настраивается в `/if/admin/` на `auth.plyglo.com` (это VM#2, но `client_id`/`secret` идут сюда, в `.env` VM#1):

**Provider:**
- Type: `OAuth2/OpenID Provider`
- Name: `plyglo-api`
- Client type: `Confidential`
- Redirect URI: `https://api.plyglo.com/auth/callback`
- Scopes: `openid profile email`

**Application:**
- Slug: `plyglo-api`
- Provider: `plyglo-api`

**Issuer:** `https://auth.plyglo.com/application/o/plyglo-api/`

**Группы:**
- `plyglo-students` → роль `student`
- `plyglo-admins` → роль `admin`

---

## 11. Безопасность

### 11.1. CSRF

**Стратегия:** `SameSite=Lax` + CSRF-токен через тело ответа `/me`.

**Почему не через куку:** `plyglo.com` и `api.plyglo.com` — разные origin. Кука на `api.plyglo.com` не видна из JS на `plyglo.com`.

**Поток:**
1. При логине BFF сохраняет `csrf:{session_id} → "<token>"` в Redis.
2. `GET /me` возвращает `{ user, csrf_token }`.
3. Фронт хранит токен **в памяти** (Pinia store / composable).
4. Небезопасные запросы → `X-CSRF-Token: <token>`.
5. BFF сверяет с Redis через `timingSafeEqual`.

См. `server/middleware/csrf.ts`.

### 11.2. CORS

`plyglo.com`, `study.plyglo.com` → `api.plyglo.com` — **cross-origin**.

`server/middleware/cors.ts`:
```typescript
const allowedOrigins = [
  'https://plyglo.com',
  'https://study.plyglo.com',
]
```

**Нельзя** `Access-Control-Allow-Origin: *` с `Access-Control-Allow-Credentials: true`. Только эхо из белого списка + `Vary: Origin`.

### 11.3. Куки

| Кука | Флаги | Что содержит |
|---|---|---|
| `__Host-session` | `HttpOnly`, `Secure`, `SameSite=Lax` | session ID |

**Одна кука.** CSRF — не в куке (см. 11.1).

### 11.4. `AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS` (справка)

Настраивается на VM#2, но касается Caddy отсюда:

```yaml
AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS: "10.10.1.0/24,10.10.2.0/24"
```

Без этого Authentik проигнорирует `X-Forwarded-*` от Caddy (VM#1) и сгенерирует редиректы на `http://10.10.2.12:9000`, что сломает вход.

---

## 12. Мониторинг

**Prometheus (VM #4) скрейпит отсюда:**
- `10.10.1.10:9100/metrics` — node-exporter
- `10.10.1.10:8080/metrics` — cAdvisor (только `vmbr1`)
- `10.10.1.10:9180/metrics` — Caddy metrics (нужно добавить `metrics` в Caddyfile, если ещё нет)

**Loki (VM #4) получает логи от:**
- `promtail` на этой VM → `http://10.10.4.10:3100/loki/api/v1/push`

**Ключевые алерты:**
- `5xx rate > 5%` на любом роутере (Caddy) → проблемы site/app/Authentik
- `p95 latency > 1s` → тормоза
- `site` недоступен > 30s → весь прод лежит
- TLS-сертификат истекает < 14 дней (Caddy обновляет сам, но если ACME сломался — узнаешь)
- Диск VM > 80% (Caddy data + Docker images растут)

**Caddy metrics** (если включены) — `caddy_http_request_duration_seconds`, `caddy_http_requests_total`, etc.

---

## 13. Бэкапы

**Что бэкапить:**
- `Caddyfile` (в Git)
- `docker-compose.yml` (в Git)
- `.env` (в защищённом месте, не в Git)
- `caddy/data/` — сертификаты ACME (не критично — Caddy перевыпустит, но лучше сохранить, чтобы не упираться в rate limit Let's Encrypt)

**Что не бэкапить:**
- Docker images (пересоберутся из GHCR)
- `caddy/config/` (генерируется Caddy)

---

## 14. Обновление

### 14.1. `site` (Nuxt)

```bash
# 1. Найти новый тег в GHCR
#    https://github.com/imirjar/plyglo-website/pkgs/container/plyglo-website

# 2. Поменять тег в docker-compose.yml
nano docker-compose.yml
#    image: ghcr.io/imirjar/plyglo-website:v0.5.5

# 3. Перезапустить только site
docker compose pull site
docker compose up -d site

# 4. Проверить
docker compose logs -f site | grep -i error
curl -sI https://plyglo.com/
```

**Zero-downtime:** `docker compose up -d site` пересоздаёт контейнер. Caddy на 1–3 секунды может отдавать `502`. Для настоящего zero-downtime — blue-green (два контейнера `site-blue`/`site-green`, Caddy роутит на активный; переключение — reload Caddyfile). Пока не нужно.

### 14.2. `app` (Expo)

Аналогично `site`: поменять тег в compose → `pull` → `up -d app`.

### 14.3. `Caddy`

```bash
docker compose pull caddy
docker compose up -d caddy
```

Caddy редко требует обновления. Следи за security-релизами на GitHub.

### 14.4. `Caddyfile`

```bash
nano Caddyfile
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

---

## 15. Диагностика

**502 на `plyglo.com`:**
```bash
docker compose ps site
docker compose logs site --tail 50
curl -sI http://localhost:3000/
```
Если `site` в `Restarting` — смотри логи, скорее всего падает на старте (OIDC/Redis).

**502 на `auth.plyglo.com`:**
```bash
curl -sI http://10.10.2.12:9000/-/health/live/
```
Если не отвечает — VM#2 упала. Если отвечает, а через Caddy — 502: проблема с `authentik.internal.lan` (DNS) или с сетью `vmbr1`.

**`site` падает в crash-loop:**
```bash
docker compose logs site | tail -100
```
Частые причины:
- `NUXT_OIDC_ISSUER` недоступен → проверить `curl https://auth.plyglo.com/.../.well-known/openid-configuration` с VM#1
- `NUXT_REDIS_URL` неверный или Redis недоступен → `nc -zv 10.10.2.11 6379`
- `NUXT_ENCRYPTION_KEY` невалидный (не hex/не та длина) → проверить `openssl rand -hex 32` (64 hex-символа)

**`redis.internal.lan` не резолвится из `site`:**
```bash
docker exec site sh -c 'nslookup redis.internal.lan'
```
Если NXDOMAIN — временно используется `extra_hosts` в compose. Долгосрочное решение — внутренний DNS на `vmbr1` (см. открытые вопросы).

**Логин возвращает `iss mismatch`:**
```bash
curl -s https://auth.plyglo.com/application/o/plyglo-api/.well-known/openid-configuration | jq .issuer
```
Должно быть `https://auth.plyglo.com/application/o/plyglo-api/`. Если не так — `AUTHENTIK_HOST` на VM#2 не применился, перезапустить `authentik-server`.

**Caddy не выпускает сертификат:**
```bash
docker compose logs caddy | grep -i acme
```
Частые причины:
- Порт 80 не доступен из интернета (проверь firewall, DNS A-запись домена)
- Rate limit Let's Encrypt (5 неудачных попыток в час) — подожди или используй staging (`acme_ca https://acme-staging-v02.api.letsencrypt.org/directory` в global options Caddyfile)

**CSRF-ошибка `403` на POST:**
```bash
docker exec sessions-redis redis-cli -a <PASSWORD> --scan --pattern 'csrf:*'
```
Если ключа нет — сессия истекла, фронт должен перезапросить `/me`. Если есть, но не совпадает — смотри, что фронт шлёт в `X-CSRF-Token`.

---

## 16. Открытые вопросы (специфично для VM #1)

1. **Внутренний DNS на `vmbr1`** — сейчас в `site` используется `extra_hosts` для `redis.internal.lan`, `authentik.internal.lan`, `courses.internal.lan` и т.д. Нужен нормальный DNS (Pi-hole/dnsmasq).
2. **Caddy metrics** — включить `metrics` в global options Caddyfile, чтобы Prometheus скрейпил `:9180`.
3. **Rate limiting на `auth.plyglo.com`** — требует кастомного образа Caddy с `xcaddy`. Отложено.
4. **Zero-downtime для `site`** — blue-green (два контейнера + reload Caddyfile). Сейчас краткий `502` при обновлении.
5. **Expo — Docker image vs Releases+Volume** — сейчас используется образ `app:3000`. Альтернатива: GitHub Releases + symlink + Caddy `file_server` (zero-downtime, минус контейнер).
6. **Бэкапы `caddy/data/`** — куда складывать.
7. **`forward_auth` в Caddy** — на старте не используется (проверка сессии — в Nuxt middleware). Добавить, когда понадобится блокировать трафик до Nuxt (например, для защиты `/api/admin/*`).

---

## 17. Полезные команды

```bash
# Статус
docker compose ps
docker compose logs -f caddy
docker compose logs -f site

# Перезапуск только site
docker compose restart site

# Пересоздание с новым образом
docker compose pull site
docker compose up -d site

# Reload Caddy без даунтайма
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile

# Проверить Caddyfile синтаксически
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile

# Зайти в контейнер site
docker exec -it site sh

# Проверить сессию в Redis
docker exec -it sessions-redis redis-cli -a <PASSWORD> GET session:<id>

# Все активные сессии
docker exec -it sessions-redis redis-cli -a <PASSWORD> --scan --pattern 'session:*' | wc -l

# Место на диске (Docker images + Caddy data)
df -h /var/lib/docker /srv/plyglo-edge

# Пересобрать образ локально (если нужно)
# docker compose build site  — только при build:, для GHCR не нужно
```

---

## 18. Стек технологий

- **Reverse proxy:** Caddy 2 (автоматический TLS, HTTP/3)
- **SSR + BFF:** Nuxt 3 / Nitro (TypeScript), OIDC через `openid-client`
- **SPA:** Expo web (React Native for Web)
- **Контейнеризация:** Docker + Docker Compose
- **CI/CD:** GitHub Actions → GHCR (`ghcr.io/imirjar/plyglo-website`, `ghcr.io/imirjar/plyglo-expo`)
- **Мониторинг:** Prometheus, Grafana, Loki, Alertmanager (на VM #4)
- **OS:** Debian/Ubuntu (Linux)
```
