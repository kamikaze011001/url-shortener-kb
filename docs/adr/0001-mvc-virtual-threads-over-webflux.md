---
status: accepted
date: 2026-09-03
---

# Spring MVC with virtual threads, not WebFlux

The redirect path must hold many concurrent, short, I/O-bound requests. We use
**Spring MVC with virtual threads enabled** (`spring.threads.virtual.enabled=true` on
Java 21) rather than WebFlux, because the constraint is *concurrency*, not throughput
per core: each redirect blocks briefly on Redis or Postgres and then ends. A virtual
thread parking on that block costs a few hundred bytes instead of a platform thread's
megabyte stack, which removes the thread-pool ceiling without changing how the code is
written.

## Considered options

**WebFlux + R2DBC.** Better numbers in benchmarks, and the reflex answer for a
"high-throughput redirect service". Rejected because the cost lands entirely on the
parts of this project that are scarcest: every repository becomes reactive, every
blocking library has to be replaced or wrapped, stack traces stop being readable, and
blocking
accidentally anywhere in the chain silently destroys the benefit. For a solo build with
a ~14-hour budget, that is a large risk against a bottleneck we do not have.

**Plain MVC on a platform thread pool.** Entirely adequate at the Real scale in
[02-nfr.md](../02-nfr.md) — a few RPS. Rejected because the ceiling arrives at a few
hundred concurrent requests and the fix costs one configuration line, so declining to
take it would be perverse.

## Consequences

- Repository code stays imperative and blocking. JDBC, JPA and the Redis client all
  work normally. This is the point, not a compromise.
- `synchronized` blocks around I/O would pin a virtual thread to its carrier and undo
  the benefit. Any lock on a request path must be a `ReentrantLock`. This is the one
  new rule the decision introduces, and the one way to get it wrong.
- The JDBC connection pool becomes the real concurrency limit, since virtual threads
  are effectively unbounded. Pool sizing is now a deliberate capacity decision rather
  than an afterthought.
- Reversing this later means rewriting every repository. It is recorded here because a
  future reader will otherwise assume WebFlux was never considered.
