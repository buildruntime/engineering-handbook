[Engineering handbook](../../README.md) / Quality

# Performance

Performance work starts with a measurement and ends with one. Guessing where time goes is usually wrong, and an optimisation that was not measured may have made things slower, or more complex for no gain. This page covers how to measure, and the causes of slowness that come up most often in Buildruntime's stack.

## Measure first

1. **Define what "fast enough" means**, as a number: "`GET /v1/projects` responds in under 200 ms at the 95th percentile", "LCP under 2.5 s at the 75th percentile". Averages hide the slow requests that users notice.
2. **Measure where the time goes**, with the right tool for the layer:

   | Layer | Tool |
   | :-- | :-- |
   | A slow endpoint, in production | Request duration metrics and traces, broken down by route |
   | Python code | `py-spy record` against a running process, or `cProfile` for a script |
   | A slow query | `EXPLAIN (ANALYZE, BUFFERS)` (see [Reading a query plan](../data/postgresql.md#reading-a-query-plan)), and `pg_stat_statements` for the queries that take the most time in total |
   | Frontend loading | Lighthouse and the Performance panel in browser developer tools, and real-user Core Web Vitals |
   | Bundle size | `next build` output, and `next analyze` for an interactive view (Turbopack builds) |
   | Behaviour under load | A load test with a tool such as k6 or Locust, against a production-like environment |

3. **Fix the largest cost**, then measure again.

## Backend

### N+1 queries

The most common cause of a slow endpoint is one query per item in a list. It is invisible with test data of three rows, and dominant with real data of three hundred.

```python
from uuid import UUID

from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload


# Slow: one query for the projects, then one more for each project's members.
async def members_by_project_slow(session: AsyncSession) -> dict[UUID, list[MemberRow]]:
    members: dict[UUID, list[MemberRow]] = {}
    for project in await session.scalars(select(ProjectRow).limit(100)):
        query = select(MemberRow).where(MemberRow.project_id == project.id)
        members[project.id] = list(await session.scalars(query))
    return members


# Two queries in total, whatever the page size.
async def projects_with_members(session: AsyncSession) -> list[ProjectRow]:
    query = select(ProjectRow).options(selectinload(ProjectRow.members)).limit(100)
    return list(await session.scalars(query))
```

The example assumes a `members` relationship on `ProjectRow`. With async SQLAlchemy, a relationship that was not loaded up front is not loaded implicitly when the attribute is accessed: it raises an error instead. That makes N+1 queries easier to spot, but only if relationships are loaded explicitly with `selectinload` or `joinedload`.

To find them, log the number of queries per request in development, or watch for the same statement repeated in a trace. In tests of list endpoints, it can be worth asserting the number of queries.

### Queries and the database

- **Every list is paginated**, with keyset pagination and a maximum page size (see [Pagination](../backend/api-design.md#pagination)).
- **Select only the columns you need** for large tables and wide rows, especially `jsonb` and `text` columns that are not displayed.
- **Index for the queries you run**, and check with `EXPLAIN` that the index is used (see [Indexes](../data/postgresql.md#indexes)).
- **Do the work in the database** when it is set-based: an `UPDATE ... WHERE` over many rows is one statement, not a loop of single-row updates.
- **Watch the connection pool.** Requests that wait for a connection are slow even when every query is fast. Size the pool for the expected concurrency, and keep transactions short (see [Connection pooling](../data/postgresql.md#connection-pooling)).

### Application code

- **Do independent I/O concurrently.** Two calls to other services that do not depend on each other can run together with `asyncio.TaskGroup`.
- **Never block the event loop** (see [Async code](../backend/fastapi-architecture.md#async-code)). One blocking call slows down every request on that worker.
- **Move slow work out of the request.** Sending email, generating reports and calling slow third parties belong in a background job; the request returns `202 Accepted` or completes without waiting.
- **Cache what you have measured to be expensive and frequently read**, with a TTL that matches how stale it may be (see [Caching](../data/redis.md#caching)).
- **Compress responses.** Enable gzip for JSON responses above a small size, at the load balancer or with Starlette's `GZipMiddleware`.

## Frontend

The targets are the Core Web Vitals in [Frontend standards](../frontend/frontend-standards.md#performance).

- **Render on the server.** Server Components send HTML and no JavaScript. Keep `"use client"` at the leaves of the component tree (see [Server and Client Components](../frontend/nextjs.md#server-and-client-components)).
- **Send less JavaScript.** Check what each new dependency adds to the bundle before adding it. Load heavy components that are not needed immediately, such as a chart or a rich text editor, with `next/dynamic`.
- **Images** use `next/image`, which serves modern formats at the right size and lazy-loads images below the fold. Give every image its dimensions, so that the layout does not shift when it loads. Mark the image that is the Largest Contentful Paint with `priority`.
- **Fonts** use `next/font`, which hosts the font files with the application and avoids layout shift while they load.
- **Reserve space** for anything that loads after the first paint, such as images, embeds and banners, so that CLS stays low.
- **Keep interactions responsive.** Long tasks on the main thread hurt INP. Break up expensive work, and use `useTransition` for updates that do not need to happen immediately.
- **Fetch in parallel on the server** with `Promise.all`, rather than one request after another (see [Fetching data](../frontend/nextjs.md#fetching-data)).

## Load testing

Run a load test before launching something that will see significant traffic, and after changes to how a hot path works:

- test against an environment with production-like data volumes, never against production without agreement;
- ramp up gradually, and watch latency percentiles, error rates and database load, not only throughput;
- find the point where latency starts to climb, and know which resource runs out first: CPU, database connections, or a dependency.

## Related

- [PostgreSQL](../data/postgresql.md)
- [Redis](../data/redis.md)
- [Frontend standards](../frontend/frontend-standards.md)
