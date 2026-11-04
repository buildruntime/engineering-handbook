[Engineering handbook](../../README.md) / Delivery

# CI/CD

Every change reaches production through the same automated pipeline. A pull request runs every check; a merge to `main` builds the image once and promotes that image through each environment. The examples use GitHub Actions.

## Pipeline

```mermaid
flowchart LR
    PR["Pull request"] --> C["Checks<br/>format, lint, types,<br/>migrations, tests"]
    C -->|"all green, approved"| M["Merge to main"]
    M --> C2["Checks again,<br/>on the merged code"]
    C2 --> B["Build image<br/>tag: commit SHA"]
    B --> S["Deploy to staging<br/>migrate, then roll out"]
    S --> V["Smoke tests"]
    V --> P["Deploy to production<br/>same image"]
```

- **The checks on a pull request are the same as on `main`.** A check that only runs after merge finds problems when they are already on everyone's branch.
- **Build once.** The image deployed to production is the one that was tested in staging, identified by its commit SHA. It is not rebuilt for each environment.
- **`main` is always deployable.** If the pipeline on `main` fails, fixing it comes before merging anything else.

## Checks

A pull request cannot be merged until every required check passes. Configure them as required status checks in the repository's branch protection rules for `main`.

| Check | Python service | Next.js application |
| :-- | :-- | :-- |
| Formatting | `ruff format --check` | `prettier --check .` |
| Linting | `ruff check` | `eslint .` |
| Types | `mypy` | `tsc --noEmit` |
| Migrations | `alembic upgrade head` and `alembic check` | |
| Tests | `pytest`, with PostgreSQL and Redis as service containers | Unit tests, then Playwright against a production build |
| Build | Docker image | `next build`, then the Docker image |
| Dependencies | Dependency review and audit (see [Security](../quality/security.md#dependencies)) | The same |

Keep the pipeline fast. Under ten minutes for a pull request is a good target. Run independent jobs in parallel, cache dependency downloads, and cancel superseded runs.

## Workflow for a Python service

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  check:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: projects_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 2s
          --health-timeout 5s
          --health-retries 15
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 2s
          --health-timeout 5s
          --health-retries 15
    env:
      APP_DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/projects_test
      APP_REDIS_URL: redis://localhost:6379/0
      APP_TEST_DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/projects_test
      APP_TEST_REDIS_URL: redis://localhost:6379/15
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v6
        with:
          enable-cache: true
      - name: Install
        run: uv sync --locked
      - name: Format and lint
        run: |
          uv run ruff format --check .
          uv run ruff check .
      - name: Type-check
        run: uv run mypy app tests
      - name: Migrations
        run: |
          uv run alembic upgrade head
          uv run alembic check
      - name: Tests
        run: uv run pytest

  image:
    needs: check
    if: github.event_name == 'push'
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Points to note:

- **`permissions: contents: read`** at the top limits the token every job gets. The `image` job asks for `packages: write` because it needs it, and nothing else does.
- **`concurrency`** cancels the previous run when a pull request is updated, but never cancels runs on `main`.
- **`timeout-minutes`** stops a hung job from running for the six-hour default.
- **Actions are referenced by major version here for readability.** For third-party actions, pin the full commit SHA instead, with the version in a comment, so that a moved tag cannot change what runs. Dependabot keeps pinned SHAs up to date.

## Workflow for a Next.js application

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  check:
    runs-on: ubuntu-24.04
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - run: npx prettier --check .
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test
      - run: npm run build
      - name: Install browsers
        run: npx playwright install --with-deps chromium
      - name: End-to-end tests
        run: npm run test:e2e
        env:
          API_URL: ${{ vars.E2E_API_URL }}
          API_TOKEN: ${{ secrets.E2E_API_TOKEN }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7
```

The end-to-end tests need a running API. Point them at a test instance, or start the API's published image as a service container in the same job. They must never run against production data.

## Deploying

A deployment of a new version follows the same steps in every environment:

1. **Run migrations** with the new image (`alembic upgrade head`). Migrations are backwards compatible with the version still running (see [Zero-downtime schema changes](../data/postgresql.md#zero-downtime-schema-changes)).
2. **Roll out** the new version gradually, replacing instances one at a time, and only once each new instance reports ready on its readiness endpoint (see [Health checks](../backend/fastapi-architecture.md#health-checks)).
3. **Verify**: smoke tests against the deployed environment, and a look at error rates and latency for the first minutes.

Use GitHub environments for staging and production, with their own secrets. Production may require an approval before the deployment job runs. Configure this in the environment's protection rules.

## Rolling back

The fastest fix for a bad deployment is usually to deploy the previous image again. That works only if the database is still compatible with it, which is another reason migrations are backwards compatible and destructive changes wait for a later release.

- **Roll back first, investigate second.** Restore service, then find the cause.
- **Do not roll back migrations in a hurry.** A down migration that drops a column also drops its data. Fix forward instead, with a new migration.
- **Keep risky features behind flags**, so that turning one off does not need a deployment.

## Secrets

- Secrets live in the platform's secret store or in GitHub encrypted secrets, scoped to the environment that needs them.
- Workflows that deploy to a cloud provider authenticate with OpenID Connect where the provider supports it, rather than a long-lived access key stored as a secret.
- Never print a secret in a log, even partly. GitHub masks registered secrets, but not values derived from them.
- Workflows triggered by `pull_request` from forks do not receive secrets. Do not work around this with `pull_request_target` and a checkout of the fork's code: that runs untrusted code with access to your secrets.

## Related

- [Docker](docker.md)
- [Versioning and releases](versioning-and-releases.md)
- [Pull requests](../workflow/pull-requests.md)
