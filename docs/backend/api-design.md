[Engineering handbook](../../README.md) / Backend

# REST API design

Buildruntime HTTP APIs are resource-oriented JSON APIs, described by an OpenAPI document that FastAPI generates from the code. This page sets the conventions that make one API predictable to someone who has used another. Errors have their own page: [Error handling](error-handling.md).

## Resources and URLs

- Name collections with plural nouns: `/projects`, `/projects/{slug}/members`.
- Use lowercase and hyphens in path segments: `/billing-accounts`, not `/billingAccounts` or `/billing_accounts`.
- Identify a resource by a stable identifier. A slug is fine if it never changes; otherwise use the ID.
- Nest only to express ownership, and no deeper than one level: `/projects/{slug}/members/{id}`, not `/orgs/{org}/projects/{slug}/members/{id}/roles`.
- Put the major version in the path, as `/v1`. See [Versioning](#versioning).

JSON field names use `snake_case`, the same as the Python models. This is a choice, not a law. What matters is that every Buildruntime API makes the same one, and that the frontend's generated types match it.

## Methods

| Method | Use | Safe | Idempotent | Typical success |
| :-- | :-- | :-- | :-- | :-- |
| `GET` | Read a resource or a collection. | Yes | Yes | `200 OK` |
| `POST` | Create a resource in a collection, or perform an action. | No | No | `201 Created` with `Location`, or `200 OK` |
| `PUT` | Replace a resource completely. | No | Yes | `200 OK` |
| `PATCH` | Change some fields of a resource. | No | No | `200 OK` |
| `DELETE` | Remove a resource. | No | Yes | `204 No Content` |

`GET` never changes state, so that caches, crawlers and retries can call it freely.

Some operations are not a field update but a business action with its own rules, such as archiving a project. Model these as `POST` to a sub-resource named after the action: `POST /v1/projects/{slug}/archive`. The intent is explicit, and the endpoint can enforce the rules of the action in one place.

## Status codes

Use the most specific code that applies, and use it consistently across services.

| Code | When |
| :-- | :-- |
| `200 OK` | A successful read, update or action that returns a body. |
| `201 Created` | A resource was created. Include a `Location` header. |
| `202 Accepted` | The request was accepted for asynchronous processing. Return a way to check its status. |
| `204 No Content` | Success with no body, typically `DELETE`. |
| `400 Bad Request` | The request is malformed in a way that is not a validation error, such as an invalid cursor. |
| `401 Unauthorized` | No valid credentials were provided. |
| `403 Forbidden` | The credentials are valid, but do not allow this action. |
| `404 Not Found` | The resource does not exist, or the caller may not know that it does. |
| `409 Conflict` | The request conflicts with the current state, such as a duplicate slug. |
| `412 Precondition Failed` | An `If-Match` precondition did not hold. See [Concurrency control](#concurrency-control). |
| `422 Unprocessable Content` | The body or parameters failed validation. FastAPI returns this for Pydantic errors. |
| `429 Too Many Requests` | A rate limit was exceeded. Include `Retry-After`. |
| `500 Internal Server Error` | An unexpected failure. Never used deliberately. |
| `503 Service Unavailable` | The service is temporarily unable to handle the request. |

Return `404` instead of `403` when revealing that the resource exists would itself leak information, for example another customer's project.

## Request and response bodies

- A response for a single resource is the resource object itself, not wrapped in `{"data": ...}`.
- A collection response is an object with `items` and pagination fields, never a bare array, so that fields can be added later without breaking clients.
- Timestamps are strings in RFC 3339 format, in UTC, with an explicit offset: `"2026-03-12T09:41:07.123Z"` or `"2026-03-12T09:41:07.123+00:00"`.
- Money is an integer amount in the currency's minor unit, with an ISO 4217 currency code: `{"amount": 1999, "currency": "EUR"}`. Never a floating-point number.
- Enumerations are lowercase strings: `"status": "archived"`. Clients must tolerate values they do not recognise, so that new values can be added.
- Absent and `null` mean different things in a `PATCH` body: absent leaves the field unchanged, `null` clears it. In Pydantic, check `model_fields_set` to tell them apart.

```json
{
  "id": "6a0d3c4e-2b1f-4f0e-9a51-3c1d2e7b9f10",
  "slug": "website-redesign",
  "name": "Website redesign",
  "created_at": "2026-03-12T09:41:07.123000Z",
  "archived_at": null
}
```

## Pagination

Collections are paginated with an opaque cursor. Offset pagination (`?page=40`) gets slower as the offset grows, because the database still reads and discards every skipped row, and it skips or repeats items when rows are inserted between requests.

A cursor encodes the sort key of the last item on the page. The next query continues from that position using an index (see [Keyset pagination](../data/postgresql.md#keyset-pagination)).

```text
GET /v1/projects?limit=20
→ 200 {"items": [...20 items...], "next_cursor": "eyJ0Ijo..."}

GET /v1/projects?limit=20&cursor=eyJ0Ijo...
→ 200 {"items": [...7 items...], "next_cursor": null}
```

Clients must treat the cursor as opaque, so that its contents can change. Encoding it as URL-safe Base64 makes that clear, and makes it safe in a query string. Decoding failures are the client's fault, so they raise a domain error that becomes a `400`.

```python
# app/api/pagination.py
import base64
import binascii
import json
from datetime import datetime
from uuid import UUID

from app.core.errors import InvalidInput


class InvalidCursor(InvalidInput):
    code = "invalid_cursor"

    def __init__(self) -> None:
        super().__init__("The cursor is not valid. Start again without a cursor.")


def encode_cursor(created_at: datetime, id: UUID) -> str:
    raw = json.dumps({"t": created_at.isoformat(), "id": str(id)}).encode()
    return base64.urlsafe_b64encode(raw).decode().rstrip("=")


def decode_cursor(cursor: str) -> tuple[datetime, UUID]:
    try:
        padded = cursor + "=" * (-len(cursor) % 4)
        data = json.loads(base64.urlsafe_b64decode(padded))
        return datetime.fromisoformat(data["t"]), UUID(data["id"])
    except (binascii.Error, ValueError, KeyError, TypeError) as exc:
        raise InvalidCursor() from exc
```

`limit` has a default and a maximum. An API that returns every row when asked has no upper bound on its response time.

## Filtering and sorting

- Filter with query parameters named after the field: `?status=archived&owner_id=...`.
- Express ranges with suffixes: `?created_after=2026-01-01T00:00:00Z&created_before=...`.
- Sort with `?sort=created_at` or `?sort=-created_at` for descending, and accept only fields that are indexed. An arbitrary sort on a large table is a slow query.
- Reject unknown parameters with a `422` rather than ignoring them, when the framework makes that practical. A misspelt filter that is silently ignored returns the wrong data.

## Idempotency

Clients retry requests after timeouts and network errors. `GET`, `PUT` and `DELETE` are safe to retry by definition. A retried `POST` can create the same thing twice.

Endpoints that create something with side effects, such as charging a card or sending an email, accept an `Idempotency-Key` header:

1. The client generates a unique key, usually a UUID, for each logical operation, and reuses it for retries of that operation.
2. The server stores the key with a fingerprint of the request and the response, in the same transaction as the operation.
3. A repeat with the same key and the same request returns the stored response without repeating the operation. The same key with a different request returns `422`.
4. Keys expire after a period that the API documents.

For simple creation where the client chooses a natural key, such as a slug, the unique constraint already prevents duplicates, and the retry gets a `409`.

## Concurrency control

When two clients edit the same resource, the second update can silently overwrite the first. Where that matters, use optimistic concurrency:

1. `GET` returns an `ETag` header that identifies the resource's version, for example a `version` column that increments on every update.
2. The client sends it back in `If-Match` on `PUT` or `PATCH`.
3. The server updates only if the version still matches (`UPDATE ... WHERE id = :id AND version = :version`). If no row was updated, it returns `412 Precondition Failed`.

## Versioning

The major version is part of the path: `/v1/projects`. Within a major version, changes must be backwards compatible.

| Compatible, allowed in `v1` | Breaking, needs `v2` |
| :-- | :-- |
| Adding an endpoint | Removing or renaming an endpoint or field |
| Adding an optional request field | Making an optional field required |
| Adding a response field | Changing a field's type or format |
| Adding an enumeration value, given that clients tolerate unknown values | Changing the meaning of a value |
| Relaxing a validation rule | Tightening a validation rule |

Before removing anything, mark it as deprecated in the OpenAPI document (`deprecated=True` on the route or field), and announce a removal date in the changelog. Deprecated endpoints may also return a `Deprecation` header. See [Versioning and releases](../delivery/versioning-and-releases.md).

## Documentation

FastAPI generates the OpenAPI document from the routes and models. Keep it accurate rather than writing separate documentation:

- give every route a `summary`, or a docstring, and every model field a `description` where the name is not enough;
- declare the error responses each route can return, using `responses=` on the route;
- generate the frontend's API types from the OpenAPI document instead of writing them by hand, and check in CI that they are up to date.

## Related

- [Error handling](error-handling.md)
- [FastAPI architecture](fastapi-architecture.md)
- [PostgreSQL](../data/postgresql.md)
