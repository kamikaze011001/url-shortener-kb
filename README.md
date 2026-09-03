# URL Shortener — Knowledge Base

The **single source of truth** for the URL Shortener project. Backend and frontend
implementations are derived from this repository; where an implementation disagrees
with this repository, **the implementation is wrong**.

## Repositories

| Repo | Role |
|---|---|
| [`url-shortener-kb`](https://github.com/kamikaze011001/url-shortener-kb) | This repo. Requirements, design, contract, decisions. |
| [`url-shortener-backend`](https://github.com/kamikaze011001/url-shortener-backend) | Java 21 / Spring Boot 4.1 API + redirect service. |
| [`url-shortener-frontend`](https://github.com/kamikaze011001/url-shortener-frontend) | Vite + React + TypeScript dashboard. |

## How to navigate

Read in order. Each document assumes the previous one.

| # | Document | What it answers |
|---|---|---|
| — | [CONTEXT.md](./CONTEXT.md) | What do the words mean? (glossary — no implementation detail) |
| 01 | [Requirements](./docs/01-requirements.md) | What must it do, and explicitly not do? |
| 02 | [Non-functional requirements](./docs/02-nfr.md) | How fast, how big, how safe — with numbers |
| 03 | [Architecture](./docs/03-architecture.md) | What are the pieces, and how does a request flow? |
| 04 | [Data model](./docs/04-data-model.md) | What is stored, and under what constraints? |
| 05 | [API contract](./docs/05-api-contract.md) + [`openapi.yaml`](./docs/openapi.yaml) | The exact wire format |
| 06 | [Roadmap](./docs/06-roadmap.md) | What changes in the future, and what triggers each change |
| — | [ADRs](./docs/adr/) | Why each hard-to-reverse decision was made |

## Decision record index

Every entry below was a genuine trade-off with a rejected alternative.

| ADR | Decision |
|---|---|
| [0001](./docs/adr/0001-mvc-virtual-threads-over-webflux.md) | Spring MVC + virtual threads, not WebFlux |
| [0002](./docs/adr/0002-random-base62-short-codes.md) | Random base62 codes, not a monotonic counter |
| [0003](./docs/adr/0003-302-not-301.md) | 302 temporary redirect, not 301 permanent |
| [0004](./docs/adr/0004-redis-is-cache-not-truth.md) | Redis is a cache and rate limiter, never a source of truth |
| [0005](./docs/adr/0005-synchronous-click-recording.md) | Synchronous click recording behind a `ClickRecorder` seam |
| [0006](./docs/adr/0006-two-hostname-topology.md) | Two hostnames; the short domain's root namespace is reserved |
| [0007](./docs/adr/0007-handwritten-openapi-is-the-contract.md) | Hand-written OpenAPI is the contract; generated spec is not |
| [0008](./docs/adr/0008-soft-delete-and-uniform-404.md) | Soft delete, codes never recycled, uniform 404 |
| [0009](./docs/adr/0009-mutable-destination-with-audit.md) | Destinations are mutable, with an audit trail |
| [0010](./docs/adr/0010-defer-external-url-screening.md) | Built-in SSRF guard now; external URL screening deferred behind an interface |
| [0011](./docs/adr/0011-one-class-per-use-case.md) | One class per use case; no service layer |
| [0012](./docs/adr/0012-modulith-verified-boundaries.md) | Module boundaries verified by Spring Modulith, not asserted |

## Project constraints

This is a **course demo** built solo in roughly 12–16 hours, presented **2026-09-05**.
Scope decisions throughout are shaped by that budget, and every place where the demo
budget forced a compromise is labelled **Demo compromise** rather than hidden.

The grading emphasis is *design reasoning* — decisions, changes, and trade-offs, now
versus in the future. The working application is the evidence, not the point.
