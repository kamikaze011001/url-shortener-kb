# 04 — Data model

Postgres 16. Migrations are Flyway, in `url-shortener-backend/src/main/resources/db/migration`.
**This document is the specification; the migration implements it.**

## Diagram

```
┌──────────────┐         ┌────────────────────────────┐
│   owners     │────1:N──│           links            │
└──────────────┘         └────────────┬───────────────┘
                                      │
                          ┌───────────┴───────────┐
                          │ 1:N                   │ 1:N
              ┌───────────▼──────────┐  ┌─────────▼────────────────┐
              │ link_destination_    │  │      click_events        │
              │      history         │  │                          │
              └──────────────────────┘  └──────────────────────────┘
```

## `owners`

```sql
CREATE TABLE owners (
    id            uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    email         text        NOT NULL,
    password_hash text        NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX owners_email_lower_key ON owners (lower(email));
```

**Email uniqueness is case-insensitive** via a functional index, while the original
casing is preserved for display. `Sonanh@Example.com` and `sonanh@example.com` are the
same person; storing the typed form and comparing on `lower()` gets both properties
without a `citext` extension.

`id` is a UUID, not a sequence: an Owner id appears in JWT claims, and a sequential id
there would leak how many people have registered.

## `links`

The core table. Every constraint below exists to enforce something in
[02-nfr.md § Correctness properties](./02-nfr.md).

```sql
CREATE TABLE links (
    id              bigserial   PRIMARY KEY,
    code            varchar(32) NOT NULL,
    destination     text        NOT NULL,
    owner_id        uuid        NOT NULL REFERENCES owners(id),
    is_custom_alias boolean     NOT NULL DEFAULT false,
    status          varchar(16) NOT NULL DEFAULT 'ACTIVE',
    click_count     bigint      NOT NULL DEFAULT 0,
    expires_at      timestamptz NULL,
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now(),
    deleted_at      timestamptz NULL,

    CONSTRAINT links_status_check
        CHECK (status IN ('ACTIVE', 'DISABLED')),
    CONSTRAINT links_destination_scheme_check
        CHECK (destination ~* '^https?://')
);

-- The Code Namespace. Not partial: a deleted Link still holds its code forever.
CREATE UNIQUE INDEX links_code_key ON links (code);

-- Dashboard listing: an Owner's live links, newest first.
CREATE INDEX links_owner_created_idx
    ON links (owner_id, created_at DESC)
    WHERE deleted_at IS NULL;
```

### Decisions embedded in this table

**`links_code_key` is not a partial index, and that is the point.** Making it
`WHERE deleted_at IS NULL` would let a deleted code be claimed again — the exact
hazard [ADR-0008](./adr/0008-soft-delete-and-uniform-404.md) exists to prevent. The
Code Namespace is append-only. The index is also the collision detector for
[ADR-0002](./adr/0002-random-base62-short-codes.md): generation does not check
first, it inserts and handles the unique violation.

**`code` is `varchar(32)` and case-sensitive.** Generated codes are 7 characters;
Aliases go to 32. Default collation is byte-comparison, so `AbC` and `abc` are
different codes. That doubles the keyspace and makes the unique index cheap, at the
cost of confusion when a URL is read aloud or printed. Accepted, and noted here so
nobody "fixes" it into a lowercase index and silently collides two existing links.

**Only two statuses are stored.** `ACTIVE` and `DISABLED` are state; *Expired* and
*Deleted* are derived from `expires_at` and `deleted_at`. A Link is redirectable when:

```sql
status = 'ACTIVE' AND deleted_at IS NULL AND (expires_at IS NULL OR expires_at > now())
```

Storing `EXPIRED` as a status would require a job to write it and would be wrong for
the window between expiry and that job running. Derived state cannot drift.

**`click_count` is a denormalised counter, updated in the same transaction as the
click event insert.** It is a denormalisation, not a second source of truth — both
writes commit together, so they cannot disagree. It exists so the dashboard never
runs an aggregate query. It is also **the first thing that breaks under load**: every
concurrent Click on one popular Link contends on this single row. That is documented,
expected, and the trigger for the change in
[ADR-0005](./adr/0005-synchronous-click-recording.md).

**There is no `max_clicks`.** Enforcing it would require reading a live counter on
every redirect, which defeats the destination cache — the cached value would be stale
the moment anyone clicked. Time-based expiry has no such cost because `expires_at` is
immutable between writes and can be cached safely. See
[01-requirements.md § Explicit non-goals](./01-requirements.md).

## `link_destination_history`

```sql
CREATE TABLE link_destination_history (
    id              bigserial   PRIMARY KEY,
    link_id         bigint      NOT NULL REFERENCES links(id),
    old_destination text        NOT NULL,
    new_destination text        NOT NULL,
    changed_by      uuid        NOT NULL REFERENCES owners(id),
    changed_at      timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ldh_link_idx ON link_destination_history (link_id, changed_at DESC);
```

Append-only, written in the same transaction as the `links.destination` update.
This table is what makes a mutable Destination defensible rather than a loophole:
"someone could bait-and-switch a shared link" is answered with *and every switch is
recorded, with who and when*. See
[ADR-0009](./adr/0009-mutable-destination-with-audit.md).

`changed_by` is retained separately from `links.owner_id` because ownership transfer,
though out of scope today, would otherwise silently rewrite history.

## `click_events`

```sql
CREATE TABLE click_events (
    id           bigserial   PRIMARY KEY,
    link_id      bigint      NOT NULL REFERENCES links(id),
    occurred_at  timestamptz NOT NULL DEFAULT now(),
    referrer     text        NULL,
    user_agent   varchar(512) NULL,
    country_code char(2)     NOT NULL DEFAULT 'XX',
    device_type  varchar(16) NOT NULL DEFAULT 'UNKNOWN',
    ip_hash      char(64)    NULL,

    CONSTRAINT click_device_check
        CHECK (device_type IN ('DESKTOP', 'MOBILE', 'TABLET', 'BOT', 'UNKNOWN'))
);

CREATE INDEX click_events_link_time_idx ON click_events (link_id, occurred_at DESC);
```

**No raw IP address column exists, and none will be added.** `ip_hash` is
`SHA-256(ip + application_salt)` — enough to spot the same visitor twice, useless for
identifying who they are, and harmless in a database leak. This is the single most
consequential privacy decision in the schema and it costs one function call. See
[02-nfr.md § Privacy](./02-nfr.md).

`country_code` defaults to `'XX'` because `CF-IPCountry` does not exist outside
production. The absence of a header is normal, not an error.

**This table grows without bound**, at roughly 60 GB/month at Paper scale. It is
append-only and safe to prune, which is the whole reason statistics are defined as
approximate. The rollup that makes pruning possible is
[R-3 in the roadmap](./06-roadmap.md); it is deliberately not built yet, because at
demo scale `GROUP BY` over a few thousand rows is instant.

## Reserved words

Enforced in application code at Alias creation, not by a database constraint — the
list will change, and a migration per word is absurd. Rejected with `409 RESERVED_ALIAS`.

```
api, app, admin, auth, login, logout, register, signup, signin,
dashboard, settings, account, health, status, metrics, actuator,
static, assets, public, favicon.ico, robots.txt, sitemap.xml,
_next, .well-known, s, www, help, about, terms, privacy, support
```

Comparison is **case-insensitive** for this check only: `Admin` is as reserved as
`admin`, even though the two would otherwise be distinct Short Codes. Blocking a
reserved word is about preventing confusion, not about namespace mechanics.

## Retention

| Data | Kept | Why |
|---|---|---|
| `owners` | forever | no account deletion flow in scope |
| `links` (incl. soft-deleted) | **forever** | deleting a row would release its code — see [ADR-0008](./adr/0008-soft-delete-and-uniform-404.md) |
| `link_destination_history` | forever | an audit trail that expires is not one |
| `click_events` | forever *today*, 90 days once rollups exist | the only table where retention is a real question |
