# 02 — Non-functional requirements

Every number here is written twice: what the system **actually** faces on demo day,
and what the **design reasoning** targets. The gap between the two columns is the
whole subject of this project.

Quoting only the right-hand column would be dishonest; designing only for the
left-hand column would be pointless. Both are stated, and every decision says which
one it serves.

## Scale

| | **Real (2026-09-05)** | **Paper (the design target)** |
|---|---|---|
| Registered Owners | ~10 | 1,000,000 |
| Links stored | < 1,000 | 100,000,000 |
| New Links / month | ~500 | 5,000,000 |
| Redirects / month | < 10,000 | 500,000,000 |
| Read : write ratio | ~10 : 1 | ~100 : 1 |
| Average redirect rate | negligible | ~200 RPS |
| Peak redirect rate | ~5 RPS | ~2,000 RPS |
| Concurrent users | ~10 | ~20,000 |

**Where the Paper column comes from.** 500M redirects/month ÷ 2.6M seconds ≈ 190 RPS
average. Short-link traffic is extremely bursty — one link in a popular post carries
most of a day's traffic in ten minutes — so peak is taken at 10× average, not the
usual 2–3×. That burstiness, not the average, is what any redirect design must survive.

## Capacity

At the Paper scale, with ~500 bytes per Link row plus indexes:

- **Links**: 100M × 500 B ≈ **50 GB** — comfortably a single Postgres instance.
- **Click events**: 500M/month × ~120 B ≈ **60 GB per month**, growing without bound.

That second number is the interesting one. **Link storage is not a scaling problem;
click storage is.** Raw click events must be rolled up into daily aggregates and then
pruned, or the analytics table outgrows the entire rest of the system within a
quarter. See [06-roadmap.md](./06-roadmap.md).

Short Code exhaustion is a non-issue and should not be discussed as one: 62⁷ ≈
3.5 × 10¹² codes against 10⁸ links is a fill ratio of 0.003%.

## Latency

Two budgets, kept separate on purpose. Conflating them produces a number that means
nothing, because most of the second one is outside this system entirely.

| Path | Budget | Measured where |
|---|---|---|
| **Redirect, server-side** | p50 < 5 ms, **p99 < 20 ms** | inside the application, Micrometer timer |
| **Redirect, end-to-end** | **p99 < 300 ms** | the Visitor's browser |
| Create Link (API) | p95 < 200 ms | inside the application |
| Dashboard list (API) | p95 < 300 ms | inside the application |

The end-to-end budget is loose because the deployment (see
[03-architecture.md](./03-architecture.md)) routes every request through Cloudflare's
edge and a tunnel into a home server. **That path adds tens of milliseconds that no
amount of application tuning will recover.** The server-side budget is the part
actually under our control, so it is the one held to a tight number and instrumented.

## Availability

**There is no availability target, and claiming one would be a lie.** The service runs
on a single machine, on a residential internet connection, with a single-instance
Postgres and no replica. A power cut, an ISP outage, or a `docker compose down` is a
total outage, and recovery is manual.

This is a stated consequence of the deployment decision, not an oversight. What is in
place instead:

- **Data durability**: Postgres data lives on a named Docker volume, so container
  restarts never lose data. Backups are a documented gap, not a solved problem.
- **Graceful degradation**: if Redis is unreachable the application keeps serving —
  the cache falls through to Postgres and rate limiting fails **open**. That choice is
  deliberate: an unavailable rate limiter should not take down redirects. It also
  means Redis is an availability *dependency* for abuse resistance but not for
  correctness. See [ADR-0004](./adr/0004-redis-is-cache-not-truth.md).
- **Stateless application**: no in-process session state, so the app container can be
  restarted at any moment and, later, replicated without redesign.

## Security

| Requirement | How |
|---|---|
| Passwords never recoverable | bcrypt, cost 10 |
| Session token not readable by JavaScript | JWT in an `httpOnly`, `SameSite=Strict` cookie — never `localStorage` |
| Cookie not sent over plaintext | `Secure` flag, on in the `prod` profile, off in `local` (there is no HTTPS on localhost) |
| No open redirect | Destination must be absolute `http(s)`, must not be a Private Destination, must not point at this service |
| No namespace enumeration | Uniform `404`; another Owner's Link is `404`, not `403` |
| No IP spoofing past the rate limiter | `CF-Connecting-IP` trusted **only** from the tunnel's address |
| No credential stuffing | 5 login attempts / minute / IP |

**Known, accepted weaknesses.** Each is a demo compromise with a named fix:

- **A session cannot be revoked before its 1-hour expiry.** Fix: a token-version
  column on the Owner, checked per request — one query per request, which is why it
  isn't free.
- **No refresh token**, so an Owner is logged out abruptly after an hour. Fix: a
  refresh token with rotation and reuse detection. Materially more code than it looks.
- **No CSRF token.** The session cookie is `SameSite=Strict`, which blocks the
  cross-site form-post attack in every browser this demo will run in. That is
  defence-in-depth reduced to defence-in-one-depth, and it is stated as such.
- **No account lockout**, only rate limiting. A slow distributed attack is unimpeded.

## Privacy

**Raw IP addresses are never stored.** A Click records a salted hash of the IP for
coarse deduplication, plus a two-letter country code. This is not a formality: an IP
address plus a timestamp plus a destination is enough to identify a person, and a URL
shortener sees precisely that for every link its Owners share.

Storing the hash instead costs one function call and removes the entire category of
"what happens when this database leaks". User agents are stored raw but only to derive
a device class, and are truncated to 512 characters.

**This extends to logs.** Passwords, JWTs, the `Cookie` header and raw IP addresses are
never written to a log line. Promising that IPs are never stored and then writing them
to a log file would be a distinction without a difference. Where correlation across
requests is needed, the first 8 characters of `ip_hash` are logged instead. See
[03-architecture.md § Observability and logging](./03-architecture.md).

## Correctness properties

Three invariants that must hold no matter what else fails. Everything else is
negotiable; these are not:

1. **A Short Code is never reused.** Not after deletion, not after expiry, not ever.
   Reuse would silently repoint a link that someone already shared and trusted.
2. **A Redirect never fails because analytics failed.** Recording is downstream of
   answering, always.
3. **A Visitor cannot distinguish "never existed" from "deleted", "disabled", or
   "expired".** All four are the same response, byte for byte.

## What breaks first

Named in order, with the trigger that would force each fix. This ordering *is* the
roadmap; [06-roadmap.md](./06-roadmap.md) is the same list with the work attached.

| # | Breaks | Trigger | Fix |
|---|---|---|---|
| 1 | The `UPDATE links SET click_count = click_count + 1` on the redirect path serializes every concurrent Click on one popular Link | one link above ~50 concurrent clicks/s | move click recording off the hot path — the seam already exists ([ADR-0005](./adr/0005-synchronous-click-recording.md)) |
| 2 | `click_events` outgrows everything else | ~2 months at Paper scale | daily rollup + prune |
| 3 | Single Postgres saturates on redirect reads | ~1,000 RPS sustained | read replicas; the destination cache already absorbs most of this |
| 4 | Single application instance saturates CPU | ~2,000 RPS | horizontal replicas — already stateless, so this is a compose change |
| 5 | Home uplink and tunnel saturate | well before any of the above | move off the home server entirely |

Row 5 is the honest headline: **the deployment fails long before the software does.**
Any claim that this design "handles 2,000 RPS" is a claim about the code, not about
the machine it will run on this Saturday.
