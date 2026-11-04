[Engineering handbook](../../README.md) / Delivery

# Docker

Every Buildruntime service is built into a container image, and the same image runs in CI, in staging and in production. Configuration comes from environment variables at run time, never from the image. This page shows images for a FastAPI service and a Next.js application, and a Compose file for local development.

## Image rules

- **Multi-stage builds.** Build tools and caches stay in the build stage. The final image contains only what runs.
- **Small, official base images**, pinned to a version: `python:3.13-slim`, `node:24-alpine`. Rebuild regularly so that base image security fixes are picked up.
- **Dependencies before source.** Copy the lock file and install dependencies in a layer of their own, so that it is cached until the dependencies change.
- **Never run as root.** The final stage switches to an unprivileged user.
- **No secrets in images.** No `.env` files, keys or tokens in any layer, including build stages: a secret removed in a later layer is still in the earlier one. Use build secrets (`RUN --mount=type=secret`) if the build needs credentials.
- **One process per container.** Migrations, the web server and workers are separate containers started from the same image with different commands.
- **Log to standard output.** The platform collects it.

## Python services

The image uses [uv](https://docs.astral.sh/uv/) to install the locked dependencies into a virtual environment, and copies that environment into a clean runtime stage.

```dockerfile
# Dockerfile
FROM python:3.13-slim AS build
COPY --from=ghcr.io/astral-sh/uv:0.13 /uv /bin/uv
ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never
WORKDIR /app

# Dependencies only. This layer is reused until pyproject.toml or uv.lock changes.
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-install-project

COPY alembic.ini ./
COPY migrations ./migrations
COPY app ./app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

FROM python:3.13-slim AS runtime
RUN groupadd --system app && useradd --system --gid app --uid 10001 --no-create-home app
WORKDIR /app
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1
COPY --from=build --chown=app:app /app /app
USER app
EXPOSE 8000
CMD ["uvicorn", "app.main:create_app", "--factory", "--host", "0.0.0.0", "--port", "8000"]
```

Exclude everything the image does not need. Besides keeping the image small, this keeps local secrets out of the build context:

```gitignore
# .dockerignore
.git
.venv
.env
.env.*
!.env.example
**/__pycache__
.pytest_cache
.mypy_cache
.ruff_cache
tests
```

Run migrations from the same image, as a separate step before the new version starts:

```bash
docker build -t projects-api:dev .
docker run --rm --env-file .env projects-api:dev alembic upgrade head
docker run --rm --env-file .env -p 8000:8000 projects-api:dev
```

## Next.js applications

With `output: "standalone"` in `next.config.ts` (see [Next.js](../frontend/nextjs.md#configuration)), the build produces a minimal server and only the dependencies it uses. The runtime stage copies that, plus the static assets, which the standalone output does not include.

```dockerfile
# Dockerfile
FROM node:24-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM node:24-alpine AS build
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:24-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    HOSTNAME=0.0.0.0 \
    PORT=3000
COPY --from=build --chown=node:node /app/public ./public
COPY --from=build --chown=node:node /app/.next/standalone ./
COPY --from=build --chown=node:node /app/.next/static ./.next/static
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Variables prefixed `NEXT_PUBLIC_` are fixed at build time. If the same image must run in several environments, keep environment-specific values out of `NEXT_PUBLIC_` variables, and read them on the server instead.

## Local development

A Compose file in each service repository starts the service and its dependencies with one command. It is for development only. Production does not use it.

```yaml
# compose.yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: projects
      POSTGRES_PASSWORD: projects
      POSTGRES_DB: projects
    ports:
      - "5432:5432"
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U projects -d projects"]
      interval: 2s
      timeout: 5s
      retries: 15

  redis:
    image: redis:7
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 2s
      timeout: 5s
      retries: 15

  migrate:
    build: .
    command: ["alembic", "upgrade", "head"]
    environment: &api-environment
      APP_ENVIRONMENT: development
      APP_DATABASE_URL: postgresql+asyncpg://projects:projects@db:5432/projects
      APP_REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy

  api:
    build: .
    environment: *api-environment
    ports:
      - "8000:8000"
    depends_on:
      migrate:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/live')"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  db-data:
```

```bash
docker compose up --build --wait     # Start everything and wait until it is healthy
docker compose up -d db redis        # Only the dependencies, to run the API on the host
docker compose logs -f api
docker compose down                  # Stop. Add --volumes to delete the database too.
```

The `migrate` service runs once, and `api` starts only after it has succeeded. The credentials here are for a local database. Production credentials never appear in a Compose file.

## Tagging and publishing

- CI builds the image once per commit on `main`, and tags it with the full commit SHA. That tag is immutable and identifies exactly what is running.
- Releases add the version tag, such as `1.4.0` (see [Versioning and releases](versioning-and-releases.md)).
- Deployments refer to an immutable tag or a digest, never to `latest`.
- Images are scanned for known vulnerabilities in CI (see [Security](../quality/security.md#dependencies)).

## Related

- [CI/CD](ci-cd.md)
- [Versioning and releases](versioning-and-releases.md)
- [FastAPI architecture](../backend/fastapi-architecture.md)
