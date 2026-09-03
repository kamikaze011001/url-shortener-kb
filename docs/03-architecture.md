# 03 — Architecture

## Shape

One Spring Boot application, one Postgres, one Redis, one Caddy. **A single deployable
backend**, not microservices — at this scale, splitting the redirect service from the
management API would buy independent scaling nobody needs and cost a network hop on
the one path that matters.

The split is real, but it lives at the **module** level inside one process, so that
extracting it later is a refactor rather than a rewrite:

```
url-shortener-backend/
  redirect/      ← the hot path. Read-only against Links. No dependency on management.
  links/         ← create, edit, disable, delete. Owns the Link.
  analytics/     ← recording Clicks and reading statistics.
  identity/      ← Owners, registration, login, session.
  shared/        ← error model, IP resolution, config.
```

`redirect` depends on nothing but `shared` and its own read model. That is the seam
along which the service would be split if the redirect path ever needed to scale
independently, and keeping it clean costs nothing today.

## Containers

```
                    Internet
                        │
              ┌─────────▼─────────┐
              │ Cloudflare edge   │  TLS terminates here.
              │                   │  Adds CF-Connecting-IP, CF-IPCountry.
              └─────────┬─────────┘
                        │  (tunnel)
              ┌─────────▼─────────┐
              │   cloudflared     │  existing tunnel, two new ingress rules
              └────┬─────────┬────┘
                   │         │
    s.example.com  │         │  app.example.com
                   │         │
          ┌────────▼───┐  ┌──▼──────────────┐
          │            │  │  Caddy :8081    │
          │            │  │  /api/* ──────► │──┐
          │            │  │  /*  static SPA │  │
          │            │  └─────────────────┘  │
          │            │                       │
          └────────┬───┴───────────────────────┘
                   │
        ┌──────────▼──────────┐
        │  Spring Boot :8080  │  Java 21, virtual threads
        └───┬─────────────┬───┘
            │             │
   ┌────────▼──────┐  ┌───▼──────────┐
   │ Postgres 16   │  │  Redis 7     │
   │ source of     │  │  cache +     │
   │ truth         │  │  rate limits │
   └───────────────┘  └──────────────┘
```

Everything below `cloudflared` runs as one `docker compose` stack on a single machine.

## Hostname topology

Two public hostnames. See [ADR-0006](./adr/0006-two-hostname-topology.md) for why
two and not one or three.

| Hostname | Path | Served by | Purpose |
|---|---|---|---|
| `s.example.com` | `/{code}` | Spring Boot `:8080` | Redirect. **The entire root namespace belongs to Short Codes.** |
| `s.example.com` | `/` | Spring Boot `:8080` | Minimal landing page |
| `app.example.com` | `/api/v1/*` | Caddy → Spring Boot `:8080` | REST API |
| `app.example.com` | `/*` | Caddy → static bundle | React dashboard |

Because the dashboard and the API share an origin, there is **no CORS configuration
anywhere in this system**, and the session cookie is a plain same-origin cookie.

### Local development uses ports where production uses hostnames

| Production | Local | Role |
|---|---|---|
| `https://s.example.com/{code}` | `http://localhost:8080/{code}` | Redirect |
| `https://app.example.com/` | `http://localhost:8081/` | Dashboard |
| `https://app.example.com/api/v1/*` | `http://localhost:8081/api/v1/*` | API |

Same shape, same separation, same Caddy configuration. Deploying is two DNS records,
two tunnel ingress rules, and one environment variable — **no code changes.**

### The one rule that makes that true

The short base URL is **configuration, never derived from the request and never
hardcoded**:

```yaml
app:
  short-base-url: ${SHORT_BASE_URL:http://localhost:8080}
```

The API returns a fully-formed `shortUrl` built from this value. Building it from
`request.getServerName()` or assembling it in the frontend would both work locally and
both hand out `http://localhost:8080/aB3xY9z` to real users on the first day of
deployment. This is the single most common way a URL shortener breaks on first deploy.

## Request flow: Redirect

The hot path. Every step here is on the p99 < 20 ms budget.

```
GET s.example.com/aB3xY9z
  │
  ├─ 1. Rate limit check (Redis, per-IP)         ── over limit ─► 429 + Retry-After
  │
  ├─ 2. Cache lookup: code → LinkTarget (Redis)
  │        hit  ──────────────────────────────┐
  │        miss ─► 3. SELECT ... FROM links   │  single-row read by unique index,
  │                  WHERE code = ?           │  JdbcTemplate, not JPA
  │                  ─► populate cache        │
  │                                           │
  ├─ 4. Evaluate: ◄───────────────────────────┘
  │      status != ACTIVE  ─┐
  │      expires_at passed ─┼─► 404 (identical for all cases, incl. not found)
  │      deleted_at set    ─┘
  │
  ├─ 5. 302 Found
  │      Location: <destination>
  │      Cache-Control: private, no-cache
  │      X-Robots-Tag: noindex
  │
  └─ 6. Record the Click  ◄── after the response is decided, never before
```

**Step 6 can never affect steps 1–5.** It is called through a `ClickRecorder`
interface whose contract is *must not throw, must not block meaningfully*. Today its
implementation writes synchronously; see
[ADR-0005](./adr/0005-synchronous-click-recording.md) for why that is the right
starting point and exactly what replaces it.

### What the cache holds, and what it deliberately does not

The cached value is the minimum needed to answer a redirect:

```
code → { linkId, destination, status, expiresAt }
```

It does **not** hold the click count. A cached counter would be stale the moment
anyone clicked, and reconciling it against Postgres is a whole problem that buys
nothing. See [ADR-0004](./adr/0004-redis-is-cache-not-truth.md).

**Invalidation** is the price of a mutable Destination
([ADR-0009](./adr/0009-mutable-destination-with-audit.md)): every write to a Link —
destination change, disable, expiry change, delete — evicts `code` from the cache in
the same transaction boundary. TTL is 1 hour as a backstop, so the worst case of a
missed eviction is bounded rather than permanent.

## Request flow: Create Link

```
POST app.example.com/api/v1/links   { destination, alias?, expiresAt? }
  │
  ├─ 1. Authenticate (JWT from httpOnly cookie)   ── invalid ─► 401
  ├─ 2. Rate limit (per-IP and per-Owner)         ── over    ─► 429
  ├─ 3. Validate destination syntax               ── bad     ─► 422 INVALID_DESTINATION
  ├─ 4. DestinationScreener.screen(url)           ── refused ─► 422 DESTINATION_NOT_ALLOWED
  │        • scheme must be http/https
  │        • must not resolve to a Private Destination
  │        • must not be one of our own hostnames
  │
  ├─ 5a. Alias supplied?
  │        • reserved word? ─► 409 RESERVED_ALIAS
  │        • INSERT; unique violation ─► 409 ALIAS_TAKEN
  │
  ├─ 5b. No alias?
  │        • generate 7-char base62 from SecureRandom
  │        • INSERT; on unique violation, retry (max 3)
  │        • 3 failures ─► 500, and an alert-worthy metric
  │
  └─ 6. 201 Created  { id, code, shortUrl, destination, ... }
```

`DestinationScreener` is an interface with one implementation today
(`LocalRulesScreener`). It is the defined insertion point for external reputation
checks; see [ADR-0010](./adr/0010-defer-external-url-screening.md).

## Technology and why

| Choice | Why this, not the obvious alternative |
|---|---|
| **Java 21 + Spring Boot 3.4** | Fluency. The fastest stack to finish in beats the theoretically best one. |
| **Spring MVC + virtual threads** | Not WebFlux. The bottleneck is I/O concurrency, not thread memory — [ADR-0001](./adr/0001-mvc-virtual-threads-over-webflux.md) |
| **Postgres 16** | Single source of truth. Nothing here needs a document store or a wide-column store, and a unique constraint is the cheapest collision detector available. |
| **JPA for management, JdbcTemplate for redirect** | The redirect is one indexed row read. An entity manager, dirty checking and a persistence context are pure overhead on the one path with a latency budget. |
| **Flyway** | Migrations are versioned artefacts, applied identically on a laptop and in production. |
| **Redis 7** | Destination cache and distributed rate limiting only — [ADR-0004](./adr/0004-redis-is-cache-not-truth.md) |
| **Bucket4j** | Token bucket, Redis-backed, so limits survive a restart and would work across replicas. |
| **Caddy** | Static file serving plus an `/api` reverse proxy in ~8 lines, no TLS config needed since Cloudflare terminates. |
| **Vite + React + TS + Tailwind + shadcn/ui** | Static bundle; SSR would buy nothing here and would add a Node container. shadcn gives a credible dashboard in the ~3 hours the frontend gets. |
| **TanStack Query** | Server state has caching, retry and invalidation requirements that `useState` does not. |

## Deployment

One `docker-compose.yml`, four services:

| Service | Notes |
|---|---|
| `postgres` | named volume, healthcheck gating app start |
| `redis` | no persistence configured — losing it costs a cold cache, nothing more |
| `backend` | multi-stage Dockerfile, `SPRING_PROFILES_ACTIVE=prod` |
| `caddy` | serves the built SPA, proxies `/api` |

The tunnel already exists. Deployment adds two ingress rules and two DNS records:

```yaml
# ~/.cloudflared/config.yml — appended to the existing tunnel
ingress:
  - hostname: s.example.com
    service: http://localhost:8080
  - hostname: app.example.com
    service: http://localhost:8081
  # ... existing rules ...
  - service: http_status:404
```

### Three things that exist only in production

Handled from day one, because each is a two-line fix now and an hour of panic later:

1. **`CF-IPCountry` is absent locally.** Missing → store `UNKNOWN`, never throw.
2. **`Secure` cookies will not set over plain HTTP.** The flag is profile-dependent:
   off in `local`, on in `prod`. Otherwise login works locally and silently fails on
   deploy.
3. **The client IP is the socket address locally and `CF-Connecting-IP` in
   production.** One `ClientIpResolver` handles both — and trusts the header only from
   the tunnel, or every rate limit in [01-requirements.md](./01-requirements.md)
   becomes decorative.

**Deployment is the last task, and it is droppable.** If Friday runs out, the demo runs
from `localhost` and nothing about the design changes.
