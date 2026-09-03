# 03 — Architecture

## Shape

One Spring Boot application, one Postgres, one Redis, one Caddy. **A single deployable
backend**, not microservices — at this scale, splitting the redirect service from the
management API would buy independent scaling nobody needs and cost a network hop on
the one path that matters.

The split is real, but it lives at the **module** level inside one process, so that
extracting it later is a refactor rather than a rewrite. Modules are enforced by
Spring Modulith, not merely described here — see
[ADR-0012](./adr/0012-modulith-verified-boundaries.md).

```
com.<org>.urlshortener
├── shared/                    error model, ids, clock, ClientIpResolver
├── identity/                  Owners, registration, login, session
│   ├── RegisterOwnerUseCase       ← module API
│   ├── LoginUseCase
│   └── internal/                  ← invisible to other modules
│       ├── OwnerRepository, OwnerEntity, JwtIssuer
│       └── web/                   ← controllers
├── links/                     create, edit, disable, delete. Owns the Link.
│   ├── CreateLinkUseCase, UpdateLinkUseCase, DeleteLinkUseCase, ListLinksUseCase
│   ├── LinkLookup                 ← the port redirect is allowed to use
│   └── internal/
├── redirect/                  the hot path
│   ├── ResolveShortCodeUseCase
│   └── internal/
└── analytics/                 recording Clicks, reading statistics
    ├── ClickRecorder              ← the seam from ADR-0005
    ├── GetLinkStatsUseCase
    └── internal/
```

`redirect` declares `@ApplicationModule(allowedDependencies = {"links", "shared"})`
and reaches `links` only through the `LinkLookup` port. It deliberately does **not**
query the `links` table itself: doing so would let it claim zero code dependencies
while carrying a hidden data dependency, which is worse, not better. The port is the
seam along which the service would be split if the redirect path ever needed to scale
independently — extraction turns a method call into a network call rather than a
rewrite.

Business logic lives in **one class per use case**, not in a service layer. Each has a
nested `Command` and `Result` record and a single `execute` method; controllers map
HTTP to a `Command` and back, and contain nothing else. See
[ADR-0011](./adr/0011-one-class-per-use-case.md).

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
| **Java 21 + Spring Boot 4.1** | Fluency. The fastest stack to finish in beats the theoretically best one. Boot **4** rather than the 3.4 originally planned: Spring Initializr no longer offers any 3.x release, so this was chosen for us. It brings Spring Security 7, whose configuration API differs from 3.x. |
| **Spring MVC + virtual threads** | Not WebFlux. The bottleneck is I/O concurrency, not thread memory — [ADR-0001](./adr/0001-mvc-virtual-threads-over-webflux.md) |
| **Spring Modulith (`-core`, `-test`)** | Module boundaries fail the build instead of rotting in a document — [ADR-0012](./adr/0012-modulith-verified-boundaries.md). Event registry deliberately excluded. |
| **One class per use case** | Not a service layer — [ADR-0011](./adr/0011-one-class-per-use-case.md) |
| **Micrometer Tracing (otel bridge), no exporter** | Standard, propagating `traceId` in MDC for the same effort as a hand-rolled request id |
| **logstash-logback-encoder** | Logs queried by field, not by regex |
| **Postgres 16** | Single source of truth. Nothing here needs a document store or a wide-column store, and a unique constraint is the cheapest collision detector available. |
| **JPA for management, JdbcTemplate for redirect** | The redirect is one indexed row read. An entity manager, dirty checking and a persistence context are pure overhead on the one path with a latency budget. |
| **Flyway** | Migrations are versioned artefacts, applied identically on a laptop and in production. |
| **Redis 7** | Destination cache and distributed rate limiting only — [ADR-0004](./adr/0004-redis-is-cache-not-truth.md) |
| **Bucket4j** | Token bucket, Redis-backed, so limits survive a restart and would work across replicas. |
| **Caddy** | Static file serving plus an `/api` reverse proxy in ~8 lines, no TLS config needed since Cloudflare terminates. |
| **Vite + React + TS + Tailwind + shadcn/ui** | Static bundle; SSR would buy nothing here and would add a Node container. shadcn gives a credible dashboard in the ~3 hours the frontend gets. |
| **TanStack Query** | Server state has caching, retry and invalidation requirements that `useState` does not. |

## Observability and logging

The requirement is that a **complete business flow can be reconstructed from the
logs** — not that everything is logged. Undisciplined logging produces thousands of
lines nobody reads and is indistinguishable from no logging at all. Four pieces:

### Correlation comes from Micrometer Tracing, not a hand-rolled request id

`micrometer-tracing-bridge-otel`, with **no exporter configured** — no Jaeger, no
Zipkin, no extra container. `traceId` and `spanId` land in MDC automatically, in the
standard format, propagated correctly across any future service boundary. Adding an
exporter later is configuration, not a change to any log statement.

Generating a UUID per request would be the same effort for a non-standard,
non-propagating result.

Behind Cloudflare, the `CF-Ray` header is also recorded as a field. It is the join key
between these logs and Cloudflare's own — the only way to answer "the edge saw it, did
we?" once deployed.

### Structured JSON, with domain fields in MDC

`logstash-logback-encoder`. A filter puts `ownerId` and, where known, `linkId` and
`code` into MDC, so every line inside a request carries them without being passed
around. Logs are queried by field, never by regular expression.

### One deliberate business event per meaningful outcome

This is what makes a flow traceable, as opposed to an access log with extra steps.
Each is a single INFO line with a stable `event` field:

| `event` | Fields |
|---|---|
| `link.created` | `code`, `isCustomAlias`, `ownerId` |
| `link.create_rejected` | `reason` = `DESTINATION_NOT_ALLOWED` \| `ALIAS_TAKEN` \| `RESERVED_ALIAS` |
| `link.redirected` | `code`, `linkId`, `cacheHit` |
| `link.redirect_missed` | `code`, `reason` = `NOT_FOUND` \| `DISABLED` \| `EXPIRED` \| `DELETED` |
| `link.destination_changed` | `linkId`, `from`, `to` |
| `auth.login_failed` | `email` |
| `ratelimit.rejected` | `bucket`, `limit` |
| `code.collision_retry` | `attempt` |

`link.redirect_missed` carries the **real** reason while the HTTP response stays a
uniform `404`. That is the debuggability cost accepted by
[ADR-0008](./adr/0008-soft-delete-and-uniform-404.md), repaid in the one place where
it is safe to repay it.

### A single aspect around every use case

An `@Around` advice on `*UseCase.execute(..)` logs entry, outcome and duration, and
puts the use-case name into MDC. About thirty lines, and every use case is
instrumented without one hand-written log statement.

This works only because every use case has the same shape — an unplanned return on
[ADR-0011](./adr/0011-one-class-per-use-case.md). A service layer of fifteen
differently-shaped methods could not be instrumented this way.

### Never logged

Follows directly from [02-nfr.md § Privacy](./02-nfr.md): passwords, JWTs, the
`Cookie` header, and **raw IP addresses**. Promising that IPs are never stored and then
writing them to a log file would be a distinction without a difference. Where
correlation is needed, the first 8 characters of `ip_hash` are logged instead.

### Now versus later

One INFO line per Redirect is right at the Real scale and is ~2,000 lines/second at the
Paper scale. At that point redirect logging is sampled or dropped to DEBUG, and the
business events above become metrics rather than lines. Tracked as
[R-11](./06-roadmap.md).

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
