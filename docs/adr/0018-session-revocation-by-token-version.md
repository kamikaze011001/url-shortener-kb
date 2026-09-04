---
status: accepted
date: 2026-09-04
---

# Sessions are revoked by a token version, cached in Redis

Every JWT carries the Owner's `token_version` as a claim. Every authenticated request
compares the claim against the current value. A password reset increments the column,
and every token issued before that instant stops matching.

This is roadmap item **R-1**, arriving early because password reset made it necessary:
without it, an Owner resetting a compromised password leaves the attacker's session
alive for up to an hour, and the security feature is half theatre.

## The objection, and why it does not hold here

Checking a version per request means a lookup per request, and the reflex objection is
that this costs latency on the hot path.

**It does not touch the hot path.** The redirect is unauthenticated —
`SecurityConfig` permits `GET /{code}` outright — so the check never runs there. It runs
only on `/api/v1/**`, where the budgets in [02-nfr.md](../02-nfr.md) are p95 < 200 ms
for creation and < 300 ms for listing, not the redirect's 20 ms p99.

A warm Redis read is a fraction of a millisecond against a 200 ms budget, on a request
that already makes several Postgres round trips. The cost is real and it is
approximately nothing.

## Considered options

**Postgres on every authenticated request.** Correct and simplest; one indexed primary
key lookup. Rejected only because the Redis cache below is nearly free and this is the
kind of read that is worth not repeating.

**Short-lived access tokens plus refresh tokens.** The architecturally superior answer:
revocation delay becomes the access-token lifetime and there is no per-request check at
all. Deferred because R-1 already warns that rotation with reuse detection is *"more
work than it appears"*, and because it replaces a small known cost with a larger unknown
one. This remains the direction of travel.

**A denylist of revoked token ids.** Same per-request lookup, plus the problem of
knowing when an entry may be dropped. A version counter needs no expiry policy.

## Consequences

- **Redis caches the row; Postgres owns it.** A cache miss, or Redis being down, falls
  through to Postgres and answers correctly — slower, never wrong. That is
  [ADR-0004](./0004-redis-is-cache-not-truth.md) applied to a new reader rather than a
  new rule.
- **Revocation is all-or-nothing per Owner.** "Log out everywhere" is free; "log out of
  that one laptop" is not possible, because nothing distinguishes one token from
  another. Per-device control needs a session table, which is the same work as refresh
  tokens.
- **`email_verified` rides in the same cached row.** Verification status is then read
  per request at no extra cost and can never be stale, which is what lets
  [ADR-0016](./0016-verification-gates-creation.md) avoid forcing a re-login after
  verifying.
- **Changing a password now logs the Owner out of their own other sessions.** That is
  the intended behaviour and it is worth saying in the interface, because a user who is
  not told will read it as a bug.
