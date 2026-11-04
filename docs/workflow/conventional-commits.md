[Engineering handbook](../../README.md) / Workflow

# Conventional Commits

Commit messages in Buildruntime repositories follow the [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) specification. The format makes the history readable, and lets tools work out the next version number and write the changelog (see [Versioning and releases](../delivery/versioning-and-releases.md)).

## Format

```text
<type>[optional scope][!]: <description>

[optional body]

[optional footer(s)]
```

- **type**: what kind of change it is, from the table below.
- **scope**: optional, the part of the codebase in parentheses, such as `feat(api):` or `fix(projects):`. Use the scopes the repository has agreed on; do not invent a new one for each commit.
- **`!`**: marks a breaking change.
- **description**: a short summary in the imperative mood, lowercase, with no full stop: `add archive endpoint`, not `Added archive endpoint.` Keep the whole first line to 72 characters or fewer.
- **body**: optional. Explain what changed and why, wrapped at 72 characters. The diff shows how.
- **footers**: optional. `BREAKING CHANGE: <description>`, issue references such as `Refs: #142`, and trailers such as `Co-authored-by:`.

## Types

| Type | Use for | Affects the version |
| :-- | :-- | :-- |
| `feat` | A new feature for users of the project | Minor |
| `fix` | A bug fix for users of the project | Patch |
| `perf` | A change that improves performance without changing behaviour | Patch |
| `refactor` | A change to code structure that neither fixes a bug nor adds a feature | No |
| `docs` | Documentation only | No |
| `test` | Adding or correcting tests | No |
| `build` | The build system or dependencies | No |
| `ci` | CI configuration and scripts | No |
| `chore` | Maintenance that does not fit elsewhere, such as releases | No |
| `revert` | Reverting an earlier commit | Depends on what is reverted |

Any type with `!`, or a `BREAKING CHANGE:` footer, means a major version.

"Users" depends on the project. For a library, a change to its public API is a `feat` or a `fix`. For a service, a change to its HTTP API or its behaviour is. An internal change that users cannot observe is a `refactor`, however large.

## Examples

A fix, with a body that explains the cause:

```text
fix(api): decode cursors without base64 padding

Cursors are encoded without trailing "=" so that they are safe in URLs,
but decoding expected the padding and rejected every cursor longer than
one page. Restore the padding before decoding.

Refs: #142
```

A feature:

```text
feat(projects): archive projects

Add POST /v1/projects/{slug}/archive. Archived projects keep their data
and remain readable, but are left out of the default listing.
```

A breaking change, with `!` and a footer that says what users must do:

```text
feat(api)!: require cursor pagination for project lists

BREAKING CHANGE: GET /v1/projects no longer accepts the page parameter.
Use the next_cursor value from the previous response instead.
```

Smaller changes need only the first line:

```text
docs: explain the readiness endpoint
```

```text
build(deps): update sqlalchemy to 2.1.8
```

```text
test(projects): cover archiving an archived project
```

Not like this:

```text
Fixed stuff
```

```text
feat: Added the archive endpoint and fixed the cursor bug and updated the README.
```

The first has no type and says nothing. The second describes three changes in one commit, in the past tense, and is too long.

## Pull request titles

Pull requests are squash-merged, so the title becomes the commit on `main` (see [Merging](git-workflow.md#merging)). Titles follow the same format as commit messages, and CI checks them. The commits inside a pull request should still be well written, for the reviewer's sake.

## Checking messages

Use [commitlint](https://commitlint.js.org/) with the conventional configuration to check commit messages and pull request titles:

```js
// commitlint.config.mjs
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "header-max-length": [2, "always", 72],
  },
};
```

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
echo "feat(projects): archive projects" | npx commitlint     # Passes
echo "Fixed stuff" | npx commitlint                          # Fails
```

In a Python project without Node.js, run it in CI only, or use a Python implementation of the same rules.

## Related

- [Git workflow](git-workflow.md)
- [Pull requests](pull-requests.md)
- [Versioning and releases](../delivery/versioning-and-releases.md)
