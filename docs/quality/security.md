[Engineering handbook](../../README.md) / Quality

# Security

Security is part of every change, not a separate phase. This page lists the practices every Buildruntime project follows, organised around the [OWASP Top 10](https://owasp.org/www-project-top-ten/) risks that most often apply to web applications. How to report a vulnerability in a Buildruntime project is in the [security policy](https://github.com/buildruntime/.github/blob/main/SECURITY.md).

## Secrets

- Secrets are never committed: not in code, configuration, tests, Dockerfiles or documentation. Enable GitHub secret scanning and push protection on every repository.
- Applications read secrets from environment variables, populated by the platform's secret store (see [Settings](../backend/fastapi-architecture.md#settings)).
- Each environment has its own secrets. A staging credential never works in production.
- If a secret is exposed, in a commit, a log or a screenshot, rotate it straight away. Removing the commit does not undo the exposure.

## Authentication

Do not build authentication from scratch. Use an established identity provider or library, with OpenID Connect or OAuth 2.0 where they apply, and keep its configuration conservative.

Where a service stores passwords itself, hash them with Argon2id, through a maintained library that handles salts and parameters:

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

hasher = PasswordHasher()  # Argon2id, with the library's current recommended parameters.


def hash_password(password: str) -> str:
    return hasher.hash(password)


def verify_password(stored_hash: str, password: str) -> bool:
    try:
        hasher.verify(stored_hash, password)
    except VerifyMismatchError:
        return False
    return True


def needs_rehash(stored_hash: str) -> bool:
    """True if the hash used older parameters. Rehash after the next successful sign-in."""
    return hasher.check_needs_rehash(stored_hash)
```

For browser sessions:

- Session cookies are `HttpOnly`, `Secure`, and `SameSite=Lax` or `Strict`.
- Rotate the session identifier on sign-in, and invalidate it on the server on sign-out.
- Rate-limit sign-in and password reset by account and by client address (see [Rate limiting](../data/redis.md#rate-limiting)).
- Reset links and one-time codes are single use and expire quickly.

Bearer tokens such as JWTs are for service-to-service calls and APIs used by non-browser clients. If you use JWTs, verify the signature with an explicit algorithm, and check the expiry, issuer and audience. Never accept `alg: none`. Do not store tokens in `localStorage`, where any script on the page can read them.

## Authorisation

Most access control failures are a missing check, not a broken one.

- **Check permissions on the server, for every request.** Hiding a button in the interface is not access control. In Next.js, every Server Action and route handler checks the user's permissions itself, because each is a public endpoint (see [Server Actions](../frontend/nextjs.md#server-actions)).
- **Check in one place per operation**, in the service layer, where the use case is implemented. Checks scattered across routers get missed when a second entry point is added.
- **Scope queries to what the user may see.** `WHERE id = :id AND account_id = :account_id` cannot return another account's row, whatever ID is sent. Loading the row first and then checking is easy to forget.
- **Deny by default.** A new endpoint, role or resource type has no access until it is granted.

```python
from dataclasses import dataclass
from uuid import UUID

from app.core.errors import PermissionDenied


@dataclass(frozen=True)
class Actor:
    user_id: UUID
    roles: frozenset[str]


def require_role(actor: Actor, role: str) -> None:
    if role not in actor.roles:
        raise PermissionDenied(f"This action needs the {role} role.")
```

## Input handling

Treat every input as untrusted: request bodies, query parameters, headers, cookies, uploaded files, webhooks, and data from other services.

- **Validate at the boundary** with Pydantic on the backend and Zod on the frontend, against an allow-list of what is valid: types, lengths, formats, ranges. Reject requests with unknown fields.
- **Never build SQL from strings.** Use SQLAlchemy expressions, or bound parameters with `text()`. This applies to `ORDER BY` columns too: map the user's choice to a column from a fixed list.

```python
from sqlalchemy import text

slug = "website-redesign'; DROP TABLE projects; --"

# Wrong: the input becomes part of the SQL.
unsafe = text(f"SELECT id FROM projects WHERE slug = '{slug}'")

# Right: the input is sent separately, as a parameter.
safe = text("SELECT id FROM projects WHERE slug = :slug").bindparams(slug=slug)
```

- **Do not render untrusted HTML.** React escapes text by default. `dangerouslySetInnerHTML` is allowed only for HTML that has been sanitised with a maintained sanitiser such as DOMPurify, and the pull request says why it is needed.
- **Check URLs before the server fetches them.** A feature that fetches a user-supplied URL, such as a webhook or an image import, can be used to reach internal services. Allow only `https`, resolve the host, and reject private, loopback and link-local addresses. Do not follow redirects without checking the new address.
- **Uploaded files** are checked by content, not by extension or the client's `Content-Type`, have a size limit, are stored outside the web root under a generated name, and are served with `Content-Disposition: attachment` unless they are known-safe images.

## Browser security

- **HTTPS everywhere**, with `Strict-Transport-Security`.
- **Security headers** on every response: see the Next.js configuration in [Next.js](../frontend/nextjs.md#configuration). Add a Content Security Policy, tested in a production build.
- **CORS** lists the exact origins that may call an API with credentials. Never reflect the request's `Origin`, and never combine `*` with credentials.

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "Idempotency-Key"],
)
```

- **Cross-site request forgery.** Cookie-authenticated endpoints that change state need protection beyond `SameSite`. Next.js Server Actions accept only `POST` and compare the `Origin` header with the host. Custom endpoints need an equivalent check or a CSRF token.

## Database access

- The application connects with a role that can read and write its own tables and nothing else. It does not own the schema, and it is not a superuser.
- Migrations run with a separate role that may change the schema.
- Set `statement_timeout` for the application's role (see [Timeouts](../data/postgresql.md#timeouts)).
- Encrypt connections to databases outside a private network (`sslmode=verify-full`).
- Backups are encrypted, and restoring one is tested.

## Dependencies

Most of the code in a project comes from its dependencies, and their vulnerabilities are the project's vulnerabilities.

- Commit lock files, and install from them in CI and in images (`uv sync --locked`, `npm ci`).
- Enable Dependabot alerts and security updates. Configure version updates for each package ecosystem, including GitHub Actions and Docker:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: uv
    directory: /
    schedule:
      interval: weekly
    groups:
      minor-and-patch:
        update-types: [minor, patch]
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly
```

- Review new dependencies before adding them: maintenance activity, number of maintainers, licence, and whether the project needs it at all. A small function does not need a package.
- Scan container images for known vulnerabilities in CI, and rebuild images regularly to pick up base image fixes.

## Logging and privacy

- Log security-relevant events: sign-ins, failed sign-ins, permission denials, changes to roles and credentials. Include who, what and when, but no secrets.
- Do not log passwords, tokens, session identifiers, full card numbers or personal data beyond what is needed to investigate a problem.
- Collect only the personal data a feature needs, and delete it when it is no longer needed.

## Reviewing for security

Pay particular attention in review to changes that touch authentication, authorisation, cryptography, file handling, outbound requests to user-supplied URLs, raw SQL, and new dependencies (see [Code review](../workflow/code-review.md)).

## Related

- [Error handling](../backend/error-handling.md), which keeps internal details out of responses
- [CI/CD](../delivery/ci-cd.md#secrets)
- [Buildruntime security policy](https://github.com/buildruntime/.github/blob/main/SECURITY.md)
