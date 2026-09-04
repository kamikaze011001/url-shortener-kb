# 06 — Roadmap: what changes, and what triggers it

The course brief asks for *decisions and trade-offs, at the moment and in the future*.
[Documents 01–05](./01-requirements.md) are the present. This one is the future.

**Every entry has a trigger.** An item with no trigger is a wish, not a plan — and
building it early is how a two-day demo becomes a two-week project. Nothing here is
scheduled by date; each is scheduled by a condition that makes it necessary.

## Stage 0 — Cuttable, on demo day itself

Sequenced last on purpose. If Friday runs out, these are dropped and the demo is
unaffected.

| Item | Trigger | Cost |
|---|---|---|
| **Deploy behind the tunnel** | The demo script is green on localhost | ~30 min |
| **QR code per Link** | Everything above is done | ~20 min, client-side library |
| **Copy-to-clipboard, empty states, loading skeletons** | Same | polish, unbounded |

## Stage 1 — First real users

Triggered by anyone other than the author depending on the service.

### R-1 — Session revocation and refresh tokens

**Status: half delivered.** The trigger fired early, and not from where this entry
expected — password reset (FR-1.9) made revocation necessary, because resetting a
compromised password while the attacker's session stayed alive would have made the
feature half theatre.

`token_version` now exists and is checked on every authenticated request. The cost that
kept it out of the demo turned out not to apply: **the redirect path is
unauthenticated**, so the check never touches the one path with a 20 ms budget — it
runs only on the management API, where the budget is ten times looser. See
[ADR-0018](./adr/0018-session-revocation-by-token-version.md).

**What remains:**

- **Per-device revocation.** Today revocation is all-or-nothing per Owner. Naming one
  device needs a session table, which is the same work as refresh tokens.
- **Refresh tokens with rotation and reuse detection**, so expiry stops being abrupt.
  Still more work than it appears, and still not triggered.

**Trigger for the remainder:** the first "log me out of that one laptop" request, or
an Owner complaining about being signed out mid-task.

### R-2 — External Destination screening

**Trigger:** the first abuse report, or making the service publicly registerable.

The interface already exists — `DestinationScreener` — with one local implementation.
Adding Google Safe Browsing means writing a second implementation and answering the
question the demo deliberately defers: **when the external API times out, do we fail
open or fail closed?** Fail open and a malicious link gets through; fail closed and an
outage at Google stops link creation entirely.

The likely answer is a hybrid: fail open on timeout, but queue the Destination for
re-screening, and disable the Link retroactively if it comes back bad. That requires
background jobs, which the demo does not have. See
[ADR-0010](./adr/0010-defer-external-url-screening.md).

### R-3 — Roll up and prune Click events

**Trigger:** `click_events` above ~10 million rows, or dashboard statistics above
500 ms.

A `click_daily (link_id, day, clicks, ...)` table, written by a nightly job, read by
the dashboard. Raw events then become prunable at 90 days.

This is the entry that turns [02-nfr.md](./02-nfr.md)'s 60 GB/month from a fatal
problem into a bounded one, and it is only safe **because Clicks are defined as
approximate** ([CONTEXT.md](../CONTEXT.md)). If they had been defined as an audit
log, this row could not exist and the storage problem would be permanent.

## Stage 2 — Load

Triggered by measurements, taken from the metrics in
[01-requirements.md § FR-7](./01-requirements.md).

### R-4 — Move Click recording off the redirect path

**Trigger:** the `UPDATE links SET click_count = click_count + 1` shows lock waiting,
or any single Link exceeds roughly 50 concurrent clicks/second.

**This is the first thing that breaks**, and it is designed to be easy to fix. The
redirect path calls `ClickRecorder.record(event)`. Today that is `SyncClickRecorder`.
The replacement is:

1. `QueuedClickRecorder` — a bounded in-memory queue, batch-inserting on
   100-events-or-500ms. **Bounded with a drop-on-full policy**: a full queue must shed
   load, never block a redirect. A `clicks_dropped` counter makes the loss visible
   rather than silent.
2. Drop the denormalised `click_count` entirely; read from `click_daily` (R-3).
3. Later, `KafkaClickRecorder`, when analytics needs to fan out to more than one
   consumer.

**Nothing on the redirect path changes at any of those three steps.** That is the
entire return on defining the interface up front — see
[ADR-0005](./adr/0005-synchronous-click-recording.md).

The accepted cost, stated now so it is not discovered later: a hard crash loses at
most one unflushed batch.

### R-5 — Read replicas

**Trigger:** sustained redirect load above ~1,000 RPS, or Postgres CPU above 70%.

Redirect reads move to a replica; writes stay on the primary. The destination cache
already absorbs most redirect reads, so this arrives later than intuition suggests —
which is itself the argument for the cache existing.

### R-6 — Horizontal application replicas

**Trigger:** application CPU above 70% sustained.

Already possible: the application holds no in-process session state, and rate limits
already live in Redis rather than in memory. This is a compose change, not a redesign.
Both of those properties were chosen for this reason and cost nothing today.

### R-7 — Redirect at the edge

**Trigger:** end-to-end p99 above the 300 ms budget, or the home uplink saturating —
which [02-nfr.md](./02-nfr.md) predicts happens **first**, before any software limit.

A Cloudflare Worker plus KV holding `code → destination`, serving redirects at the
edge without ever reaching the origin. Writes push to KV; deletions and edits purge
it. The Click recording then has to happen at the edge too, which is a genuinely
different analytics architecture — this is the largest single change on this page.

### R-11 — Sample redirect logging

**Trigger:** redirect traffic above ~50 RPS sustained, or log volume becoming a disk
or ingest cost.

One INFO line per Redirect is the right choice at the Real scale and is roughly 2,000
lines per second at the Paper scale. The fix is to sample `link.redirected` (1 in N) or
drop it to DEBUG, and let the Prometheus counters carry what the lines were carrying.
`link.redirect_missed` stays at INFO — failures are rare and individually interesting,
which is exactly the asymmetry that makes sampling safe.

## Stage 3 — Changes that are decisions, not scaling

Not triggered by load. Triggered by wanting a different product.

### R-12 — Scoped API Keys — **done**

**Trigger, as written:** the first Owner who wants to hand a key to something they do
not fully trust — a third-party integration, or a script someone else runs.

**What actually pulled it in** was narrower and more concrete: a load generator, which
needs to create Links and must do nothing else. This page said the hard part was
*"deciding the vocabulary of scopes without a real request to shape it"* — so the
vocabulary waited for the request, and then came out of it: `links:read` and
`links:write`, independent, because write-without-read is precisely the credential that
motivated the work. Built with expiry in
[ADR-0020](./adr/0020-api-key-scopes-and-expiry.md).

Worth keeping on the record: the deferral was not laziness and the trigger was not
guessed. The item stated a test, the test was met, and the shape the item predicted —
*"a `scopes` column and a check at the edge"* — is exactly what shipped.

**What is still not built:** per-Link scopes, and scopes for anything outside Links.
Both are waiting on the same kind of request that unblocked this one.

### R-14 — Versioned frontend releases

**Trigger:** the first frontend deploy that has to be undone, or the first one that lands
while somebody is loading the page.

The frontend deploy replaces a directory in place
([ADR-0021](./adr/0021-containerised-app-host-installed-state.md)). It is not versioned,
so rolling back means rebuilding from a tag, and it is not atomic, so there is a window
in which Caddy serves a directory that is half-replaced.

The shape is release directories named by build hash and a symlink swapped in place,
keeping the last few. **Not a container:** Caddy has to serve the SPA either way, so
containerising it adds a process and a network hop to serve 372 KB without fixing
anything the symlink does not.

Worth recording *why* this was deferred rather than missed. The argument that justified
containerising the backend — that a rollback should not require a rebuild — applies here
word for word, and simply was not applied to the second half of the system until someone
asked why the two halves were treated differently. The gap is small and the fix is known;
naming it cost less than closing it on the day.

### R-13 — Deliverability and email as a dependency

**Trigger:** the first Owner who reports never receiving a code.

Email is now on the critical path for registration and recovery, and it is the least
reliable component in the system — a code that lands in spam is indistinguishable, from
the Owner's side, from one never sent. What is missing: bounce and complaint handling,
a visible send log the Owner can check, and SPF/DKIM on a real domain rather than a
provider's shared sending domain.

The deeper question this raises: **should an unverified Owner be blocked at all if the
blocking mechanism can fail silently?** [ADR-0016](./adr/0016-verification-gates-creation.md)
answers yes today, on the grounds that every recovery path stays reachable. If
deliverability turns out to be bad, that answer is the one to revisit first.

### R-8 — Key Generation Service

**Trigger:** Short Code collision retries becoming measurable — realistically a fill
ratio above ~1%, which is 35 billion links.

A service pre-generating unused codes into a pool; creation pops one instead of
generating-and-retrying. The classic system-design answer, and **honestly unnecessary
here forever** — at 10⁸ links against 62⁷ codes the collision probability is 0.003%.
Recorded so it is clear it was considered and rejected on arithmetic, not overlooked.

The more interesting variant, if codes ever needed to be shorter: **a monotonic counter
passed through a Feistel network** (a format-preserving permutation). The counter gives
guaranteed uniqueness with no retry and no coordination; the permutation scrambles it
bijectively so codes are not enumerable. That gets both properties that
[ADR-0002](./adr/0002-random-base62-short-codes.md) had to choose between — at the
cost of a fixed keyed permutation that can never be rotated without breaking every
existing code.

### R-9 — Custom domains per Owner

**Trigger:** a paying customer asks.

`links.code` stops being globally unique and becomes unique *per domain*. That changes
the primary lookup key, the unique index, the cache key, and the Reserved Word rules —
and it makes [ADR-0006](./adr/0006-two-hostname-topology.md)'s "the short host's root
namespace belongs to Short Codes" a per-tenant statement. Plus TLS provisioning per
customer domain. This is a genuine re-architecture, and the largest item on this page.

### R-10 — Permanent-link mode

**Trigger:** a user who wants CDN-cacheable links and does not care about statistics.

An opt-in flag making a Link answer `301` with a long `Cache-Control`, in exchange for
becoming **immutable and uncountable**. This makes the trade-off in
[ADR-0003](./adr/0003-302-not-301.md) explicit and user-facing instead of a
system-wide assumption — which is the right long-term shape: it was never really a
technical decision, it was a product one.

## The through-line

Read down the page and the same pattern appears five times:

> The change is cheap **because a seam was placed where the change was expected**, and
> the seam cost nothing to place.

`ClickRecorder` (R-4), `DestinationScreener` (R-2), stateless application (R-6),
approximate Clicks (R-3), configurable short base URL
([03-architecture.md](./03-architecture.md)). None of those is speculative
generality — each is one interface or one config value, chosen where a *named*,
*predicted* change lands.

The counter-example is deliberately on this page too: **R-9 has no seam**, because
supporting custom domains later was not worth constraining the data model now. That is
also a decision, and the honest one to defend.
