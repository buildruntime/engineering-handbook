[Engineering handbook](../../README.md) / Backend

# Error handling

Errors are part of an API's contract. A client should be able to tell, from the response alone, what went wrong, whether retrying could help, and what to show the user. This page covers how backend services classify, report and log errors, and how the frontend consumes them.

## Two kinds of errors

| | Expected errors | Unexpected errors |
| :-- | :-- | :-- |
| Examples | Validation failures, missing resources, conflicts, permission checks. | Bugs, a database that is down, a timeout from another service. |
| Raised as | A `DomainError` subclass, or a Pydantic validation error. | Any other exception. |
| Response | A specific 4xx status with a helpful message. | `500` or `503` with a generic message. Internal details never reach the client. |
| Logged as | `INFO`, or not at all. They are normal behaviour. | `ERROR` with a stack trace. Someone should look at it. |

Do not catch exceptions you cannot handle. An unexpected error should travel up to the global handler, which logs it once with full context. A `try/except` that logs and re-raises at every layer produces the same error five times.

## Problem Details

Error responses use the Problem Details format from [RFC 9457](https://www.rfc-editor.org/rfc/rfc9457), with the media type `application/problem+json`. Every Buildruntime service returns the same shape, so that clients can handle errors from any of them with one function.

```json
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "The slug 'website-redesign' is already in use.",
  "code": "slug_taken",
  "request_id": "0d4c1b52f1f84e5c9d0b2a7e6f3c8a91"
}
```

| Member | Meaning |
| :-- | :-- |
| `type` | `about:blank`, meaning that the status code says all there is to say about the type. A service may use a URI it controls instead, pointing at documentation for the error. |
| `title` | The HTTP status phrase. |
| `status` | The HTTP status code, repeated for clients that only see the body. |
| `detail` | A sentence for a human, explaining this occurrence. Clients may show it, but must not parse it. |
| `code` | A stable, machine-readable identifier. Clients branch on this. Changing it is a breaking change. |
| `request_id` | Matches the `X-Request-ID` response header and the server's log entries. |
| `errors` | For validation failures only: one entry per invalid field. |

## Mapping errors to responses

The interface layer owns the mapping from domain errors to status codes. The domain layer never mentions HTTP (see [Clean architecture](../principles/clean-architecture.md#the-dependency-rule)).

```python
# app/api/problems.py
import logging
import uuid
from http import HTTPStatus
from typing import Any

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException
from starlette.middleware.base import BaseHTTPMiddleware, RequestResponseEndpoint
from starlette.responses import Response

from app.core.errors import Conflict, DomainError, InvalidInput, NotFound, PermissionDenied

logger = logging.getLogger(__name__)

STATUS_FOR_ERROR: list[tuple[type[DomainError], HTTPStatus]] = [
    (NotFound, HTTPStatus.NOT_FOUND),
    (Conflict, HTTPStatus.CONFLICT),
    (PermissionDenied, HTTPStatus.FORBIDDEN),
    (InvalidInput, HTTPStatus.BAD_REQUEST),
]


def problem(
    request: Request, status: HTTPStatus, detail: str, code: str, **extra: Any
) -> JSONResponse:
    request_id: str = request.state.request_id
    body = {
        "type": "about:blank",
        "title": status.phrase,
        "status": status.value,
        "detail": detail,
        "code": code,
        "request_id": request_id,
        **extra,
    }
    return JSONResponse(
        body,
        status_code=status.value,
        media_type="application/problem+json",
        headers={"X-Request-ID": request_id},
    )


async def handle_domain_error(request: Request, exc: Exception) -> Response:
    assert isinstance(exc, DomainError)  # noqa: S101  # Registered for DomainError only.
    status = next(
        (status for cls, status in STATUS_FOR_ERROR if isinstance(exc, cls)),
        HTTPStatus.BAD_REQUEST,
    )
    return problem(request, status, exc.message, exc.code)


async def handle_validation_error(request: Request, exc: Exception) -> Response:
    assert isinstance(exc, RequestValidationError)  # noqa: S101
    errors = [
        {"location": ".".join(str(part) for part in error["loc"]), "message": error["msg"]}
        for error in exc.errors()
    ]
    return problem(
        request,
        HTTPStatus.UNPROCESSABLE_CONTENT,
        "The request is not valid. See errors for details.",
        "validation_failed",
        errors=errors,
    )


async def handle_http_exception(request: Request, exc: Exception) -> Response:
    """Unknown routes, wrong methods, and HTTPException raised by dependencies."""
    assert isinstance(exc, HTTPException)  # noqa: S101
    status = HTTPStatus(exc.status_code)
    response = problem(request, status, str(exc.detail), status.name.lower())
    response.headers.update(exc.headers or {})
    return response


async def handle_unexpected_error(request: Request, exc: Exception) -> Response:
    logger.exception("Unhandled error", extra={"request_id": request.state.request_id})
    return problem(
        request,
        HTTPStatus.INTERNAL_SERVER_ERROR,
        "Something went wrong on our side. The error has been logged.",
        "internal_error",
    )


class RequestIdMiddleware(BaseHTTPMiddleware):
    """Give every request an ID, and return it in the X-Request-ID header."""

    async def dispatch(self, request: Request, call_next: RequestResponseEndpoint) -> Response:
        incoming = request.headers.get("X-Request-ID", "")
        is_valid = incoming.isalnum() and len(incoming) <= 64
        request.state.request_id = incoming if is_valid else uuid.uuid4().hex
        response = await call_next(request)
        response.headers["X-Request-ID"] = request.state.request_id
        return response


def register_problem_handlers(app: FastAPI) -> None:
    app.add_exception_handler(DomainError, handle_domain_error)
    app.add_exception_handler(RequestValidationError, handle_validation_error)
    app.add_exception_handler(HTTPException, handle_http_exception)
    app.add_exception_handler(Exception, handle_unexpected_error)
    app.add_middleware(RequestIdMiddleware)
```

Notes on this module:

- **The order of `STATUS_FOR_ERROR` matters.** A subclass is matched by the first base class in the list, so list more specific classes first if you add any.
- **`HTTPException` is handled too**, so that a `404` for an unknown route or a `429` from a dependency has the same shape as every other error. Its code is the status name, such as `not_found` or `too_many_requests`.
- **The request ID from the client is accepted only if it looks like one.** It ends up in logs, so arbitrary input there is a log injection risk.
- **The handler for `Exception` produces the response, and Starlette then re-raises the exception** so that the server logs it too. Test clients see the re-raised exception unless they are configured not to raise it (see [Pytest](../quality/pytest.md#api-tests)).
- **Validation messages come from Pydantic.** They describe the field, not the system, so they are safe to return.

## Writing good error messages

The `detail` is read by developers integrating with the API, and often shown to end users.

- Say what was wrong and, if it is not obvious, how to fix it: "The slug 'Website' is not valid. Use lowercase letters, numbers and hyphens."
- Refer to what the client sent, not to internals: no table names, SQL, stack traces or file paths.
- Do not guess at causes you have not checked. "The payment provider did not respond" is better than "Your card was declined" when you only know about a timeout.
- Do not reveal whether something exists to someone who may not know: "No project has the slug 'x'" is right for a member of the account, and wrong for a stranger.

## Logging

Services log JSON to standard output, one object per line, and the platform collects it. Every entry includes the request ID, so that the logs for one request can be found from the ID a user reports.

- Log unexpected errors once, at the global handler, with the stack trace.
- Log expected errors at `INFO` at most. A spike of `409` responses is a metric, not an incident.
- Never log secrets, tokens, passwords, full card numbers or request bodies that may contain personal data. Log identifiers instead.
- Log at the start and end of work that runs outside a request, such as jobs, with the job's ID and duration.

## Retries

A client may retry a request when:

- it got `502`, `503` or `504`, or the connection failed;
- it got `429`, after waiting for the time in `Retry-After`;
- the request is idempotent, or carries an `Idempotency-Key` (see [Idempotency](api-design.md#idempotency)).

Retry with exponential backoff and jitter, and give up after a few attempts. Never retry `4xx` errors other than `408` and `429`: the same request will fail the same way.

## Handling errors on the frontend

The frontend reads the Problem Details body into a typed error, and branches on `code`, never on `detail`.

```ts
// lib/api/problem.ts
export type ProblemDetails = {
  type: string;
  title: string;
  status: number;
  detail: string;
  code: string;
  request_id?: string;
  errors?: { location: string; message: string }[];
};

export class ApiError extends Error {
  constructor(readonly problem: ProblemDetails) {
    super(problem.detail);
    this.name = "ApiError";
  }

  get code(): string {
    return this.problem.code;
  }
}

function isProblemDetails(value: unknown): value is ProblemDetails {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as ProblemDetails).status === "number" &&
    typeof (value as ProblemDetails).code === "string"
  );
}

export async function toApiError(response: Response): Promise<ApiError> {
  const body: unknown = await response.json().catch(() => null);
  if (isProblemDetails(body)) return new ApiError(body);
  return new ApiError({
    type: "about:blank",
    title: response.statusText,
    status: response.status,
    detail: "The server returned an unexpected response.",
    code: "unexpected_response",
  });
}
```

How the error reaches the user depends on where it happens. In a Server Action, expected errors are returned as state for the form to show. Unexpected errors are thrown, and the nearest `error.tsx` boundary renders them. See [Next.js](../frontend/nextjs.md#server-actions).

## Related

- [REST API design](api-design.md)
- [FastAPI architecture](fastapi-architecture.md)
- [Security](../quality/security.md)
