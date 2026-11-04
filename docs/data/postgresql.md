[Engineering handbook](../../README.md) / Data

# PostgreSQL

How to get predictable performance and safe schema changes out of PostgreSQL. The schema examples are those from [Database design](database-design.md). Query plans on this page are real output from PostgreSQL 16 on a table of 200,000 projects, with planning-time and per-worker lines left out. The timings depend on the machine; the row counts and plan shapes are the point.

## Reading a query plan

Before adding an index or rewriting a query, look at what PostgreSQL actually does. `EXPLAIN` shows the plan; `EXPLAIN ANALYZE` runs the query and shows what happened.

To reproduce the examples, create a test table:

```sql
CREATE TABLE projects (
    id          uuid         NOT NULL,
    slug        varchar(63)  NOT NULL,
    name        varchar(200) NOT NULL,
    created_at  timestamptz  NOT NULL DEFAULT now(),
    archived_at timestamptz,
    CONSTRAINT pk_projects PRIMARY KEY (id),
    CONSTRAINT uq_projects_slug UNIQUE (slug)
);

INSERT INTO projects (id, slug, name, created_at, archived_at)
SELECT gen_random_uuid(), 'project-' || i, 'Project ' || i,
       now() - make_interval(mins => i),
       CASE WHEN i % 10 = 0 THEN now() END
FROM generate_series(1, 200000) AS i;

ANALYZE projects;
```

The newest projects, with no suitable index:

```sql
EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF)
SELECT id, slug, name FROM projects ORDER BY created_at DESC, id DESC LIMIT 20;
```

```text
 Limit (actual rows=20 loops=1)
   ->  Gather Merge (actual rows=20 loops=1)
         Workers Planned: 1
         Workers Launched: 1
         ->  Sort (actual rows=20 loops=2)
               Sort Key: created_at DESC, id DESC
               Sort Method: top-N heapsort  Memory: 26kB
               ->  Parallel Seq Scan on projects (actual rows=100000 loops=2)
 Execution Time: 32.879 ms
```

PostgreSQL read all 200,000 rows (100,000 in each of two processes) and sorted them to return 20. With an index that matches the sort order:

```sql
CREATE INDEX ix_projects_created_at_id ON projects (created_at, id);

EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF)
SELECT id, slug, name FROM projects ORDER BY created_at DESC, id DESC LIMIT 20;
```

```text
 Limit (actual rows=20 loops=1)
   ->  Index Scan Backward using ix_projects_created_at_id on projects (actual rows=20 loops=1)
 Execution Time: 0.056 ms
```

What to look for in a plan:

- **`Seq Scan` on a large table** in a query that returns few rows usually means a missing index.
- **`actual rows` far from what you expected** shows where the work goes. A step that reads 100,000 rows to return 20 is the one to fix.
- **Estimated rows very different from actual rows** (shown without `COSTS OFF`) means the statistics are out of date, or the planner cannot estimate the condition. Run `ANALYZE`, and check that the query does not hide columns inside functions.
- **`Sort` with `external merge`** means the sort spilled to disk.

## Indexes

An index speeds up reads and slows down every write to the table. Add one for a query that needs it, and check with `EXPLAIN` that it is used.

**B-tree**, the default, serves equality, ranges and sorting. In a multi-column index, put equality columns first, then the range or sort column:

```sql
CREATE TABLE tasks (
    id         bigint GENERATED ALWAYS AS IDENTITY,
    project_id uuid        NOT NULL,
    status     text        NOT NULL,
    due_on     date,
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_tasks PRIMARY KEY (id)
);

-- Serves: WHERE project_id = $1 AND status = 'open' ORDER BY due_on
CREATE INDEX ix_tasks_project_id_status_due_on ON tasks (project_id, status, due_on);
```

**Partial indexes** cover only the rows a query asks for, and are smaller and faster as a result:

```sql
-- Most queries list active projects only.
CREATE INDEX ix_projects_active_created_at
    ON projects (created_at DESC)
    WHERE archived_at IS NULL;
```

**GIN** indexes serve `jsonb` containment and full-text search:

```sql
ALTER TABLE tasks ADD COLUMN labels jsonb NOT NULL DEFAULT '[]';
CREATE INDEX ix_tasks_labels ON tasks USING gin (labels jsonb_path_ops);

-- Uses the index:
SELECT id FROM tasks WHERE labels @> '["urgent"]';
```

An index is not used when the query wraps the column in a function or cast, as in `WHERE lower(slug) = $1` or `WHERE created_at::date = $1`. Rewrite the condition as a range on the column, or create an expression index on exactly that expression.

## Keyset pagination

`OFFSET` reads and discards every row before the page. The deeper the page, the slower it gets:

```sql
EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF)
SELECT id, slug, name FROM projects
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET 150000;
```

```text
 Limit (actual rows=20 loops=1)
   ->  Index Scan Backward using ix_projects_created_at_id on projects (actual rows=150020 loops=1)
 Execution Time: 23.199 ms
```

Keyset pagination continues from the last row of the previous page instead. A row comparison on the indexed columns reads only the rows it returns, however deep the page:

```sql
EXPLAIN (ANALYZE, COSTS OFF, TIMING OFF)
SELECT id, slug, name FROM projects
WHERE (created_at, id) < (now() - interval '50000 minutes', 'ffffffff-ffff-ffff-ffff-ffffffffffff')
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

```text
 Limit (actual rows=20 loops=1)
   ->  Index Scan Backward using ix_projects_created_at_id on projects (actual rows=20 loops=1)
         Index Cond: (ROW(created_at, id) < ROW((now() - '833:20:00'::interval), 'ffffffff-ffff-ffff-ffff-ffffffffffff'::uuid))
 Execution Time: 0.205 ms
```

The `id` column makes the order unique, so that rows with the same `created_at` are neither skipped nor repeated. The API exposes this position as an opaque cursor (see [Pagination](../backend/api-design.md#pagination)).

## Transactions and concurrency

PostgreSQL's default isolation level is `READ COMMITTED`: each statement sees the data committed before it started. Two transactions that read a value, compute in the application, and write it back can lose an update. Avoid that in one of three ways, in order of preference.

Let the database do the arithmetic in a single statement:

```sql
CREATE TABLE accounts (
    id      bigint GENERATED ALWAYS AS IDENTITY,
    balance bigint NOT NULL,
    CONSTRAINT pk_accounts PRIMARY KEY (id),
    CONSTRAINT ck_accounts_balance_not_negative CHECK (balance >= 0)
);
INSERT INTO accounts (balance) VALUES (5000);

-- Withdraw 1200 only if the balance allows it. Zero rows updated means it did not.
UPDATE accounts SET balance = balance - 1200
WHERE id = 1 AND balance >= 1200
RETURNING balance;
```

Lock the row for the rest of the transaction, when the decision needs more than one statement:

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
-- ... decide in the application ...
UPDATE accounts SET balance = balance - 300 WHERE id = 1;
COMMIT;
```

Or use optimistic concurrency with a version column, which suits user edits that take minutes (see [Concurrency control](../backend/api-design.md#concurrency-control)).

Keep transactions short. A transaction that stays open while the application calls another service holds its locks, and its connection, for as long as that call takes.

### Work queues

`FOR UPDATE SKIP LOCKED` lets several workers take jobs from one table without blocking each other or taking the same job twice:

```sql
CREATE TABLE jobs (
    id         bigint GENERATED ALWAYS AS IDENTITY,
    kind       text        NOT NULL,
    payload    jsonb       NOT NULL,
    run_after  timestamptz NOT NULL DEFAULT now(),
    attempts   integer     NOT NULL DEFAULT 0,
    locked_at  timestamptz,
    CONSTRAINT pk_jobs PRIMARY KEY (id)
);
CREATE INDEX ix_jobs_run_after ON jobs (run_after) WHERE locked_at IS NULL;

INSERT INTO jobs (kind, payload) VALUES ('send_invite', '{"member_id": 42}');

-- Each worker claims up to ten jobs that no other worker holds.
WITH claimed AS (
    SELECT id FROM jobs
    WHERE locked_at IS NULL AND run_after <= now()
    ORDER BY run_after
    LIMIT 10
    FOR UPDATE SKIP LOCKED
)
UPDATE jobs SET locked_at = now(), attempts = attempts + 1
FROM claimed WHERE jobs.id = claimed.id
RETURNING jobs.id, jobs.kind, jobs.payload;
```

The worker deletes a job when it succeeds. On failure, it clears `locked_at` and moves `run_after` forward. A periodic task releases jobs whose `locked_at` is older than the longest a job may take, in case a worker died. This is enough for most services; move to a dedicated queue when throughput or features require it, not before.

## Timeouts

Set limits so that one bad query cannot take a service down:

```sql
CREATE ROLE projects_app LOGIN;

-- Applies to every session of the application's role.
ALTER ROLE projects_app SET statement_timeout = '15s';
ALTER ROLE projects_app SET idle_in_transaction_session_timeout = '60s';
```

`statement_timeout` stops runaway queries. `idle_in_transaction_session_timeout` ends sessions that opened a transaction and then stopped doing anything, which otherwise hold locks indefinitely. Jobs and migrations that legitimately run longer set their own value with `SET LOCAL statement_timeout` inside their transaction.

## Connection pooling

Each PostgreSQL connection is a process on the server, and the server's `max_connections` is shared by every instance of every service that connects to it.

- Size each application's pool so that `instances × (pool_size + max_overflow)` stays well under `max_connections`. SQLAlchemy's default is a pool of 5 with an overflow of 10.
- Use `pool_pre_ping=True`, so that connections closed by the server or a load balancer are replaced instead of failing a request.
- When there are too many instances for direct connections, put PgBouncer in transaction mode in front of the database. With asyncpg, prepared statements need extra configuration in that mode. Check the driver's documentation before switching.

## Zero-downtime schema changes

During a deployment, the old and new versions of the application run at the same time, against the same schema. Every migration must work with both, and must not hold a lock that blocks normal queries for longer than a moment.

Every migration starts by limiting how long it waits for a lock. A migration that waits behind a long-running query also blocks every query queued behind it:

```sql
SET lock_timeout = '5s';
```

If the timeout is reached, the migration fails and can be retried. That is far better than an outage.

| Change | Safe approach |
| :-- | :-- |
| Add a column | Add it as nullable, or with a constant default. Both are instant. |
| Add `NOT NULL` | Add a `CHECK (column IS NOT NULL) NOT VALID` constraint, validate it, then set `NOT NULL`. PostgreSQL uses the valid constraint and skips the full table scan. |
| Add an index | `CREATE INDEX CONCURRENTLY`, outside a transaction. |
| Add a foreign key | Add it `NOT VALID`, then `VALIDATE CONSTRAINT` in a separate step. |
| Rename a column or table | Do not. Add the new one, write to both, backfill, switch reads, then drop the old one in a later release. |
| Change a column's type | As for a rename. A type change rewrites the table under an exclusive lock. |
| Drop a column | First release code that no longer uses it, then drop it in a later migration. |

For example, making `tasks.due_on` required without blocking writes:

```sql
-- Step 1: instant. New rows are checked; existing rows are not yet.
ALTER TABLE tasks
    ADD CONSTRAINT ck_tasks_due_on_not_null CHECK (due_on IS NOT NULL) NOT VALID;

-- Step 2: backfill in batches, then validate. Validation scans the table
-- but allows reads and writes while it runs.
UPDATE tasks SET due_on = created_at::date WHERE due_on IS NULL;
ALTER TABLE tasks VALIDATE CONSTRAINT ck_tasks_due_on_not_null;

-- Step 3: instant, because the valid constraint proves there are no nulls.
ALTER TABLE tasks ALTER COLUMN due_on SET NOT NULL;
ALTER TABLE tasks DROP CONSTRAINT ck_tasks_due_on_not_null;
```

And adding an index to a table that is in use:

```sql
CREATE INDEX CONCURRENTLY ix_tasks_created_at ON tasks (created_at);
```

If a concurrent index build fails, it leaves an invalid index behind. Drop it and run the build again.

In Alembic, `CREATE INDEX CONCURRENTLY` must run outside the migration's transaction:

```python
def upgrade() -> None:
    with op.get_context().autocommit_block():
        op.create_index(
            "ix_tasks_created_at", "tasks", ["created_at"], postgresql_concurrently=True
        )
```

## Related

- [Database design](database-design.md)
- [Redis](redis.md)
- [Performance](../quality/performance.md#backend)
