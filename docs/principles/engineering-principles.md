[Engineering handbook](../../README.md) / Principles

# Engineering principles

These principles explain the defaults in the rest of the handbook. When a page does not cover a situation, start here.

## Prefer the simple design

Choose the design with the fewest moving parts that meets today's requirements and does not block the likely next ones. A second service, a message queue or a cache is a cost that every future change pays. Add one when a measured need appears, not in anticipation of one.

Simple is not the same as easy. A clear module boundary takes more thought than putting everything in one file, and it is usually the simpler result.

## Write for the next reader

Code is read far more often than it is written. Name things after what they mean in the domain, keep functions short enough to understand without scrolling, and leave a comment where the code cannot explain itself: why a workaround exists, which constraint a number comes from, what a regular expression matches.

```python
# Bad: the reader has to reverse-engineer the intent.
if (now - user.created_at).days < 14 and not user.plan:
    ...

# Good: the rule has a name and the constant has a reason.
TRIAL_LENGTH = timedelta(days=14)  # Matches the trial period on the pricing page.

def is_in_trial(user: User, now: datetime) -> bool:
    return user.plan is None and now - user.created_at < TRIAL_LENGTH
```

## Make it explicit

Hidden behaviour is the most common source of production surprises. Prefer:

- explicit configuration over defaults that differ between environments;
- explicit dependencies, passed in, over global state;
- explicit types and schemas at every boundary: HTTP, database, queue, environment variables;
- explicit failure over silent fallbacks. A missing configuration value stops the process at start-up instead of producing a wrong result later.

## Own it in production

Work is finished when it runs correctly in production and someone can tell that it does. Every change considers:

- **Observability.** Can you see from logs and metrics whether the new code path works?
- **Failure.** What happens when a dependency is slow or unavailable?
- **Rollback.** Can the change be reverted without data loss? Database migrations are the usual reason it cannot. See [Zero-downtime schema changes](../data/postgresql.md#zero-downtime-schema-changes).

## Small, reversible changes

Small changes are reviewed properly, deployed with less risk, and reverted cleanly. Split work so that each pull request is useful on its own, and hide unfinished features behind a flag rather than keeping a long-lived branch.

Treat decisions the same way. Most decisions can be reversed cheaply, so make them quickly. Spend time on the few that cannot: public API contracts, data models, choice of database, and anything that stores user data.

## Automate the checks

If a rule matters, a tool should enforce it. Formatting, linting, type checking, tests and dependency audits run in CI on every pull request, so that review time goes to design and correctness. A rule that cannot be automated belongs in the pull request template or in this handbook.

## Recording decisions

Significant technical decisions are recorded as architecture decision records (ADRs) in the repository they affect, under `docs/decisions/`. A decision is significant if it is expensive to reverse, or if someone will later ask why things are the way they are.

An ADR is short. Number them in sequence and never rewrite an accepted one: if a decision changes, write a new record that supersedes it.

```markdown
# 0007. Use PostgreSQL advisory locks for scheduled jobs

Date: 2026-03-12
Status: Accepted

## Context

Scheduled jobs run in every application instance. Some of them must not run
concurrently, and we do not want to add a separate scheduler service.

## Decision

Each job takes a transaction-level advisory lock named after the job before it
starts, and exits if the lock is already held.

## Consequences

- No new infrastructure. Jobs keep running if one instance fails.
- Jobs are tied to PostgreSQL. Moving them to another store means revisiting this.
- A job that holds its transaction open for a long time holds the lock too.
```

## Related

- [Clean architecture](clean-architecture.md)
- [Code review](../workflow/code-review.md)
