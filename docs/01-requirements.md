# 01 — Requirements

Scope is fixed by a hard constraint: **solo, ~12–16 hours, demo on 2026-09-05.**
Everything below is either in the demo or explicitly out of it. There is no
"if we have time" tier that isn't labelled as such.

## Actors

| Actor | Authenticated? | Can do |
|---|---|---|
| **Owner** | Yes | Everything: create, list, edit, disable, delete Links; read their statistics |
| **Visitor** | No | Follow a Short Link |

There is no admin role, no team, no sharing. A Link has exactly one Owner for its
whole life; ownership is never transferred.

## Functional requirements

### FR-1 — Authentication

- **FR-1.1** An Owner registers with email + password. Registration sends a
  verification code to that address.
- **FR-1.2** An Owner logs in and receives a session valid for 1 hour.
- **FR-1.3** An Owner logs out, ending the session immediately in the browser.
- **FR-1.4** Every `/api/v1/links/**` endpoint requires an authenticated Owner.
- **FR-1.5** Passwords are stored as bcrypt hashes, never recoverable.
- **FR-1.6** An Owner verifies their email address with a 6-digit code.
- **FR-1.7** An **unverified** Owner may sign in and read their account, but may not
  create Links. See [ADR-0016](./adr/0016-verification-gates-creation.md).
- **FR-1.8** An Owner may request a replacement verification code, throttled per FR-6.
- **FR-1.9** An Owner who has forgotten their password requests a reset code by email,
  then sets a new password with it.
- **FR-1.10** Setting a new password **invalidates every existing session** for that
  Owner. See [ADR-0018](./adr/0018-session-revocation-by-token-version.md).
- **FR-1.11** Every code is 6 digits, valid for 10 minutes, single-use, dies after 5
  wrong attempts, and is **stored hashed**.
  See [ADR-0017](./adr/0017-otp-codes-in-postgres.md).
- **FR-1.12** A request for a password reset answers identically whether or not the
  address is registered.

> **This pays off part of the original demo compromise.** Email verification, password
> reset and server-side session revocation now exist. Still deliberately absent: OAuth,
> refresh tokens, and per-device revocation — logging out invalidates *every* session
> for an Owner, not one chosen device. That is enough for FR-1.10 and short of what a
> real product eventually wants.

### FR-2 — Creating Links

- **FR-2.1** An Owner submits a Destination and receives a Short Link.
- **FR-2.2** If no Alias is supplied, the system generates a 7-character Short Code.
- **FR-2.3** An Owner may supply an Alias of 3–32 characters, `[A-Za-z0-9_-]`.
- **FR-2.4** Aliases are **case-sensitive** and globally unique across the whole
  Code Namespace — first come, first served, not scoped per Owner.
- **FR-2.5** A request for a taken Alias fails with `409` and error code `ALIAS_TAKEN`.
  It never silently falls back to a generated code.
- **FR-2.6** A request for a Reserved Word fails with `409` / `RESERVED_ALIAS`.
- **FR-2.7** An Owner may set an optional expiry moment at creation.
- **FR-2.8** A Destination must be an absolute `http://` or `https://` URL. Anything
  else fails with `422` / `INVALID_DESTINATION`.
- **FR-2.9** A Destination that is a Private Destination, or that points at this
  service's own hostnames, fails with `422` / `DESTINATION_NOT_ALLOWED`. See
  [ADR-0010](./adr/0010-defer-external-url-screening.md).

### FR-3 — Redirecting

- **FR-3.1** `GET https://{short-domain}/{code}` on an Active, unexpired Link answers
  `302 Found` with the Destination in `Location`.
- **FR-3.2** Every other outcome — unknown code, Disabled, Expired, Deleted — answers
  an identical `404` with an identical body. See
  [ADR-0008](./adr/0008-soft-delete-and-uniform-404.md).
- **FR-3.3** The redirect response is not cacheable
  (`Cache-Control: private, no-cache`) and not indexable (`X-Robots-Tag: noindex`).
- **FR-3.4** A failure to record a Click must never prevent or delay a Redirect.
- **FR-3.5** The root path `/` of the short domain serves a minimal static landing
  page. No other path on the short domain serves anything but Redirects.

### FR-4 — Managing Links

- **FR-4.1** An Owner lists their Links, newest first, paginated.
- **FR-4.2** An Owner searches their Links by Short Code or Destination substring.
- **FR-4.3** An Owner changes a Link's Destination. The previous Destination is
  recorded with who changed it and when.
  See [ADR-0009](./adr/0009-mutable-destination-with-audit.md).
- **FR-4.4** An Owner changes a Link's expiry, or clears it.
- **FR-4.5** An Owner toggles a Link between Active and Disabled.
- **FR-4.6** An Owner deletes a Link. The Short Code is **never** returned to the
  Code Namespace and can never be claimed again by anyone.
- **FR-4.7** A Short Code, once created, can never be changed. Renaming would free
  the old string for someone else to claim, which is the same hazard as FR-4.6.
- **FR-4.8** An Owner can only ever see or act on their own Links. A request for
  another Owner's Link answers `404`, not `403` — `403` would confirm it exists.
- **FR-4.9** An Owner **reads the Destination history** of their own Link: every
  previous Destination, with who changed it and when.

> FR-4.9 exists because FR-4.3 was only half a feature. ADR-0009 defends mutable
> Destinations on the grounds that *"every switch is recorded, with who and when"* —
> but until now nothing could read that record, so the defence described a property the
> product did not expose. Writing an audit trail nobody can read is theatre.

### FR-5 — Statistics

- **FR-5.1** Each Link shows a total Click count.
- **FR-5.2** Each Link shows Clicks per day over a selectable range.
- **FR-5.3** Each Click records: moment, referrer, user agent, country, device class.
- **FR-5.4** Statistics are visible only to the Owner.
- **FR-5.5** Click counts are **approximate by design**. They are not an audit log.

> Country comes from Cloudflare's `CF-IPCountry` header — free at the edge, no GeoIP
> database. When absent (every local run), it is stored as `UNKNOWN` rather than
> failing. Raw IP addresses are **never stored**; see [02-nfr.md](./02-nfr.md#privacy).

### FR-6 — Abuse resistance

- **FR-6.1** Per-IP rate limit on login: 5 / minute.
- **FR-6.2** Per-IP rate limit on Link creation: 10 / minute.
- **FR-6.3** Per-Owner rate limit on Link creation: 100 / hour.
- **FR-6.4** Per-IP rate limit on Redirects: 100 / minute.
- **FR-6.5** Exceeding a limit answers `429` with a `Retry-After` header.
- **FR-6.6** The client IP is resolved from `CF-Connecting-IP` **only when the request
  arrives from the trusted tunnel**, otherwise from the socket. A spoofable header
  would make every limit above decorative.
- **FR-6.7** Per-IP rate limit on password-reset requests: 3 / hour — **and** 3 / hour
  per target email address.
- **FR-6.8** Per-Owner rate limit on requesting a code: 1 / minute and 5 / hour.
- **FR-6.9** Per-Owner rate limit on submitting a code: 10 / hour, on top of the
  5-attempt limit carried by each individual code.
- **FR-6.10** Requests authenticated by an API Key are limited **per key**, at
  60 Link creations / minute.

> **FR-6.7 has two buckets on purpose.** Limiting password-reset requests per IP alone
> lets an attacker rotate addresses and flood a victim's inbox — and that victim never
> used this service. The second bucket, keyed on the target address, caps what any
> number of attackers can do to one person.
>
> **FR-6.10 is keyed per key, not per IP,** because automation runs from shared cloud
> addresses. Per-IP would let one noisy tenant exhaust the limit for everyone else in
> the same datacenter.

### FR-7 — Operability

- **FR-7.1** `/actuator/health` reports liveness including database reachability.
- **FR-7.2** `/actuator/prometheus` exposes: redirect latency histogram, redirect
  outcome counter (hit / miss / expired / disabled), cache hit ratio, Short Code
  collision-retry counter, rate-limit rejection counter, email send outcome counter.
- **FR-7.3** Logs are structured JSON, one line per request, carrying a request id.

### FR-8 — API Keys

- **FR-8.1** An Owner creates a named API Key and uses it to call the management API
  without a browser.
- **FR-8.2** The key's plaintext is shown **exactly once**, at creation. It is stored
  only as a SHA-256 hash. See [ADR-0019](./adr/0019-api-key-authentication.md).
- **FR-8.3** The key list shows a prefix and the last four characters, so keys can be
  told apart without being recoverable.
- **FR-8.4** An Owner revokes a key. Revocation takes effect on the next request.
- **FR-8.5** A key carries the authority of its Owner **except** managing API Keys.
  A leaked key cannot mint more keys, and cannot lock its Owner out. This holds
  whatever the key's scopes are — it is not itself a scope.
- **FR-8.6** A key is presented as `Authorization: Bearer <key>`. When a request
  carries both a key and a session cookie, the key wins and the cookie is ignored.
- **FR-8.7** A key is created with an optional lifetime and expires when it runs out.
  A key created without one never expires and ends only when it is revoked.
  See [ADR-0020](./adr/0020-api-key-scopes-and-expiry.md).
- **FR-8.8** An Owner whose email is unverified cannot create Links with a key either.
  FR-1.7 is a property of the Owner, not of the credential.
- **FR-8.9** A key is created with one or more **scopes**, and may use only the
  endpoints they cover. `links:read` reads Links, their statistics and their
  Destination history; `links:write` creates, edits, disables and deletes them.
  Neither implies the other.
- **FR-8.10** A request whose key lacks the required scope is refused with `403
  INSUFFICIENT_SCOPE`, naming the scope it needed.
- **FR-8.11** An expired key stays in the Owner's key list, shown as expired with the
  date. It is the only place the difference between "expired" and "wrong key" is
  visible — authentication answers an identical `401` to both.

> **Why this exists at all.** The session is an `httpOnly` cookie
> ([ADR-0014](./adr/0014-session-in-httponly-cookie.md)), which is exactly what stops a
> script from reading it. That is the right choice for browsers and it leaves nothing
> for an automation tool, a CI job or an AI agent to authenticate with. FR-8 is the
> second door, opened deliberately rather than by weakening the first.
>
> **FR-8.7 is a real trade-off, and expiry does not settle it.** A key that dies on a
> schedule breaks an unattended integration at an hour nobody is awake. That is why a
> lifetime is *chosen*, never imposed, and why FR-8.11 exists: the failure is made
> explainable rather than prevented.
>
> **FR-8.9 was deferred once, on a stated test.** Scopes invented before a use case are
> speculative generality. The test was met by a real one — a load generator that creates
> Links and must do nothing else — and the vocabulary was shaped by it rather than
> guessed at ([ADR-0020](./adr/0020-api-key-scopes-and-expiry.md)).

## Explicit non-goals

Named so nobody has to wonder whether they were forgotten. Each is a decision.

| Not building | Why not |
|---|---|
| Click-limit expiry (`max_clicks`) | Enforcing it on the redirect path requires reading a live counter on every request, which defeats the destination cache. Time-based expiry has no such cost. Deferred to [06-roadmap.md](./06-roadmap.md). |
| Password-protected Links | No new architectural lesson; pure UI work |
| Bulk import / CSV | Same |
| Teams, sharing, ownership transfer | Multiplies the authorization model for no demo value. FR-4.9 records `changed_by` against a single Owner today; it only becomes interesting when more than one person can change a Link. |
| Per-device session revocation | FR-1.10 invalidates every session for an Owner at once. Naming individual devices needs a session table, which is the same work as refresh tokens — see R-1. |
| Decorated QR codes (logo, rounded dots) | They scan measurably worse. A code that fails on a poor camera in a lecture hall is worse than a plain one. |
| Custom domains per Owner | Real product feature, week of work, changes the routing model completely |
| Link preview / interstitial page | Contradicts the point of a redirect |
| External malicious-URL screening | [ADR-0010](./adr/0010-defer-external-url-screening.md) — deferred deliberately, interface defined |
| Real-time statistics | Statistics are read on page load. Live updating is a websocket and a reason to have one. |
| GDPR data export / deletion flows | Out of scope for a course demo; the privacy stance in [02-nfr.md](./02-nfr.md#privacy) is what makes that defensible |

## Demo script

This is the acceptance test. **Anything that does not appear here is a candidate to
cut**; anything here must work.

1. Open the dashboard → register
2. **Try to create a Link before verifying → refused, `EMAIL_NOT_VERIFIED`**
3. **Read the code from the inbox → enter it → the same action now works**
4. Paste a long URL → receive a Short Link → **click it, it redirects**
5. Create a Link with the Alias `demo` → follow it → **show its QR code**
6. Try to claim `demo` again → clean `409` surfaced in the UI
7. Try to shorten `http://192.168.1.1/admin` → refused, `DESTINATION_NOT_ALLOWED`
8. Click *create* repeatedly → `429`, with the UI showing the retry delay counting down
9. Dashboard shows the Click counts from steps 4–5
10. Edit a Destination → follow the same Short Link → arrives somewhere new →
    **open the Destination History and show the change recorded**
11. Show `/actuator/prometheus`, then open this repo and walk one ADR

**Steps 2, 7, 8 and 10 are the ones that distinguish this from a tutorial project.**
Each shows a decision refusing to do something convenient: an unverified account is
stopped, a private address is refused, a burst is throttled with an honest delay, and a
mutable Destination is made accountable rather than merely allowed.

**Droppable if the demo runs long:** step 5's QR code, and step 10's history if the
edit itself has already landed. **Not droppable:** steps 2–3, because the whole point of
adding verification was that it gates something.

Two flows are deliberately *not* in the script, and are worth having ready if asked
rather than performed: **password reset** (it invalidates the session and forces a
re-login mid-demo, which costs a minute and shows little) and **API Keys** (they exist
only in a terminal — see [ADR-0019](./adr/0019-api-key-authentication.md), which says
so plainly).
