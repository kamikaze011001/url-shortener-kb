---
status: accepted
date: 2026-09-03
---

# Destinations are mutable, and every change is recorded

An Owner can change a Link's Destination after creation. Every change writes a row to
`link_destination_history` — old value, new value, who, when — in the same transaction
as the update.

The audit trail is what makes the decision defensible. Mutable Destinations are the
mechanism behind most short-link abuse: share a link that points somewhere innocuous,
wait for it to circulate, then repoint it. "Someone could bait-and-switch" is a real
objection, and the answer is not that it cannot happen — it is that **every switch is
recorded, attributable, and reversible**.

## Considered options

**Immutable Destinations.** A Link means exactly one thing forever, the abuse vector
disappears entirely, and aggressive caching becomes safe — including the `301` that
[ADR-0003](./0003-302-not-301.md) had to give up. Rejected because it makes the
service unusable for its most common real purpose: a printed QR code or a poster URL
whose target moved, and a typo that can only be fixed by burning the code
([ADR-0008](./0008-soft-delete-and-uniform-404.md) means the old one is gone forever).
The feature is the reason people pay for link shorteners.

## Consequences

- **This decision creates the cache invalidation problem.** Every Destination change
  must evict `code` from Redis, or Visitors keep reaching the old target for up to an
  hour. If Destinations were immutable, the cache would need no invalidation at all —
  the TTL alone would be correct. See
  [ADR-0004](./0004-redis-is-cache-not-truth.md).
- It is also part of why redirects cannot be `301`: a browser that cached a permanent
  redirect would never observe the change. The two ADRs constrain each other.
- `link_destination_history` is append-only and never pruned. An audit trail with a
  retention policy is not an audit trail.
- `changed_by` is stored separately from `links.owner_id` so that a future ownership
  transfer cannot silently rewrite who made a past change.
- The history is not exposed through the API today. It exists to answer an abuse
  report, and surfacing it to Owners is a UI decision that can be made later without
  changing the schema.
