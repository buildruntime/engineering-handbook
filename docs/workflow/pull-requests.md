[Engineering handbook](../../README.md) / Workflow

# Pull requests

Every change to a Buildruntime repository arrives through a pull request. A good pull request is easy to review: it does one thing, says why, and shows that it works. This page is for authors. [Code review](code-review.md) is for reviewers.

## Size

Small pull requests get better reviews, merge sooner and are easier to revert. As a guide, aim for a diff a reviewer can read carefully in one sitting, a few hundred changed lines at most, not counting generated files and lock files.

To keep pull requests small:

- **Separate refactoring from behaviour changes.** First a pull request that restructures the code without changing what it does, then one that changes the behaviour. Each is easy to review; together they are not.
- **Split by layer when it helps.** A migration that adds a nullable column, then the code that writes to it, then the code that reads from it.
- **Merge unfinished features behind a flag** instead of keeping a long-lived branch.
- **Keep mechanical changes apart.** A rename across 80 files is a pull request of its own, and its description says it is mechanical.

## Before opening it

- [ ] The branch is up to date with `main` (see [Keeping a branch up to date](git-workflow.md#keeping-a-branch-up-to-date)).
- [ ] Formatting, linting, type checks and tests pass locally.
- [ ] You have read your own diff from start to finish, as a reviewer would. Most review comments are about things the author would have caught this way.
- [ ] Debug output, commented-out code and stray files are gone.
- [ ] Documentation, the changelog and the API description cover the change.

## Title and description

The title is a [Conventional Commit](conventional-commits.md) header, because it becomes the commit message on `main`: `feat(projects): archive projects`.

The description follows the repository's pull request template. Buildruntime repositories inherit [the default template](https://github.com/buildruntime/.github/blob/main/PULL_REQUEST_TEMPLATE.md). Whatever the template, the description answers:

- **What problem does this solve?** Link the issue with a closing keyword, such as `Fixes #142`.
- **What approach did you take, and why?** Mention alternatives you rejected, so that the reviewer does not suggest them.
- **How did you test it?** The commands, the cases, and anything checked by hand. For interface changes, before and after screenshots.
- **What should the reviewer look at closely?** The tricky part, the decision you are unsure about, the migration.
- **What is the impact on deployment?** Migrations, new configuration, feature flags, or anything that must happen in a particular order.

## Drafts

Open a draft pull request to get early feedback on an approach, or to run CI before the work is finished. Say what kind of feedback you want: "Is this the right place for the cache?" is useful, a draft with no description is not. Mark it ready for review when it meets the checklist above.

## Checks

A pull request can be merged only when every required check passes. If a check fails:

- **Fix the cause.** Do not disable, skip or delete a test to get a green build. If a test is wrong, fix the test, and say so in the description.
- **Do not re-run a failing check until it passes.** A test that fails intermittently is a bug (see [Flaky tests](../quality/testing-strategy.md#flaky-tests)).
- **If a failure is unrelated to your change**, say so in the pull request, and open an issue for it.

## Responding to review

- Reply to every comment, even if only to say it is done, and push changes as new commits so that the reviewer can see what changed.
- If you disagree, explain why. If you agree but want to do it later, open an issue and link it.
- Let the reviewer resolve their own comments, unless the fix is trivial.
- Re-request review when you have addressed everything.

## Merging

Merge with squash and merge once the pull request is approved and the checks pass. Check the commit message GitHub proposes: it is built from the title and description, and it becomes the permanent record on `main`.

After merging, watch the deployment if the change goes out automatically, and check that it behaves as expected.

## Related

- [Code review](code-review.md)
- [Git workflow](git-workflow.md)
- [Conventional Commits](conventional-commits.md)
