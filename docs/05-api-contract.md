# 05 — API contract

The machine-readable contract is [`openapi.yaml`](./openapi.yaml). This document is
the part a spec file cannot express: the conventions, and the rule that governs it.

## The governing rule

**[`openapi.yaml`](./openapi.yaml) is hand-written, and it is the source of truth.**

The backend also serves a springdoc-generated spec at `/v3/api-docs`, and a Swagger UI
at `/swagger-ui.html`, because they are useful for exploring during the demo. But:

> If the generated spec and `openapi.yaml` disagree, **`openapi.yaml` is right and the
> code is wrong.** Fix the code, never the contract-to-match-the-code.

Without that rule stated out loud, "contract-first" degrades into contract-decorative
within about two days. The order of work is: change `openapi.yaml` and commit it to
this repository **first**, then implement against it in the backend, then regenerate
the frontend client.

## Base URLs

| Environment | API base | Short link base |
|---|---|---|
| Local | `http://localhost:8081/api/v1` | `http://localhost:8080` |
| Production | `https://app.example.com/api/v1` | `https://s.example.com` |

The two are separate hostnames on purpose — see
[ADR-0006](./adr/0006-two-hostname-topology.md). The redirect endpoint is documented
in the spec but lives on the short host, never under `/api`.

## Versioning

Path-based: `/api/v1/...`. A breaking change means `/api/v2` alongside `v1`, not a
mutation of `v1`.

What counts as breaking: removing a field, renaming a field, narrowing a type, adding
a required request field, changing a status code, changing an error `code` string.
Adding an optional request field or a new response field is not breaking; clients must
ignore unknown fields.

## Authentication

A JWT in an `httpOnly` cookie named `session`. There is no `Authorization` header
anywhere in this API.

```
Set-Cookie: session=<jwt>; HttpOnly; SameSite=Strict; Path=/; Max-Age=3600 [; Secure]
```

`Secure` is present in the `prod` profile and absent in `local`, because it cannot be
set over plain HTTP. The frontend never reads the token — it cannot, and that is the
point. "Am I logged in?" is answered by calling `GET /auth/me`, not by inspecting
storage.

Every `/links/**` endpoint requires it. `401` means no valid session.

## Errors

**RFC 9457 `application/problem+json`**, one shape everywhere, produced by a single
`@RestControllerAdvice`:

```json
{
  "type": "https://example.com/problems/alias-taken",
  "title": "Alias already taken",
  "status": 409,
  "detail": "The alias 'demo' is already in use.",
  "instance": "/api/v1/links",
  "code": "ALIAS_TAKEN",
  "errors": [
    { "field": "alias", "message": "already in use" }
  ]
}
```

`title` and `detail` are for humans and may be reworded at any time. **`code` is for
machines and is part of the contract** — the frontend switches on it, so changing a
`code` string is a breaking change.

### Error codes

| `code` | Status | When |
|---|---|---|
| `VALIDATION_FAILED` | 400 | Malformed request body or parameters |
| `UNAUTHENTICATED` | 401 | Missing, expired, or invalid session |
| `NOT_FOUND` | 404 | Resource does not exist **or is not yours** |
| `ALIAS_TAKEN` | 409 | Requested Alias is in the Code Namespace already |
| `RESERVED_ALIAS` | 409 | Requested Alias is a Reserved Word |
| `EMAIL_TAKEN` | 409 | Registration with an existing email |
| `INVALID_CODE` | 400 | OTP is wrong, already used, or out of attempts |
| `CODE_EXPIRED` | 400 | OTP is past its ten-minute life |
| `EMAIL_NOT_VERIFIED` | 403 | Signed in, but the address is unconfirmed (FR-1.7) |
| `INSUFFICIENT_SCOPE` | 403 | The API Key lacks the scope this endpoint needs (FR-8.10) |
| `FORBIDDEN` | 403 | Allowed for a session, refused for this credential (FR-8.5) |
| `INVALID_DESTINATION` | 422 | Not an absolute `http`/`https` URL |
| `DESTINATION_NOT_ALLOWED` | 422 | Private Destination, or points at this service |
| `RATE_LIMITED` | 429 | Over a limit; `Retry-After` header is present |
| `INTERNAL` | 500 | Anything unhandled. Never leaks a stack trace. |

**The three `403`s are not interchangeable.** `EMAIL_NOT_VERIFIED` says *finish setting
up your account*, `INSUFFICIENT_SCOPE` says *this key was not given that permission*, and
`FORBIDDEN` says *this is a session-only operation, whatever your key holds*. All three
name a fix, which is the reason they are distinguished at all.

**`NOT_FOUND` is deliberately overloaded.** Another Owner's Link returns `404`, not
`403`, because `403` confirms the resource exists. This is the same reasoning that
makes every non-Active Link an identical `404` on the redirect path
([ADR-0008](./adr/0008-soft-delete-and-uniform-404.md)).

**Login failure is `401 UNAUTHENTICATED` for both a wrong password and an unknown
email.** Distinguishing them turns the login form into an account-enumeration oracle.

## Conventions

- **Times** are ISO-8601 with an offset, always UTC: `2026-09-05T14:30:00Z`.
- **Ids** in the API are opaque strings. Treat them as such even though `links.id` is
  a bigint today.
- **Pagination** is page-based: `?page=0&size=20`, responses carry
  `{ content, page, size, totalElements, totalPages }`. Page-based, not cursor-based,
  because the dashboard needs page numbers and the row counts are trivial. Cursor
  pagination is the future change if listing ever gets slow.
- **Partial updates** use `PATCH` with only the fields being changed. An absent field
  means "leave alone"; an explicit `null` means "clear it" — this distinction matters
  for `expiresAt`, which must be clearable.
- **`shortUrl` is always returned fully-formed by the server.** The frontend must
  never build it by concatenating a base URL with a code. One place constructs it, and
  it reads the base from configuration
  ([03-architecture.md](./03-architecture.md)).

## Rate limit responses

Every limited endpoint returns these headers, on success and on rejection:

```
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 3
Retry-After: 42          (on 429 only)
```

The frontend shows the retry delay rather than a generic failure, which is also what
makes step 6 of the demo script legible to an audience.

## Endpoint summary

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `POST` | `/auth/register` | — | Create an Owner |
| `POST` | `/auth/login` | — | Start a session |
| `POST` | `/auth/logout` | ✓ | Clear the session cookie |
| `GET` | `/auth/me` | ✓ | Current Owner |
| `GET` | `/links` | ✓ | List own Links, paginated, searchable |
| `POST` | `/links` | ✓ | Create a Link |
| `GET` | `/links/{id}` | ✓ | One Link |
| `PATCH` | `/links/{id}` | ✓ | Change destination, status, or expiry |
| `DELETE` | `/links/{id}` | ✓ | Soft delete |
| `GET` | `/links/{id}/stats` | ✓ | Click statistics |
| `GET` | `/{code}` | — | **Redirect. Short host only, not under `/api`.** |
