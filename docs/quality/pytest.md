[Engineering handbook](../../README.md) / Quality

# Pytest

Python services are tested with pytest. This page shows the layout and fixtures for the service from [FastAPI architecture](../backend/fastapi-architecture.md): unit tests with fakes, API tests against a real PostgreSQL database, and integration tests against Redis. What to test at each level is covered in [Testing strategy](testing-strategy.md).

## Layout

```text
tests/
├── conftest.py              # Shared fixtures: settings, database, client
├── fakes.py                 # In-memory implementations of ports
├── unit/
│   └── test_project_service.py
├── api/
│   └── test_projects.py
└── integration/
    └── test_redis.py
```

Pytest settings live in `pyproject.toml` (see [Tooling](../backend/fastapi-architecture.md#tooling)). Two of them matter for this layout:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
addopts = ["-ra", "--strict-markers", "--strict-config"]
xfail_strict = true
```

`pythonpath = ["."]` makes `app` and `tests` importable without installing the project. `--strict-markers` turns a misspelt marker into an error instead of a test that silently never runs.

## Writing tests

- Name tests after the behaviour they check: `test_archive_twice_fails`, not `test_archive_2`.
- One behaviour per test. Several asserts are fine if they describe one outcome.
- Arrange, act, assert, in that order, with no logic in between. A test with an `if` in it tests two things.
- Use plain `assert`. Pytest shows both sides of a failed comparison.
- Use `pytest.raises` with the most specific exception, and `match=` when the message matters.
- Use `pytest.mark.parametrize` for the same behaviour with different inputs, not a loop inside one test.

## Async tests

Async tests run with the AnyIO pytest plugin, which is installed with FastAPI. Mark each module with `pytestmark = pytest.mark.anyio`, and define the backend once, at session scope, so that session-scoped async fixtures share one event loop:

```python
# tests/conftest.py
import os
from collections.abc import AsyncIterator

import pytest
from fastapi import FastAPI
from httpx import ASGITransport, AsyncClient
from redis.asyncio import Redis
from sqlalchemy.ext.asyncio import AsyncConnection, AsyncEngine, AsyncSession, create_async_engine

from app.core.config import Settings
from app.db.base import Base
from app.db.session import get_session
from app.main import create_app


@pytest.fixture(scope="session")
def anyio_backend() -> str:
    return "asyncio"


@pytest.fixture(scope="session")
def settings() -> Settings:
    return Settings(
        environment="test",
        database_url=os.environ.get(
            "APP_TEST_DATABASE_URL",
            "postgresql+asyncpg://postgres:postgres@localhost:5432/projects_test",
        ),
        redis_url=os.environ.get("APP_TEST_REDIS_URL", "redis://localhost:6379/15"),
    )


@pytest.fixture(scope="session")
async def engine(settings: Settings) -> AsyncIterator[AsyncEngine]:
    engine = create_async_engine(str(settings.database_url))
    async with engine.begin() as connection:
        await connection.run_sync(Base.metadata.drop_all)
        await connection.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()


@pytest.fixture
async def connection(engine: AsyncEngine) -> AsyncIterator[AsyncConnection]:
    async with engine.connect() as connection:
        transaction = await connection.begin()
        yield connection
        await transaction.rollback()


@pytest.fixture
async def session(connection: AsyncConnection) -> AsyncIterator[AsyncSession]:
    async with AsyncSession(
        bind=connection, join_transaction_mode="create_savepoint", expire_on_commit=False
    ) as session:
        yield session


@pytest.fixture
def app(settings: Settings, session: AsyncSession) -> FastAPI:
    app = create_app(settings)

    async def override_get_session() -> AsyncIterator[AsyncSession]:
        async with session.begin_nested():
            yield session

    app.dependency_overrides[get_session] = override_get_session
    return app


@pytest.fixture
async def client(app: FastAPI) -> AsyncIterator[AsyncClient]:
    transport = ASGITransport(app=app, raise_app_exceptions=False)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client


@pytest.fixture
async def redis(settings: Settings) -> AsyncIterator[Redis]:
    client = Redis.from_url(str(settings.redis_url))
    await client.flushdb()
    yield client
    await client.flushdb()
    await client.aclose()
```

## Database isolation

API tests run against a real PostgreSQL database, because the behaviour under test includes the database: constraints, `ON CONFLICT`, ordering and types. An in-memory substitute such as SQLite behaves differently in exactly those places.

Each test runs inside a transaction that is rolled back at the end, so tests cannot see each other's data and need no clean-up:

1. `connection` opens a connection and begins a transaction.
2. `session` binds a session to that connection with `join_transaction_mode="create_savepoint"`. Anything the code under test commits only releases a savepoint, inside the outer transaction.
3. `app` replaces the application's session dependency with one that uses that session, and `client` sends requests to it.
4. After the test, `connection` rolls back the outer transaction.

The schema is created once per test run, from the models. A separate CI step runs the migrations and `alembic check` (see [Migrations](../data/database-design.md#migrations)), so the tests do not need to.

Use a dedicated test database. The `engine` fixture drops every table it knows about.

## Unit tests with fakes

The service depends on a port, so a unit test can give it an in-memory implementation. A fake is a small, working implementation of the interface. Prefer it to a mock that records calls: a fake lets the test check results, and does not break when the service changes the order of its calls.

```python
# tests/fakes.py
from datetime import datetime
from uuid import UUID

from app.projects.domain import Project, SlugTaken


class InMemoryProjectRepository:
    def __init__(self) -> None:
        self.projects: dict[str, Project] = {}
        self.reads = 0

    async def get_by_slug(self, slug: str) -> Project | None:
        self.reads += 1
        return self.projects.get(slug)

    async def add(self, project: Project) -> None:
        if project.slug in self.projects:
            raise SlugTaken(project.slug)
        self.projects[project.slug] = project

    async def update(self, project: Project) -> None:
        self.projects[project.slug] = project

    async def list_page(self, *, after: tuple[datetime, UUID] | None, limit: int) -> list[Project]:
        ordered = sorted(self.projects.values(), key=lambda p: (p.created_at, p.id), reverse=True)
        if after is not None:
            ordered = [p for p in ordered if (p.created_at, p.id) < after]
        return ordered[:limit]


class FixedClock:
    def __init__(self, now: datetime) -> None:
        self.now = now

    def __call__(self) -> datetime:
        return self.now
```

```python
# tests/unit/test_project_service.py
from datetime import UTC, datetime, timedelta

import pytest

from app.projects.domain import ProjectArchived, ProjectNotFound, SlugTaken
from app.projects.service import ProjectService
from tests.fakes import FixedClock, InMemoryProjectRepository

pytestmark = pytest.mark.anyio


@pytest.fixture
def clock() -> FixedClock:
    return FixedClock(datetime(2026, 3, 12, 9, 0, tzinfo=UTC))


@pytest.fixture
def service(clock: FixedClock) -> ProjectService:
    return ProjectService(InMemoryProjectRepository(), clock=clock)


async def test_create_records_creation_time(service: ProjectService, clock: FixedClock) -> None:
    project = await service.create(name="Website redesign", slug="website-redesign")

    assert project.created_at == clock.now
    assert not project.is_archived


async def test_create_rejects_a_slug_in_use(service: ProjectService) -> None:
    await service.create(name="Website redesign", slug="website-redesign")

    with pytest.raises(SlugTaken):
        await service.create(name="Another project", slug="website-redesign")


async def test_archive_records_the_time(service: ProjectService, clock: FixedClock) -> None:
    await service.create(name="Website redesign", slug="website-redesign")
    clock.now += timedelta(days=3)

    archived = await service.archive("website-redesign")

    assert archived.archived_at == clock.now


async def test_archive_twice_fails(service: ProjectService) -> None:
    await service.create(name="Website redesign", slug="website-redesign")
    await service.archive("website-redesign")

    with pytest.raises(ProjectArchived, match="already archived"):
        await service.archive("website-redesign")


async def test_get_unknown_project_fails(service: ProjectService) -> None:
    with pytest.raises(ProjectNotFound):
        await service.get("does-not-exist")
```

## API tests

API tests send real HTTP requests to the application, through `httpx` and an ASGI transport, with no server and no network. They check the contract: status codes, headers and response bodies.

The `client` fixture sets `raise_app_exceptions=False`. Without it, the test client re-raises unexpected exceptions instead of returning the `500` response, and the error handler cannot be tested.

```python
# tests/api/test_projects.py
import pytest
from fastapi import FastAPI
from httpx import AsyncClient, Response

from app.projects.router import get_project_service

pytestmark = pytest.mark.anyio


async def create(client: AsyncClient, slug: str, name: str = "A project") -> Response:
    return await client.post("/v1/projects", json={"name": name, "slug": slug})


async def test_create_project(client: AsyncClient) -> None:
    response = await create(client, "website-redesign", name="Website redesign")

    assert response.status_code == 201
    assert response.headers["location"] == "/v1/projects/website-redesign"
    assert response.json()["name"] == "Website redesign"
    assert response.json()["archived_at"] is None


async def test_duplicate_slug_is_a_conflict(client: AsyncClient) -> None:
    await create(client, "website-redesign")

    response = await create(client, "website-redesign")

    assert response.status_code == 409
    assert response.headers["content-type"] == "application/problem+json"
    assert response.json()["code"] == "slug_taken"


@pytest.mark.parametrize("slug", ["Website", "two--hyphens", "-leading", "", "a" * 64])
async def test_invalid_slug_is_rejected(client: AsyncClient, slug: str) -> None:
    response = await create(client, slug)

    assert response.status_code == 422
    assert response.json()["errors"][0]["location"] == "body.slug"


async def test_pagination_returns_each_project_once(client: AsyncClient) -> None:
    slugs = {f"project-{n}" for n in range(5)}
    for slug in slugs:
        await create(client, slug)

    seen: list[str] = []
    params: dict[str, str | int] = {"limit": 2}
    while True:
        page = (await client.get("/v1/projects", params=params)).json()
        seen += [item["slug"] for item in page["items"]]
        if page["next_cursor"] is None:
            break
        params["cursor"] = page["next_cursor"]

    assert sorted(seen) == sorted(slugs)


async def test_invalid_cursor_is_a_bad_request(client: AsyncClient) -> None:
    response = await client.get("/v1/projects", params={"cursor": "not-a-cursor"})

    assert response.status_code == 400
    assert response.json()["code"] == "invalid_cursor"


async def test_archive_twice_is_a_conflict(client: AsyncClient) -> None:
    await create(client, "website-redesign")
    await client.post("/v1/projects/website-redesign/archive")

    response = await client.post("/v1/projects/website-redesign/archive")

    assert response.status_code == 409
    assert response.json()["code"] == "project_archived"


async def test_unknown_route_returns_problem_details(client: AsyncClient) -> None:
    response = await client.get("/v1/nothing-here")

    assert response.status_code == 404
    assert response.json()["code"] == "not_found"


async def test_unexpected_errors_do_not_leak_details(app: FastAPI, client: AsyncClient) -> None:
    def broken_service() -> None:
        raise RuntimeError("connection to db-internal-7 refused")

    app.dependency_overrides[get_project_service] = broken_service

    response = await client.get("/v1/projects/website-redesign")

    assert response.status_code == 500
    assert response.json()["code"] == "internal_error"
    assert "db-internal-7" not in response.text
    assert response.headers["x-request-id"] == response.json()["request_id"]
```

## Integration tests

Code that talks to Redis is tested against a real Redis, on a database number reserved for tests. Time-dependent code takes its time from something the test can control. Here, `monkeypatch` freezes the clock that the rate limiter reads, so the test cannot fail when it runs across a window boundary.

```python
# tests/integration/test_redis.py
from datetime import UTC, datetime, timedelta
from uuid import uuid4

import pytest
from redis.asyncio import Redis

from app.core.cache import JsonCache
from app.core.rate_limit import hit
from app.projects.cached_repository import CachedProjectRepository
from app.projects.domain import Project
from tests.fakes import InMemoryProjectRepository

pytestmark = pytest.mark.anyio


async def test_rate_limit_blocks_after_the_limit(
    redis: Redis, monkeypatch: pytest.MonkeyPatch
) -> None:
    # 20 seconds into a 60-second window.
    monkeypatch.setattr("app.core.rate_limit.time.time", lambda: 999_980.0)

    results = [await hit(redis, "test", limit=3, window_seconds=60) for _ in range(4)]

    assert [result.allowed for result in results] == [True, True, True, False]
    assert results[-1].retry_after == 40


async def test_cached_repository_reads_through_once(redis: Redis) -> None:
    inner = InMemoryProjectRepository()
    project = Project(
        id=uuid4(), slug="website-redesign", name="Website redesign", created_at=datetime.now(UTC)
    )
    await inner.add(project)
    cache = JsonCache(redis, namespace="test:project", type_=Project, ttl=timedelta(minutes=5))
    repository = CachedProjectRepository(inner, cache)

    first = await repository.get_by_slug("website-redesign")
    second = await repository.get_by_slug("website-redesign")

    assert first == second == project
    assert inner.reads == 1
```

## Running tests

```bash
uv run pytest                          # Everything
uv run pytest tests/unit               # Unit tests only, no services needed
uv run pytest -k archive               # Tests whose name contains "archive"
uv run pytest --lf                     # Only the tests that failed last time
uv run pytest -x --pdb                 # Stop at the first failure and open the debugger
```

Locally, start PostgreSQL and Redis with Docker Compose (see [Docker](../delivery/docker.md#local-development)). In CI, they run as service containers (see [CI/CD](../delivery/ci-cd.md)).

## Related

- [Testing strategy](testing-strategy.md)
- [Playwright](playwright.md)
- [FastAPI architecture](../backend/fastapi-architecture.md)
