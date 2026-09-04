---
status: accepted
date: 2026-09-04
---

# API Keys are the second door, opened deliberately

An Owner can create a named API Key and call the management API with
`Authorization: Bearer sk_live_…`, with no browser and no session.

## Why a second mechanism at all

The session is an `httpOnly` cookie ([ADR-0014](./0014-session-in-httponly-cookie.md)),
and its whole value is that **no script can read it**. That is correct for browsers and
it leaves nothing for anything else to authenticate with — a CI job, an automation tool,
or an AI agent cannot log in, because logging in hands back a credential they are
structurally unable to hold.

So there are two options: weaken the cookie, or add a credential designed to be held by
a program. Weakening the cookie would undo ADR-0014 for every user in order to serve a
few. The second door is the smaller change.

## SHA-256, not bcrypt — and why that is not a contradiction

Passwords are bcrypt (FR-1.5). Keys are SHA-256. Two reasons, and the first is the one
that matters.

**Bcrypt cannot be indexed.** Its salt is per row, so verifying a key would mean loading
candidate rows and comparing one at a time — with no way to find the row first. A fast
hash makes the stored hash *itself* the lookup key, so authentication is a single
indexed read.

**Bcrypt exists to slow the brute-forcing of guessable secrets.** A password is chosen
by a human and lives in a dictionary somewhere. A key is 256 bits from a CSPRNG; there
is no dictionary, and no amount of slowness is what protects it. Applying bcrypt here
would buy nothing and cost the latency budget on every request.

The reasoning does not transfer back to passwords, and the moment it is used to argue
for fast password hashing, it has been misread.

## Considered options

**Show the key once, or store it recoverably.** Recoverable is friendlier — a lost key
can be looked up rather than rotated — and it means a database leak is a leak of live
credentials. Shown once, stored hashed. The list carries `key_prefix` and the last four
characters so keys can be told apart, because an Owner with three keys must be able to
choose which to revoke without being able to use any of them.

**Scoped keys (read-only, write, per-Link).** Deferred. Nobody has asked for one, and
scopes chosen before a use case exists are speculative generality of exactly the kind
[06-roadmap.md](../06-roadmap.md) argues against. One exception is not speculative and
is kept: **a key cannot manage API Keys.** A leaked key therefore cannot mint more keys
and cannot lock its Owner out — a real containment property for almost no code.

> **Superseded by [ADR-0020](./0020-api-key-scopes-and-expiry.md).** The deferral had a
> stated test — *no use case yet* — and the test was met: the load generator needs a key
> that creates Links and does nothing else. The exception above survives unchanged, and
> deliberately did not become a scope.

**Expiring keys.** Better hygiene, worse operations: a key that dies on a schedule
breaks an unattended integration at an hour nobody is awake, and the failure appears
long after the decision that caused it. Revocation-only keeps a human in the loop.

> **Superseded by [ADR-0020](./0020-api-key-scopes-and-expiry.md).** The objection was
> never wrong, and it is not dismissed there — expiry is opt-in, and an expired key stays
> visible in the list so the 3am failure can be explained rather than only suffered.

## Consequences

- **The header wins over the cookie, and they never mix.** A request carrying both is
  authenticated by the key alone. The header is explicit and the cookie is ambient, and
  a request that half-uses each is the kind of thing that turns into a privilege bug.
- **Keyed requests are rate-limited per key, not per IP** (FR-6.10). Automation runs
  from shared cloud addresses; keying on IP would let one tenant exhaust the ceiling for
  every unrelated tenant in the same datacenter. The limit is also higher than the
  browser's, because a script making sixty requests a minute is working, not attacking.
- **Verification still applies** (FR-8.8). An unverified Owner cannot create Links with
  a key either. Attaching the gate to the session rather than the Owner would have made
  API Keys an accidental bypass of
  [ADR-0016](./0016-verification-gates-creation.md).
- **`last_used_at` is written on use.** It answers the only question that matters before
  revoking anything: is something still using this? The write is best-effort and must
  never fail a request — the same posture `ClickRecorder` takes in
  [ADR-0005](./0005-synchronous-click-recording.md).
- **This is invisible in a live demo.** It shows up in a terminal, not on a dashboard.
  Recorded here so the choice to build it is understood as one made for the design
  record, not for the presentation.
