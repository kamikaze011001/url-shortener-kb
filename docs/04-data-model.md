# 04 — Data model

Postgres 16. Migrations are Flyway, in `url-shortener-backend/src/main/resources/db/migration`.
**This document is the specification; the migration implements it.**

## Diagram

```
                  ┌──────────────┐
      ┌───1:N─────│   owners     │─────1:N───┐
      │           └──────┬───────┘           │
      │                  │ 1:N               │
┌─────▼──────┐    ┌──────▼───────────────┐  ┌▼───────────┐
│ otp_codes  │    │        links         │  │  api_keys  │
└────────────┘    └──────────┬───────────┘  └────────────┘
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
    id             uuid        PRIMARY KEY DEFAULT gen_random_uuid(),
    email          text        NOT NULL,
    password_hash  text        NOT NULL,
    email_verified boolean     NOT NULL DEFAULT false,
    token_version  integer     NOT NULL DEFAULT 0,
    created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX owners_email_lower_key ON owners (lower(email));
```

**`token_version` is how a session dies before it expires.** Every JWT carries the
version that was current when it was issued; every authenticated request compares the
claim against this column. A password reset increments it, and every token issued
before that moment stops matching (FR-1.10,
[ADR-0018](./adr/0018-session-revocation-by-token-version.md)).

It is an `integer` and it only ever goes up. Nothing reads its absolute value, so
wraparound is not a correctness concern before it is an archaeological one.

**`email_verified` is backfilled to `true` for rows that already exist.** Accounts
created before verification existed were never verified under the old rules either, and
retroactively locking them out buys no security while breaking working accounts. New
registrations start `false`.

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

## `otp_codes`

One row per outstanding code, for both verification and password reset.

```sql
CREATE TABLE otp_codes (
    id          bigserial   PRIMARY KEY,
    owner_id    uuid        NOT NULL REFERENCES owners(id) ON DELETE CASCADE,
    purpose     varchar(24) NOT NULL,
    code_hash   char(64)    NOT NULL,
    attempts    smallint    NOT NULL DEFAULT 0,
    expires_at  timestamptz NOT NULL,
    consumed_at timestamptz NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),

    CONSTRAINT otp_purpose_check
        CHECK (purpose IN ('EMAIL_VERIFICATION', 'PASSWORD_RESET'))
);

-- The lookup: the newest live code for this Owner and purpose.
CREATE INDEX otp_codes_owner_purpose_idx
    ON otp_codes (owner_id, purpose, created_at DESC)
    WHERE consumed_at IS NULL;
```

**Postgres, not Redis, and that follows from a rule already written down.**
[ADR-0004](./adr/0004-redis-is-cache-not-truth.md) says Redis holds a cache and rate
limits and is *never a source of truth*. A code that authorises a password change **is**
a source of truth — losing it must not silently grant or deny access, and a Redis
restart must not invalidate every outstanding reset in flight. See
[ADR-0017](./adr/0017-otp-codes-in-postgres.md).

**`code_hash`, never the code.** Same argument as FR-1.5 for passwords: a database leak
must not hand over a set of live reset codes. SHA-256 rather than bcrypt is defensible
here only because the code is machine-generated — see the same ADR for why that
reasoning does *not* transfer to passwords.

**`attempts` is on the row, not in Redis,** so the 5-attempt limit survives a cache
restart. A brute-force limit that resets when a container does is not a limit.

`consumed_at` rather than deleting the row: a consumed code is evidence, and the window
where the same code is submitted twice is exactly the window worth being able to see.

## `api_keys`

```sql
CREATE TABLE api_keys (
    id           bigserial   PRIMARY KEY,
    owner_id     uuid        NOT NULL REFERENCES owners(id) ON DELETE CASCADE,
    name         varchar(64) NOT NULL,
    key_hash     char(64)    NOT NULL,
    key_prefix   varchar(16) NOT NULL,
    key_last4    char(4)     NOT NULL,
    scopes       text[]      NOT NULL,
    expires_at   timestamptz NULL,
    last_used_at timestamptz NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    revoked_at   timestamptz NULL,

    CONSTRAINT api_keys_scopes_known CHECK (
        cardinality(scopes) > 0
        AND scopes <@ ARRAY['links:read', 'links:write']
    )
);

-- The authentication lookup, on every keyed request.
CREATE UNIQUE INDEX api_keys_hash_key ON api_keys (key_hash);

CREATE INDEX api_keys_owner_idx
    ON api_keys (owner_id, created_at DESC)
    WHERE revoked_at IS NULL;
```

**The hash is the lookup key, so authentication is one indexed read** — hash what
arrived, find the row, or answer 401. There is no scan and no per-row comparison, which
is the reason SHA-256 rather than bcrypt: a per-row bcrypt comparison cannot use an
index at all. [ADR-0019](./adr/0019-api-key-authentication.md) explains why that is
safe for a 256-bit random key and would not be for a password.

**`scopes` is an array with a `CHECK`, not a join table.** The set is small and fixed,
it is always read with the key, and it is never queried across keys — the normalised
shape would buy nothing. The constraint is what makes an unknown scope unstorable, even
by a hand-written `INSERT` that skips the application.

**`expires_at` is checked in the authentication query**, not after it:
`AND (expires_at IS NULL OR expires_at > now)`. It costs nothing on a lookup that was
already one indexed read, and no code path can end up holding an expired key and
forgetting to look. `NULL` means never — which is what every key created before
[ADR-0020](./adr/0020-api-key-scopes-and-expiry.md) was backfilled to, along with every
scope, because a migration must never narrow the authority of a credential that is
already in use.

**Expired keys are still listed.** The list filters on `revoked_at IS NULL` only. A
revoked key was ended by a person who knows they ended it; an expired key ended on its
own, and its Owner is the one debugging why a script stopped working (FR-8.11).

**`key_prefix` and `key_last4` exist because the plaintext is gone.** An Owner with
three keys has to be able to tell which one to revoke, and `sk_live_8f2a…c3d4` is
enough to recognise a key without being enough to use one.

`last_used_at` answers the only question that matters before revoking something: *is
anything still using this?*

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
