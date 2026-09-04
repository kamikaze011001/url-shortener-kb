---
status: accepted
date: 2026-09-04
---

# An unverified Owner may sign in, but may not create Links

Email verification blocks exactly one thing: **creating a Link**. An unverified Owner
can register, sign in, see their empty dashboard, read their account, request a new
code, and change their password. They cannot make a Short Link until the address is
confirmed.

## Considered options

**Block nothing; show a banner.** The feature becomes decorative. An address that is
never confirmed and never needs to be is not verified, it is decorated, and the
"verified" column would be a field nobody may rely on.

**Block login entirely until verified.** The reflex answer, and it builds a dead end:
the one thing an Owner needs when the email did not arrive is a way to ask for another
one — and that control lives behind the login they cannot pass. Every recovery path
then has to be rebuilt as an unauthenticated endpoint keyed on the email address, which
is a fresh enumeration surface for no gain.

**Let them create Links, but make those Links not redirect.** Rejected hardest. The
Owner shares a link that silently 404s, so the cost of the Owner's unverified address
lands on a **Visitor** who did nothing wrong and has no way to fix it. A penalty must
land on the party who can act on it.

## Consequences

- **Verification status is read on every authenticated request**, not carried as a JWT
  claim. It rides in the same Redis-cached row as `token_version`
  ([ADR-0018](./0018-session-revocation-by-token-version.md)), so it costs nothing
  extra — and it cannot go stale, which means an Owner who verifies is not forced to
  log out and back in to escape the gate.
- **It is a property of the Owner, not of the credential.** An API Key belonging to an
  unverified Owner is refused for the same reason (FR-8.8). Attaching the check to the
  session instead would have left the key as an unintended bypass.
- **The gate is `403`, not `404`.** This is the one deliberate exception to the uniform
  404 in [ADR-0008](./0008-soft-delete-and-uniform-404.md). That rule exists to avoid
  confirming whether a *resource* exists; here the caller is asking about their own
  account, already knows it exists, and needs to be told what to do about it. A 404
  would be actively unhelpful and would leak nothing either way.
- **An unverified account is not a dead account.** It can be recovered, upgraded and
  deleted through the normal flows. Nothing needs a separate administrative path.
