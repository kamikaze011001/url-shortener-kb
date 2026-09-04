---
status: accepted
date: 2026-09-04
---

# One-time codes live in Postgres, hashed

Verification and password-reset codes are rows in `otp_codes`: six digits, valid ten
minutes, single-use, dead after five wrong attempts, stored as a SHA-256 hash.

## Why not Redis

Redis is the obvious home — it has TTL built in, and the code is short-lived by nature.
It is rejected by a rule this project already wrote down.

[ADR-0004](./0004-redis-is-cache-not-truth.md) says Redis holds a destination cache and
rate-limit counters and is **never a source of truth**, because losing it must cost a
cold cache and temporarily unenforced limits — never a wrong answer. A code that
authorises a password change is not a cache of anything. Losing it mid-flight would
invalidate every reset in progress, and the failure would look to the user like the
code was wrong.

The attempt counter makes this sharper. Keeping `attempts` in Redis means a container
restart resets it to zero, and **a brute-force limit that clears when a process
restarts is not a limit.** Six digits is a million possibilities; five attempts makes
guessing hopeless and infinite attempts makes it a matter of patience.

## Why hashed, and why SHA-256 here

Passwords are bcrypt (FR-1.5). Codes are SHA-256. The difference is not an
inconsistency, it is the reason bcrypt exists.

Bcrypt is deliberately slow so that a stolen hash cannot be brute-forced back into a
guessable secret. A six-digit code has only a million values, so slowness would not save
it — what saves it is that it dies in ten minutes and after five attempts. Those bounds
do the work bcrypt would fail to do here, and a fast hash keeps verification off the
latency budget.

What hashing buys is narrower and still worth it: **a database leak does not hand the
reader a set of live codes.** The window is small, and it should not be a free one.

## Consequences

- **Ten minutes is a guess, and it is the number most likely to be wrong.** Too short
  and a slow mail relay produces codes that expire before they arrive; too long and the
  window for an intercepted email widens. It is one constant, changeable when real
  delivery times are known — which is a reason to measure them.
- **A consumed code is kept, not deleted.** `consumed_at` marks it. The case worth being
  able to see is the same code arriving twice, and a deleted row cannot testify.
- **Codes are per Owner and per purpose**, so requesting a password reset does not
  invalidate a verification code in flight. Sharing one row for both would make two
  unrelated flows interfere.
- **Delivery is not guaranteed and is not treated as if it were.** A send failure is
  counted (FR-7.2) and surfaced to the Owner as "we could not send it, try again",
  never swallowed. The commonest real failure here is not a bug, it is a spam folder.
