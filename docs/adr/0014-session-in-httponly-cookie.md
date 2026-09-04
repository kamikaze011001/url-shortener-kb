---
status: accepted
date: 2026-09-04
---

# The session lives in an httpOnly cookie, and the frontend asks the server who it is

The JWT is set as an `httpOnly`, `SameSite=Strict` cookie. **JavaScript cannot read it**,
by design. There is no token in `localStorage` and there must never be one.

The consequence lands on the frontend: "am I signed in?" cannot be answered locally. It
is a request — `GET /api/v1/auth/me` — and the answer arrives after a round trip.

## Considered options

**A token in `localStorage`, sent as an `Authorization` header.** The reflex answer, and
genuinely simpler: the check is synchronous, there is no loading state, and the route
guard is two states instead of three. Rejected because `localStorage` is readable by any
script that executes on the page. A single XSS — in our code, in a dependency, in
anything a bundler pulled in — turns into a stolen session that survives until the token
expires, exfiltrated with one line. An `httpOnly` cookie cannot be read that way; an
attacker has to relay requests through the victim's browser, which is louder, slower, and
ends when the tab closes.

**A cookie plus a mirrored non-`httpOnly` "isLoggedIn" flag,** to skip the round trip.
Rejected because it introduces a second source of truth that can disagree with the first,
and the failure is silent in the worst direction: the flag says signed in, every request
401s, and the user sees a dashboard that answers nothing.

## Consequences

- **The session check is three states, not two.** `pending`, signed out, signed in. The
  frontend renders nothing during `pending` rather than a spinner — the query resolves in
  around ten milliseconds and a spinner that flashes and vanishes reads as a glitch.
- **One `RequireSession` gate owns this.** No screen beneath it handles a missing Owner,
  so the three-state logic exists in exactly one file.
- **A 401 from any request is recorded as "signed out" in one place.** There is no way to
  see an expiring JWT coming, because the token cannot be inspected — so the app learns
  its session died by being told, and the alternative is every screen handling it.
- **`SameSite=Strict` is what removes CSRF**, which is why the API carries no CSRF token.
  The cookie is not sent on cross-site requests at all, and there is no other
  browser-form surface.
- **Logout clears the cookie but does not invalidate the JWT.** There is no revocation
  list; a copied token stays valid until expiry. Documented in
  [02-nfr.md](../02-nfr.md) and unchanged by this ADR — it is the cost of stateless
  sessions, and the fix is a token version column or a deny-list in Redis.
- **The cost is one request on boot.** That is the whole price of making session theft
  require a network relay instead of a `console.log`.
