---
status: accepted
date: 2026-09-03
---

# Two hostnames; the short host's root namespace is reserved for Short Codes

The system is served on two public hostnames: `s.example.com`, whose entire root path
namespace belongs to Short Codes and nothing else, and `app.example.com`, which serves
both the dashboard and the API under `/api/v1`.

## Considered options

**One hostname for everything.** `example.com/dashboard`, `example.com/login`,
`example.com/aB3xY9z`. Rejected because the application and its users then compete for
the same strings **permanently**: the day `/settings` becomes a page, no Owner can ever
hold the Alias `settings`, and every request needs a rule to decide whether a path is a
page or a Short Code. The Reserved Word list would grow with every feature shipped, and
each addition would be a breaking change for whoever already owned that Alias.

**Three hostnames**, splitting the API onto `api.example.com`. Rejected because it buys
nothing at this size and costs two things immediately: a CORS configuration to get
right, and cross-site cookie rules that would push the session token out of an
`httpOnly` cookie and into `localStorage`. Sharing one origin between the dashboard and
the API means **there is no CORS configuration anywhere in this system**, and the
session cookie is an ordinary same-origin cookie.

## Consequences

- The Reserved Word list ([04-data-model.md](../04-data-model.md)) exists to protect
  the short host's few genuine needs (`/`, `favicon.ico`, `robots.txt`), not to protect
  application routes — because there are none on that host.
- Locally, **ports play the role hostnames play in production**: `:8080` is the short
  host, `:8081` is the app host. Same separation, same Caddy config, so deploying
  changes two DNS records and one environment variable and no code. See
  [03-architecture.md](../03-architecture.md).
- The short base URL must therefore come from configuration and never from the incoming
  request or the frontend. This is the constraint that makes local-first development
  safe, and the most common way a URL shortener breaks on its first deploy.
- Splitting the API onto its own hostname later is a Caddy change plus a CORS block —
  genuinely reversible, which is exactly why it should not cost time now.
- This decision is scoped to one shared short domain. Per-Owner custom domains
  ([R-9](../06-roadmap.md)) would make "the root namespace belongs to Short Codes" a
  per-tenant statement and change the uniqueness of `links.code` itself.
