---
status: accepted
date: 2026-09-03
---

# The hand-written OpenAPI file is the contract, not the generated one

[`docs/openapi.yaml`](../openapi.yaml) in this repository is written by hand and is the
source of truth for the wire format. The backend implements it; the frontend generates
its TypeScript client from it. The backend *also* serves a springdoc-generated spec at
`/v3/api-docs` for exploration, but that document has no authority:

> Where the generated spec and `openapi.yaml` disagree, **`openapi.yaml` is right and
> the code is wrong.**

That sentence is the decision. Without it stated explicitly, contract-first degrades
into contract-decorative within days — the generated spec quietly becomes the real one,
and the hand-written file rots into documentation nobody trusts.

## Considered options

**Generate the spec from Java annotations (springdoc as the source of truth).** What
most projects actually do, and it is never out of date with the code. Rejected because
it makes the contract a *consequence* of implementation choices rather than a
constraint on them: the shape of a DTO, the default serialisation of a nullable field,
an accidental rename all become API changes that nobody decided. It also means the
frontend cannot be built against a contract that does not exist yet.

## Consequences

- **Order of work is fixed**: change `openapi.yaml` and commit it here first, then
  implement in the backend, then regenerate the frontend client. A backend change that
  alters the wire format without a prior commit in this repository is a defect,
  regardless of whether it works.
- The two specs can drift, and nothing automatically catches it. A schema-diff check in
  CI is the obvious fix and is deliberately not built for a 14-hour demo; the rule above
  is enforced by discipline until then. This is the weakest point of the decision and is
  recorded as such.
- Writing the YAML first costs about 45 minutes and removes the entire category of
  frontend/backend field-name mismatches, which is a good trade when the frontend and
  backend are built by the same person on the same weekend and a mismatch is discovered
  at midnight.
- The error `code` enum in the spec is part of the contract; `title` and `detail` are
  not. The frontend switches on `code` only.
