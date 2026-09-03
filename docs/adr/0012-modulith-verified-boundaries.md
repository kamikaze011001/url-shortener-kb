---
status: accepted
date: 2026-09-04
---

# Module boundaries are verified by Spring Modulith, not asserted in a document

The five modules described in [03-architecture.md](../03-architecture.md) —
`identity`, `links`, `redirect`, `analytics`, `shared` — are enforced by
`spring-modulith-starter-core` and a single test:

```java
@Test
void verifiesModularStructure() {
    ApplicationModules.of(UrlShortenerApplication.class).verify();
}
```

A module's root package is its public API; everything in its `internal` sub-package is
invisible to other modules, and reaching into it fails the build. Cross-module access
that is not declared fails the build.

The reason for the decision is narrow: **a boundary that lives only in a document is
not a boundary.** This project's whole premise is that design decisions are written
down and honoured; a claim the build cannot check is a claim that will quietly stop
being true within a week.

## Consequences

- **`redirect` depends on a `links.LinkLookup` port**, declared as
  `@ApplicationModule(allowedDependencies = {"links", "shared"})`. An earlier draft of
  the architecture claimed `redirect` depended on nothing but `shared`; that was only
  achievable by having `redirect` query the `links` table directly, which trades a
  visible code dependency for an invisible data one. The port is the honest version,
  and extraction later turns it into a network call rather than a rewrite.
- **Modulith verifies code dependencies, not data dependencies.** Two modules querying
  the same table is a real coupling that this test will never catch. Stated here
  because the test's green tick is otherwise easy to over-read.
- Controllers live in `internal/web`. Nothing outside a module calls its controllers,
  so they are not module API.
- The `-core` and `-test` starters only. The **event publication registry**
  (`spring-modulith-events-jdbc`) is deliberately excluded: it solves reliable
  asynchronous event delivery, and [ADR-0005](./0005-synchronous-click-recording.md)
  establishes that there is no asynchronous delivery here yet. Adding it would be
  infrastructure for a problem this system does not have.
- Modulith can generate module diagrams from the code
  (`ApplicationModules.documentation()`), which keeps
  [03-architecture.md](../03-architecture.md)'s picture honest for free.
- The constraint is real: package layout is now load-bearing, and moving a class
  between packages can fail the build. That is the feature.
