---
status: accepted
date: 2026-09-03
---

# Redis is a cache and a rate limiter, never a source of truth

Redis holds exactly two things: the destination cache
(`code → {linkId, destination, status, expiresAt}`, 1 hour TTL) and Bucket4j's rate
limit buckets. **Postgres is the only source of truth.** Losing Redis entirely costs a
cold cache and temporarily unenforced rate limits; it never costs data and never
returns a wrong answer.

The rule that follows from this, and the reason the ADR exists: **Click counters do
not go in Redis**, even though that is the standard advice for this exact problem. A
counter in Redis is a second source of truth that will drift from Postgres, and
reconciling the two is a whole subsystem — write-behind flushing, crash recovery,
double-counting on retry — bought in exchange for saving one indexed `UPDATE` that,
at the Real scale in [02-nfr.md](../02-nfr.md), costs well under a millisecond.

## Considered options

**No Redis at all**, with a Caffeine in-process cache and in-memory rate limits. Very
nearly the right answer: at 1,000 Links, Postgres serves every redirect from its own
buffer cache in under a millisecond, so the cache saves nothing measurable. Rejected
for two specific reasons rather than reflex — in-memory rate limits reset on every
restart and are per-instance, so they stop being limits the moment there are two
replicas ([R-6](../06-roadmap.md)); and the destination cache is where
[ADR-0009](./0009-mutable-destination-with-audit.md)'s invalidation problem becomes
visible, which is worth having in the design.

**Redis for everything, including counters.** Rejected above.

## Consequences

- **Cache invalidation is now a real obligation.** Every write to a Link — destination
  change, disable, expiry change, delete — must evict its `code`. The 1-hour TTL is a
  backstop so a missed eviction is bounded rather than permanent, not a substitute for
  evicting.
- **Rate limiting fails open.** If Redis is unreachable, requests are allowed rather
  than rejected. Deliberate: an unavailable rate limiter must not take down redirects.
  It does mean an attacker who can knock Redis over also disables the limiter.
- The cached value deliberately excludes `clickCount`, which would be stale the instant
  anyone clicked.
- Redis is configured without persistence. There is nothing in it worth recovering.
