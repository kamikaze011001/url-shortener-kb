---
status: accepted
date: 2026-09-04
supersedes: partially ADR-0019 ("Considered options": scoped keys, expiring keys)
---

# API Keys get a scope and an expiry

An API Key is created with a set of **scopes** and an optional **expiry**. Both were
considered in [ADR-0019](./0019-api-key-authentication.md) and deferred. This records
what changed and why the deferral no longer holds.

## What changed

ADR-0019 deferred scopes with a specific reason: *"Nobody has asked for one, and scopes
chosen before a use case exists are speculative generality."* That was the right test,
and it has now been met. The use case is the **load generator**: a script that hammers
the API to demonstrate empirically where the system breaks
([02-nfr.md](../02-nfr.md)). It needs to create Links and nothing else. That is not a
scope invented to be general — it is one credential, one job, described.

[R-12](../06-roadmap.md) called the hard part *"deciding the vocabulary of scopes
without a real request to shape it."* The request arrived, and it shaped the vocabulary:
read and write, separately, because the load generator is exactly the key that wants
write without read.

The expiry deferral was different. It was not "no use case" but a genuine trade-off:
**a key that dies on a schedule breaks an unattended integration at an hour nobody is
awake.** That objection is still true. What changed is that we can now answer it (see
below) rather than only avoid it.

## The scope vocabulary

Two scopes, and they are independent:

| Scope | Grants |
|---|---|
| `links:read` | List Links, read one, its statistics, its Destination history |
| `links:write` | Create, edit, disable and delete Links |

Three things this vocabulary deliberately does **not** have:

**`links:write` does not imply `links:read`.** Implication would make write-only
inexpressible, and write-only is the single use case that motivated the feature. A key
that can create Links but cannot enumerate them is also strictly the safer thing to hand
to a script.

**No separate scope for statistics.** Reading a Link's clicks is reading a Link. A
`stats:read` would be a scope for a resource nobody has asked to separate, which is the
mistake ADR-0019 warned about, committed one level down.

**No scope for managing API Keys — not even one that is always absent.** FR-8.5 stays
what it is: a key cannot manage keys, *structurally*, whatever its scopes say. Expressing
it as a scope would turn a containment property into a permission, and a permission is
something that can be granted. Nothing should be able to grant it.

A session cookie carries every scope. A session is the human, and the human already has
full authority over their own account; scoping the browser would be scoping a person's
access to their own dashboard.

## Expiry, and the objection that has not gone away

`expiresAt` is nullable and **defaults to never** at the API level. A key expires only
because someone chose a lifetime for it. The default in the *interface* is 90 days,
because a person creating a key in a browser is choosing hygiene and a person calling
`POST /api-keys` from a provisioning script is choosing operations, and the two want
different defaults from the same endpoint.

Creation takes `expiresInDays`, not an absolute timestamp. A caller sending an absolute
time needs a clock and a timezone and gets both wrong in the interesting cases; a
duration is unambiguous and the server owns the clock either way.

**The 3am objection is answered by making the failure legible, not by preventing it.**
An expired key still appears in the key list, marked expired, with the date it died.
The thing that makes schedule-driven breakage awful is not that it happens — it is that
the person debugging it at 3am has no way to see *why* the calls started failing. One
row in a list they already know how to find turns an unexplained 401 into a fact.

## Considered options

**Scopes as a `text[]` column, or a join table.** A join table is the normalised answer
and buys nothing here: scopes are a small fixed set, they are always read with the key,
and they are never queried across keys. The column carries a `CHECK` that the array is
non-empty and every element is one of the known scopes, so an unknown scope cannot be
stored even by a hand-written `INSERT`.

**A distinct error for an expired key at authentication time, or a uniform 401.**
Uniform. The API Key filter is deliberately silent about failure — it leaves the context
anonymous and lets the security chain answer — because rejecting inside it would 401 the
public redirect path for any client sending a stale header. Distinguishing "expired"
from "unknown" would mean plumbing a reason out of a filter whose whole design is to
have no opinion. **This is a real cost:** the caller sees the same 401 for an expired
key as for a wrong one. The key list is where the difference is visible, and that is a
worse place to learn it than the response would have been.

**Backfilling existing keys.** Existing rows get every scope and no expiry — exactly the
authority they have today. Adding a column must never change what an existing row means.
The same rule was applied to `email_verified` in V2 and for the same reason: a migration
that retroactively narrows a working credential breaks things nobody changed.

## Consequences

- **`INSUFFICIENT_SCOPE` (403) is a new error code**, and therefore part of the
  contract. Its `detail` names the missing scope, because the audience is a person
  reading a script's log at the point where they can fix it.
- **Scope is checked at the web edge**, beside `requireVerified()` and
  `requireSessionCredential()`, and for the same reason: it is a fact about the caller,
  not about the business operation. A use case that read the security context to find
  out would be an edge concern wearing business clothing.
- **Expiry is checked in the authentication query**, not after it. `WHERE ... AND
  (expires_at IS NULL OR expires_at > now)` costs nothing on a lookup that was already
  a single indexed read, and it means no code path can hold an expired key in its hand
  and forget to look.
- **This narrows what a leaked key can do, and bounds how long it can do it.** ADR-0019
  could say only that a leaked key cannot escalate. It can now also say the blast radius
  is whatever that key was scoped to, for as long as its Owner allowed.
- **The revocation story is unchanged.** Expiry is not a substitute for revoking a key;
  it is what happens to the keys nobody remembers to revoke.
