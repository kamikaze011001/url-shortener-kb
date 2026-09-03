---
status: accepted
date: 2026-09-03
---

# Soft delete, codes never recycled, and one uniform 404

Deleting a Link sets `deleted_at`; the row stays forever and **its Short Code is never
returned to the Code Namespace**. To a Visitor, four different situations — the code
never existed, the Link was deleted, the Link is Disabled, the Link has Expired —
produce a byte-for-byte identical `404`.

Two decisions, recorded together because they are the same underlying concern: a Short
Code, once shared, is out of our control forever.

## Why codes are never recycled

Someone printed `s.example.com/promo` on a poster. The Owner deletes the Link. If
`promo` returns to the namespace, a stranger can claim it and the poster now points at
their content. There is no notification, no version, and no way for anyone holding the
old link to know. **This is a security hole, not a housekeeping question**, and it is
why the unique index on `links.code` is deliberately *not* partial on `deleted_at`
([04-data-model.md](../04-data-model.md)).

The same reasoning forbids renaming a Short Code
([FR-4.7](../01-requirements.md)): a rename frees the old string just as a delete would.

## Why every failure looks identical

Distinguishing the cases — `410 Gone` for deleted, `403` for disabled — is more
informative and more RESTful, and it leaks. Each distinct response **confirms that a
code existed**, which turns the redirect endpoint into an oracle for enumerating the
namespace: an attacker learns which codes are real without ever seeing a Destination.
The same reasoning makes another Owner's Link a `404` rather than a `403` in the
management API ([05-api-contract.md](../05-api-contract.md)), and makes a failed login
`401` for both a wrong password and an unknown email.

## Consequences

- The `links` table grows monotonically and is never pruned. At the scale in
  [02-nfr.md](../02-nfr.md) this is 50 GB of Links against 60 GB *per month* of Click
  events, so it is not the storage problem worth solving.
- The dashboard must filter `deleted_at IS NULL` everywhere. The partial index
  `links_owner_created_idx` carries that predicate so the filter is free on the listing
  path, which is the only place it is hot.
- Debugging is worse: "why is this link 404?" cannot be answered from the response, only
  from the logs. Accepted, and the logs record the actual reason with the request id.
- An Owner cannot free an Alias they no longer want. The correct product answer, if this
  ever matters, is to let them edit the Destination
  ([ADR-0009](./0009-mutable-destination-with-audit.md)) — not to release the code.
