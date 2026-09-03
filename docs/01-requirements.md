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

- **FR-1.1** An Owner registers with email + password.
- **FR-1.2** An Owner logs in and receives a session valid for 1 hour.
- **FR-1.3** An Owner logs out, ending the session immediately in the browser.
- **FR-1.4** Every `/api/v1/links/**` endpoint requires an authenticated Owner.
- **FR-1.5** Passwords are stored as bcrypt hashes, never recoverable.

> **Demo compromise.** No email verification, no password reset, no OAuth, no refresh
> token, no server-side session revocation. A session cannot be invalidated before it
> expires. See [02-nfr.md](./02-nfr.md#security) for why this is acceptable here and
> what it would take to fix.

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

### FR-7 — Operability

- **FR-7.1** `/actuator/health` reports liveness including database reachability.
- **FR-7.2** `/actuator/prometheus` exposes: redirect latency histogram, redirect
  outcome counter (hit / miss / expired / disabled), cache hit ratio, Short Code
  collision-retry counter, rate-limit rejection counter.
- **FR-7.3** Logs are structured JSON, one line per request, carrying a request id.

## Explicit non-goals

Named so nobody has to wonder whether they were forgotten. Each is a decision.

| Not building | Why not |
|---|---|
| Click-limit expiry (`max_clicks`) | Enforcing it on the redirect path requires reading a live counter on every request, which defeats the destination cache. Time-based expiry has no such cost. Deferred to [06-roadmap.md](./06-roadmap.md). |
| Password-protected Links | No new architectural lesson; pure UI work |
| Bulk import / CSV | Same |
| QR codes | Genuinely nice, genuinely cuttable — a client-side library and 20 lines. Build only if the demo script is already green. |
| Teams, sharing, ownership transfer | Multiplies the authorization model for no demo value |
| Custom domains per Owner | Real product feature, week of work, changes the routing model completely |
| Link preview / interstitial page | Contradicts the point of a redirect |
| External malicious-URL screening | [ADR-0010](./adr/0010-defer-external-url-screening.md) — deferred deliberately, interface defined |
| Real-time statistics | Statistics are read on page load. Live updating is a websocket and a reason to have one. |
| GDPR data export / deletion flows | Out of scope for a course demo; the privacy stance in [02-nfr.md](./02-nfr.md#privacy) is what makes that defensible |

## Demo script

This is the acceptance test. **Anything that does not appear here is a candidate to
cut**; anything here must work.

1. Open the dashboard → register → log in
2. Paste a long URL → receive a Short Link → **click it, it redirects**
3. Create a Link with the Alias `demo` → follow it
4. Try to claim `demo` again → clean `409` surfaced in the UI
5. Try to shorten `http://192.168.1.1/admin` → refused, `DESTINATION_NOT_ALLOWED`
6. Click *create* repeatedly → `429`, with the UI showing the retry delay
7. Dashboard shows the Click counts from steps 2–3
8. Edit a Destination → follow the same Short Link → arrives somewhere new
9. Show `/actuator/prometheus`, then open this repo and walk one ADR

Steps 5, 6 and 9 are the ones that distinguish this from a tutorial project.
