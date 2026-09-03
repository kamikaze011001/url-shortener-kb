---
status: accepted
date: 2026-09-04
---

# One class per use case; no service layer

There is no `LinkService`. Each business operation is its own class with a single
entry point, a nested `Command` record for its input, and a nested `Result` record for
its output:

```java
public class CreateLinkUseCase {
    public record Command(OwnerId owner, URI destination, String alias, Instant expiresAt) {}
    public record Result(LinkId id, String code, String shortUrl) {}

    @Transactional
    public Result execute(Command command) { ... }
}
```

Controllers map HTTP to a `Command`, call `execute`, and map the `Result` back. They
contain no business logic and no orchestration.

## Considered options

**A conventional service layer** — `LinkService`, `AuthService`, `AnalyticsService`.
The default in every Spring codebase, and rejected for three specific reasons rather
than fashion:

1. **A service accumulates.** By the end of a project `LinkService` holds fifteen
   methods and the union of every dependency any of them needs, so nothing about its
   constructor tells you what creating a Link actually touches. One class per use case
   makes that structurally impossible.
2. **Transaction boundaries become accidental.** `@Transactional` on a multi-method
   service is per-method by default and nobody revisits it. On `execute()` the
   boundary is exactly one business operation, by construction.
3. **The class name is the requirement.** `CreateLinkUseCase` maps to FR-2,
   `ResolveShortCodeUseCase` to FR-3. Tracing a requirement in
   [01-requirements.md](../01-requirements.md) to the code that implements it becomes
   mechanical rather than a matter of reading around.

This is **not** clean architecture, hexagonal architecture, or ports and adapters.
There is no domain/application/infrastructure layering, no dependency inversion for
its own sake, and repositories are called directly. Only the service layer changed.

## Consequences

- **Roughly ten small classes instead of three large ones.** More files, each of which
  fits on a screen. Accepted deliberately.
- **A `LinkHelper` junk drawer is the failure mode to watch for.** Shared behaviour
  goes into a *named collaborator with a real job* — `ShortCodeGenerator`,
  `DestinationScreener`, `ClickRecorder`, `LinkFinder` — or onto the domain object.
  Anything named `*Helper` or `*Util` in a feature package is a defect.
- **Some use cases are anemic**, wrapping a single repository call
  (`ListLinksUseCase`). Accepted: a uniform shape is worth more than eight saved files,
  and a reader who finds every operation in the same form navigates faster.
- **The uniform `execute(Command)` shape is interceptable**, which is what makes the
  logging aspect in [03-architecture.md](../03-architecture.md) possible. A service
  layer of differently-shaped methods could not be instrumented the same way. That was
  not the reason for the decision, but it is a real return on it.
- Reversing this means merging classes back into services — mechanical, but it touches
  every controller. Recorded so a future reader does not assume the service layer was
  simply forgotten.
