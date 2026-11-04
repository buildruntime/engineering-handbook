[Engineering handbook](../../README.md) / Workflow

# Code review

Every change is reviewed by someone other than its author before it is merged. Review catches defects, but its larger value is shared understanding: after a review, two people know how the code works and why. This page is for reviewers. [Pull requests](pull-requests.md) is for authors.

## What to look at

In roughly this order, because a problem higher in the list makes the ones below it irrelevant:

1. **The problem.** Is this the right change to make? Is it solving the problem in the issue?
2. **Design.** Does it fit the architecture (see [Clean architecture](../principles/clean-architecture.md))? Is it as simple as the problem allows? Will it be easy to change later?
3. **Correctness.** Edge cases, error handling, concurrency, time zones, empty and very large inputs. What happens when a dependency fails?
4. **Tests.** Would the tests fail if the change were broken? Are the important cases covered, including the failure paths?
5. **Security.** Input validation, authorisation checks, secrets, injection, data exposure (see [Security](../quality/security.md)).
6. **Operations.** Migrations that are safe to run on a live database, logging and metrics for the new path, configuration, and a way to roll back.
7. **Readability.** Names, structure, comments where the code cannot speak for itself.
8. **Documentation.** README, API description, changelog.

Formatting and style are not on the list. Tools enforce them.

## How to review

- **Read the description first**, then the tests, then the code. The tests show what the author thinks the code should do.
- **Check out the branch** for anything non-trivial, and run it. Some problems only show when the code runs.
- **Review in one pass** where you can, so that the author gets all the comments together rather than a new set after each round.
- **If the change is too large to review well, say so.** Ask for it to be split rather than approving what you could not read.
- **Respond as your availability allows**, and tell the author if a review will take a while, so that they can plan or ask someone else.

## Writing comments

Comment on the code, not the person. "This loop runs a query per project" is a comment about code; "you forgot about N+1 queries" is about the author.

Make clear how much each comment matters, so that the author knows what must change before merging. Start comments with a label:

| Label | Meaning |
| :-- | :-- |
| `blocking:` | Must be resolved before merging. Explain why. |
| `suggestion:` | An improvement worth considering. The author decides. |
| `question:` | You do not understand something. Not a disguised request. |
| `nit:` | Minor and optional. If you have many, a tool should probably catch them. |
| `praise:` | Something done well. Worth saying. |

Explain the reason behind a request, and propose an alternative where you can. GitHub's suggestion feature is useful for small, concrete changes.

```text
blocking: this loop calls get_members(project.id) once per project, so a page
of 100 projects makes 101 queries. Could the repository load the members for
the whole page in one query, with selectinload?
```

```text
question: why does the cache key include the user ID? Projects look the same for
every user, so I expected one entry per project.
```

## Approving

Approve when the change is correct and makes the codebase better, even if it is not how you would have written it. The aim is not a perfect pull request, but continuous improvement.

- **Approve with comments** when only suggestions and nits remain, and trust the author to handle them.
- **Request changes** only for blocking comments, and say what would resolve them.
- **Re-review promptly** when the author responds. A pull request waiting on a re-review is the most expensive kind of waiting, because the author has already moved on.

## For authors

- Assume good intent. A terse comment is usually a busy reviewer, not an annoyed one.
- A question from a reviewer often means the code needs a comment or a better name, not just an answer in the pull request.
- If a discussion goes back and forth more than twice, talk directly, then summarise the outcome in the pull request.

## Related

- [Pull requests](pull-requests.md)
- [Engineering principles](../principles/engineering-principles.md)
- [Security](../quality/security.md)
