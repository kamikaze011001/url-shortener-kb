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

- **`redirect` reaches other modules only through ports**, declared as
  `@ApplicationModule(allowedDependencies = {"links", "analytics", "shared"})`: the
  `links.LinkLookup` port to resolve a Short Code, and the `analytics.ClickRecorder`
  port to record the Click. An earlier draft of the architecture claimed `redirect`
  depended on nothing but `shared`; that was only achievable by having `redirect` query
  the `links` table directly, which trades a visible code dependency for an invisible
  data one. The ports are the honest version, and extraction later turns a method call
  into a network call rather than a rewrite.

  > This ADR originally listed `{"links", "shared"}` and omitted `analytics`, which was
  > simply wrong — a Redirect records a Click, so the dependency was always there. The
  > error surfaced the moment the modules were declared in code, which is an argument
  > for [ADR-0012](./0012-modulith-verified-boundaries.md) rather than against it: a
  > dependency list nothing checks is a dependency list nobody gets right.
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
