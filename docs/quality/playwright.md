[Engineering handbook](../../README.md) / Quality

# Playwright

End-to-end tests drive the application in a real browser with [Playwright](https://playwright.dev/), against a production build and a running backend. They cover the journeys that must not break. Everything else is tested at a lower level (see [Testing strategy](testing-strategy.md)).

## Configuration

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

const port = Number(process.env.PORT ?? 3000);
const baseURL = process.env.E2E_BASE_URL ?? `http://localhost:${port}`;

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: Boolean(process.env.CI),
  retries: process.env.CI ? 2 : 0,
  reporter: process.env.CI ? [["github"], ["html", { open: "never" }]] : "list",
  use: {
    baseURL,
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "mobile", use: { ...devices["Pixel 7"] } },
  ],
  webServer: process.env.E2E_BASE_URL
    ? undefined
    : {
        command: "npm run start",
        url: `${baseURL}/api/health`,
        reuseExistingServer: !process.env.CI,
        timeout: 60_000,
      },
});
```

- **Tests run against `next start`**, after `next build`, not against the development server. Production builds behave differently, and are what users get.
- **`webServer`** starts the application and waits for its health check. Set `E2E_BASE_URL` to test an already deployed environment instead.
- **`forbidOnly`** fails the CI run if someone left a `test.only` in the code.
- **Retries in CI** record a trace of the failure (`trace: "on-first-retry"`). A test that passes only on retry is reported as flaky, and must be fixed (see [Flaky tests](testing-strategy.md#flaky-tests)).

## Writing tests

Find elements the way a user does: by role and accessible name, by label, by text. These locators survive changes to markup and styling, and a test that cannot find a button by its name has found an accessibility problem.

| Prefer | Avoid |
| :-- | :-- |
| `page.getByRole("button", { name: "Create project" })` | `page.locator(".btn-primary")` |
| `page.getByLabel("Slug")` | `page.locator("#slug")`, `page.locator("input").nth(1)` |
| `page.getByText("This slug is already in use.")` | XPath |
| `page.getByTestId("project-row")`, when there is no accessible alternative | Selectors tied to component structure |

Use web-first assertions, which wait and retry until the condition holds or the timeout expires. Never wait for a fixed time.

```ts
// e2e/projects.spec.ts
import { expect, type Page, test } from "@playwright/test";

function uniqueSlug(prefix: string): string {
  return `${prefix}-${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 6)}`;
}

async function createProject(page: Page, name: string, slug: string) {
  await page.goto("/projects/new");
  await page.getByLabel("Name").fill(name);
  await page.getByLabel("Slug").fill(slug);
  await page.getByRole("button", { name: "Create project" }).click();
}

test("creating a project opens its page", async ({ page }) => {
  const slug = uniqueSlug("website");

  await createProject(page, "Website redesign", slug);

  await expect(page).toHaveURL(`/projects/${slug}`);
  await expect(page.getByRole("heading", { level: 1 })).toHaveText(
    "Website redesign",
  );
});

test("a slug that is in use is rejected next to the field", async ({
  page,
}) => {
  const slug = uniqueSlug("taken");
  await createProject(page, "First", slug);

  await createProject(page, "Second", slug);

  await expect(page.getByText("This slug is already in use.")).toBeVisible();
  await expect(page.getByLabel("Slug")).toHaveAttribute("aria-invalid", "true");
  await expect(page.getByLabel("Name")).toHaveValue("Second");
});

test("an invalid slug is explained, and nothing is created", async ({
  page,
}) => {
  await createProject(page, "Website redesign", "Website Redesign");

  await expect(
    page.getByText(/Use lowercase letters and numbers/),
  ).toBeVisible();
  await expect(page).toHaveURL("/projects/new");
});
```

Each test creates its own data, with a unique slug, so that tests can run in parallel and against a shared environment.

## Mocking the network

`page.route` intercepts requests made **by the browser**. In a Next.js application, Server Components and Server Actions call the API from the server, so `page.route` cannot see those requests.

- To test against controlled data, point the application's server at a test instance of the API, seeded by the test, or at a stub server.
- Use `page.route` for requests the browser makes itself, such as a third-party script or a client-side fetch, and to simulate failures:

```ts
import { expect, test } from "@playwright/test";

test("the page still works when analytics is blocked", async ({ page }) => {
  await page.route("**/analytics/**", (route) => route.abort());
  await page.goto("/projects");
  await expect(page.getByRole("heading", { name: "Projects" })).toBeVisible();
});
```

## Authentication

Sign in once, save the browser's storage state, and reuse it in every test, rather than signing in through the interface in each one. A setup project runs first, and the other projects depend on it:

```ts
// In playwright.config.ts, the projects become:
//   { name: "setup", testMatch: /.*\.setup\.ts/ },
//   { name: "chromium", use: { ...devices["Desktop Chrome"], storageState: "e2e/.auth/user.json" },
//     dependencies: ["setup"] },
import { expect, test as setup } from "@playwright/test";

setup("sign in", async ({ page }) => {
  await page.goto("/sign-in");
  await page.getByLabel("Email").fill(process.env.E2E_USER_EMAIL ?? "");
  await page.getByLabel("Password").fill(process.env.E2E_USER_PASSWORD ?? "");
  await page.getByRole("button", { name: "Sign in" }).click();
  await expect(page.getByRole("button", { name: "Account" })).toBeVisible();
  await page.context().storageState({ path: "e2e/.auth/user.json" });
});
```

Add `e2e/.auth/` to `.gitignore`. Test accounts exist only in test environments, and their credentials are CI secrets.

## Accessibility checks

[axe-core](https://github.com/dequelabs/axe-core) finds many common accessibility problems automatically: missing labels, insufficient contrast, invalid ARIA. Run it on the key pages:

```ts
// e2e/accessibility.spec.ts
import AxeBuilder from "@axe-core/playwright";
import { expect, test } from "@playwright/test";

for (const path of ["/projects", "/projects/new"]) {
  test(`${path} has no detectable accessibility violations`, async ({
    page,
  }) => {
    await page.goto(path);

    const results = await new AxeBuilder({ page })
      .withTags(["wcag2a", "wcag2aa", "wcag21a", "wcag21aa", "wcag22aa"])
      .analyze();

    expect(results.violations).toEqual([]);
  });
}
```

Automated checks find only part of the problems. They do not replace a keyboard pass by the author (see [Accessibility](../frontend/frontend-standards.md#accessibility)).

## Running tests

```bash
npx playwright install --with-deps chromium   # Once, and in CI
npm run build && npm run test:e2e             # Run everything against a production build
npx playwright test e2e/projects.spec.ts      # One file
npx playwright test --ui                      # Interactive mode, for writing and debugging
npx playwright show-report                    # The HTML report of the last run
npx playwright show-trace trace.zip           # Step through a recorded failure
```

In CI, upload `playwright-report/` as an artefact when the job fails, so that traces and screenshots can be inspected (see [CI/CD](../delivery/ci-cd.md#workflow-for-a-nextjs-application)).

## Related

- [Testing strategy](testing-strategy.md)
- [Next.js](../frontend/nextjs.md)
- [Frontend standards](../frontend/frontend-standards.md)
