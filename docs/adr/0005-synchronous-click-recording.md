---
status: accepted
date: 2026-09-03
---

# Synchronous Click recording, behind a `ClickRecorder` seam

A Redirect records its Click **synchronously**: one `INSERT INTO click_events` plus one
`UPDATE links SET click_count = click_count + 1`, in the same transaction, after the
response has been decided. At the Real scale in [02-nfr.md](../02-nfr.md) this costs
well under a millisecond, and an asynchronous pipeline would be optimising a problem
that does not exist.

The decision worth recording is not "synchronous" — it is that the redirect path calls
an interface it does not own:

```java
public interface ClickRecorder {
    /** Must not throw. Must not meaningfully block. */
    void record(ClickEvent event);
}
```

Today the implementation is `SyncClickRecorder`. That interface is one file, costs
nothing, and is the entire reason [R-4](../06-roadmap.md) is a swap rather than a
rewrite.

## Considered options

**Asynchronous batching from day one** — bounded queue, batch insert, drop on full.
Rejected as premature: it is not needed at this scale, and presenting it as necessary
would be indefensible. It is fully specified in [R-4](../06-roadmap.md), with the
trigger that makes it necessary.

**Increment only, no event rows.** Cheapest possible, and it makes any statistics
beyond a total count impossible — including the per-day chart in
[FR-5.2](../01-requirements.md). Rejected: the counter is the denormalisation, the
event rows are the data.

## Consequences

- **The `UPDATE ... click_count` is the first thing that will break under load**, and
  we know it. Every concurrent Click on one popular Link contends on a single row.
  Named as row 1 in [02-nfr.md § What breaks first](../02-nfr.md), triggered at roughly
  50 concurrent clicks/second on one Link.
- The `must not throw` clause is a **correctness** property, not a performance one:
  [02-nfr.md](../02-nfr.md) requires that a Redirect never fails because analytics
  failed. `SyncClickRecorder` therefore catches and logs everything, and increments a
  failure metric rather than propagating.
- Recording happens *after* the redirect response is determined, so a slow insert
  delays the response but can never change it.
- The eventual `QueuedClickRecorder` will lose at most one unflushed batch on a hard
  crash. Acceptable because Clicks are defined as approximate
  ([CONTEXT.md](../../CONTEXT.md)) — that definition is what buys the freedom, and it
  had to be made before the code was written, not after.
