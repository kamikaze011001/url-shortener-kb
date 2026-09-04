---
status: accepted
date: 2026-09-04
---

# A fixed-window counter in Redis, not a token bucket

[FR-6](../01-requirements.md) is enforced by a **fixed-window counter per key**, held in
Redis and incremented by an eleven-line Lua script. Not Bucket4j, which earlier drafts of
[03-architecture.md](../03-architecture.md) named.

```lua
local count = redis.call('INCR', KEYS[1])
if count == 1 then
  redis.call('PEXPIRE', KEYS[1], ARGV[1])
end
return { count, redis.call('PTTL', KEYS[1]) }
```

The increment and the expiry are one script because they must be atomic. As two round
trips, a process that died between them would leave a key with **no TTL**, and that key
would reject its client forever.

## What the fixed window costs

A window opens on the first request and closes exactly one window later. So a client that
spends its whole allowance at 11:59:59 and its next allowance at 12:00:01 gets **twice the
limit through in two seconds**.

That burst is stated here rather than buried because it is the entire reason to prefer a
token bucket, and pretending it does not exist would make this ADR marketing. It is
accepted because the limits in FR-6 exist to stop scripted credential stuffing and flood
traffic, both of which are orders of magnitude above these ceilings. A burst of 10 logins
in two seconds is not the attack; 10,000 in a minute is, and that is caught either way.

## Considered options

**Bucket4j over Redis.** The standard answer, smooth refill, no boundary burst. Rejected
for now on dependency cost, not on merit: it is a new library to pin and a Lettuce
integration to wire the night before the demo, to remove a burst that is not the abuse
being defended against. It remains the right upgrade, and the abstraction was shaped so
that taking it is a one-class change — a key, a policy, a verdict.

**A sliding window log** (a sorted set of timestamps per key). Exact, no burst, and costs
memory proportional to requests rather than to keys. Rejected because it makes the
redirect path's memory use a function of traffic, which is the wrong shape for the one
path with a latency budget.

**In-memory counters.** Simplest, and wrong: limits would reset on every restart and
would be per-instance, so the horizontal scaling in row 4 of "what breaks first"
([02-nfr.md](../02-nfr.md)) would silently multiply every limit by the replica count.

## Consequences

- **The limiter fails open.** If Redis is unreachable the request proceeds and a warning
  is logged. A rate limiter that takes the service down when its cache blinks has done
  more damage than the abuse it prevents — and on the redirect path, where the Visitor is
  not our user and cannot meaningfully retry, the trade is not close. This is the same
  posture [02-nfr.md](../02-nfr.md) already states under Availability.
- **The bucket key is the salted IP hash, not the IP.** Stable enough to count against,
  useless to anyone who reads the Redis keyspace, and consistent with the rule that raw
  addresses are never stored (02-nfr.md § Privacy).
- **Layered limits each consume their allowance even when a later one rejects.** A Link
  creation is checked per-IP and per-Owner; if the per-Owner check fails, the per-IP
  token is still spent. That is intended — the per-IP counter measures what the IP asked
  for, not what it was granted.
- **The redirect path now touches Redis on every request.** It already did, for the
  destination cache, so this adds a second command rather than a second dependency. If it
  ever shows up in the p99, the two can be combined into one script.
- **Registration shares the login limit.** FR-6.1 names login only, but the contract
  declares `429` on registration too and bulk account creation is at least as abusable.
