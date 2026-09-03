---
status: accepted
date: 2026-09-03
---

# Random base62 Short Codes, not a monotonic counter

Short Codes are **7 random base62 characters from `SecureRandom`**, inserted against a
unique index, with a bounded retry (3 attempts) when the insert collides. No
coordination between application instances, no read before write, and codes that
cannot be guessed or walked.

## Considered options

**Monotonic counter → base62** (a Postgres sequence or Snowflake id, encoded).
Guarantees uniqueness with zero retries and produces the shortest possible codes.
Rejected because the codes are **enumerable**: anyone can walk `/1`, `/2`, `/3` and
harvest every Link in the system, and the length of a code leaks how many Links exist.
That is a privacy defect, not an aesthetic one — a URL shortener's contents are
frequently things people chose not to publish.

**Hash of the Destination, truncated.** Gives free deduplication of identical
Destinations. Rejected because that deduplication is usually *wrong*: two Owners
shortening the same URL want separate Links with separate statistics, and one of them
must not be able to see the other's. Collisions still need handling, so it keeps the
retry problem while adding a correctness problem.

**Key Generation Service.** A pool of pre-generated codes, popped on creation.
Rejected as premature by three orders of magnitude — see
[06-roadmap.md R-8](../06-roadmap.md), which also records the counter-plus-Feistel
variant that would get both uniqueness and unguessability if codes ever had to be
shorter.

## Consequences

- **The unique index is the collision detector.** Creation does not check
  availability first; it inserts and handles the unique violation. Checking first would
  be both slower and racy.
- Collision probability at the target scale is negligible: 10⁸ Links against
  62⁷ ≈ 3.5 × 10¹² codes is a 0.003% fill ratio. Three consecutive failures is
  effectively impossible, so it is treated as a `500` and an alertable metric rather
  than as a normal path.
- The collision-retry counter is exported to Prometheus. If the assumption above is
  ever wrong, it will be visible before it is painful.
- Codes are case-sensitive, which doubles the keyspace and complicates reading a URL
  aloud. See [04-data-model.md](../04-data-model.md).
- Generated codes and Aliases share one namespace, so a generated code can in
  principle collide with an Alias someone wants later. Same index, same handling.
