<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/buildruntime/buildruntime/main/assets/brand/logo-horizontal-dark.svg">
  <img alt="Buildruntime" src="https://raw.githubusercontent.com/buildruntime/buildruntime/main/assets/brand/logo-horizontal-light.svg" height="24">
</picture>

# Engineering handbook

How Buildruntime designs, builds, tests and ships software. The handbook sets the defaults for Buildruntime projects: the architecture we start from, the conventions we follow, and the checks a change has to pass before it reaches production.

It is written for the people doing the work. Each page states a default, explains why it is the default, and shows it in code.

## Contents

### Principles

| Page | Covers |
| :-- | :-- |
| [Engineering principles](docs/principles/engineering-principles.md) | The rules behind every other page, and how decisions are recorded. |
| [Clean architecture](docs/principles/clean-architecture.md) | Layers, the dependency rule, and how far to take it. |

### Frontend

| Page | Covers |
| :-- | :-- |
| [Frontend standards](docs/frontend/frontend-standards.md) | Project structure, accessibility, styling, state and Core Web Vitals. |
| [TypeScript](docs/frontend/typescript.md) | Compiler settings, modelling with types, and validating data at the boundary. |
| [React](docs/frontend/react.md) | Components, state, effects, hooks and forms. |
| [Next.js](docs/frontend/nextjs.md) | App Router, Server and Client Components, data fetching, caching and Server Actions. |

### Backend

| Page | Covers |
| :-- | :-- |
| [FastAPI architecture](docs/backend/fastapi-architecture.md) | Project layout, settings, dependencies, and the router, service and repository layers. |
| [REST API design](docs/backend/api-design.md) | Resources, methods, status codes, pagination, idempotency and versioning. |
| [Error handling](docs/backend/error-handling.md) | Domain errors, Problem Details responses, logging and the client side. |

### Data

| Page | Covers |
| :-- | :-- |
| [Database design](docs/data/database-design.md) | Naming, keys, constraints, data types and migrations. |
| [PostgreSQL](docs/data/postgresql.md) | Indexes, query plans, transactions, locking and zero-downtime schema changes. |
| [Redis](docs/data/redis.md) | Caching, rate limiting, key design and what not to use Redis for. |

### Delivery

| Page | Covers |
| :-- | :-- |
| [Docker](docs/delivery/docker.md) | Images for Python and Next.js services, and local development with Compose. |
| [CI/CD](docs/delivery/ci-cd.md) | Pipeline stages, a GitHub Actions workflow, deployment and rollback. |
| [Versioning and releases](docs/delivery/versioning-and-releases.md) | Semantic Versioning, changelogs, tags and release steps. |

### Workflow

| Page | Covers |
| :-- | :-- |
| [Git workflow](docs/workflow/git-workflow.md) | Trunk-based development, branches, rebasing and history. |
| [Conventional Commits](docs/workflow/conventional-commits.md) | Commit message format, types, scopes and breaking changes. |
| [Pull requests](docs/workflow/pull-requests.md) | Size, description, checks and merging. |
| [Code review](docs/workflow/code-review.md) | What reviewers look for, how to write comments, and how authors respond. |

### Quality

| Page | Covers |
| :-- | :-- |
| [Testing strategy](docs/quality/testing-strategy.md) | What to test at which level, test data and flaky tests. |
| [Pytest](docs/quality/pytest.md) | Layout, fixtures, database isolation and API tests. |
| [Playwright](docs/quality/playwright.md) | Locators, assertions, authentication and running in CI. |
| [Security](docs/quality/security.md) | Secrets, authentication, authorisation, input handling and dependencies. |
| [Performance](docs/quality/performance.md) | Measuring, budgets, and the usual causes of slow backends and frontends. |

## How to read this handbook

Guidance uses three levels of obligation:

- **Must**: required. A project that departs from it records the reason in a decision record (see [Recording decisions](docs/principles/engineering-principles.md#recording-decisions)).
- **Should**: the default. Depart from it when you have a reason you could explain in review.
- **May**: an option that is often useful, with no expectation either way.

Code examples are complete enough to copy. File paths in the first line of a block, such as `# app/projects/service.py`, show where the code lives in a project laid out as described in [FastAPI architecture](docs/backend/fastapi-architecture.md) or [Next.js](docs/frontend/nextjs.md). Diagrams are written in Mermaid and render on GitHub.

## Versions

The examples were run, not only read, against the following versions. They do not use features that are specific to the latest minor release unless the text says so.

| Area | Checked with |
| :-- | :-- |
| Python | Python 3.13, FastAPI 0.162, Pydantic 2.14, SQLAlchemy 2.1, Alembic 1.20, pytest 9.2 |
| JavaScript | Node.js 22 and 24, TypeScript 7.0, React 19.3, Next.js 16.4, Zod 4.6, Playwright 1.64 |
| Data | PostgreSQL 16, Redis 7 |
| Delivery | Docker Engine 29, Docker Compose 5. GitHub Actions workflows were checked with actionlint, not run. |

## Changing the handbook

The handbook changes through pull requests, like code. A pull request that changes a standard explains the problem with the current guidance and what the change means for existing projects. Follow [Pull requests](docs/workflow/pull-requests.md) and the Buildruntime [contributing guide](https://github.com/buildruntime/.github/blob/main/CONTRIBUTING.md).

## Licence

The content of this repository, including the code examples, is licensed under the [Creative Commons Attribution 4.0 International License](LICENSE) (CC BY 4.0).
