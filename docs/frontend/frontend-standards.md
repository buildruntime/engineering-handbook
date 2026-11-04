[Engineering handbook](../../README.md) / Frontend

# Frontend standards

The standards every Buildruntime web interface meets, whatever framework it uses. The technical detail is in [TypeScript](typescript.md), [React](react.md) and [Next.js](nextjs.md).

## Tooling

| Concern | Tool | Runs |
| :-- | :-- | :-- |
| Package management | npm, with a committed `package-lock.json` | `npm ci` in CI, never `npm install` |
| Types | TypeScript, strict | `tsc --noEmit` in CI |
| Linting | ESLint with the Next.js configuration | In CI, and in the editor |
| Formatting | Prettier, with default settings | In CI as `prettier --check .`, and on save |
| Unit and component tests | Vitest and Testing Library | In CI |
| End-to-end tests | Playwright | In CI, against a production build (see [Playwright](../quality/playwright.md)) |

Pin the Node.js version in `.nvmrc` and in the `engines` field of `package.json`, and use the same version in CI and in the container image. Use an even-numbered release that is in its Long Term Support period.

The applications build with `output: "standalone"` (see [Next.js](nextjs.md#configuration)), and `next start` does not support that output. `start` therefore runs the standalone server, and `postbuild`, which npm runs after `build`, copies the static assets that the standalone server expects next to it. The same server runs locally, in end-to-end tests and in the container image.

```json
{
  "engines": { "node": ">=24 <25" },
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "postbuild": "cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/",
    "start": "node .next/standalone/server.js",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "format": "prettier --write .",
    "test": "vitest run",
    "test:e2e": "playwright test"
  }
}
```

## Accessibility

Interfaces meet [WCAG 2.2](https://www.w3.org/TR/WCAG22/) level AA. Accessibility is part of the definition of done, not a later phase: it is far cheaper to build in than to retrofit.

- **Use the right HTML element.** A `<button>` for actions, an `<a href>` for navigation, `<label>` for every input, headings in order. Native elements come with keyboard support and screen reader semantics that a `<div onClick>` does not have.
- **Everything works with a keyboard.** Every interactive element can be reached with Tab, has a visible focus style, and can be operated with Enter or Space. Test this by hand on every change to an interactive component.
- **Text has enough contrast**: at least 4.5:1 for normal text, and 3:1 for large text and for the boundaries of controls.
- **Colour is never the only signal.** An error is red and also has an icon or a message.
- **Images have `alt` text** that says what the image conveys, or `alt=""` if it is decorative.
- **Changes are announced.** Content that appears in response to an action, such as a status message or a validation error, uses a live region (`role="status"` or `role="alert"`) or receives focus.
- **Motion respects `prefers-reduced-motion`.**

Automated checks find some problems, not all. Playwright tests can run axe on key pages (see [Playwright](../quality/playwright.md#accessibility-checks)), and a keyboard pass by the author is part of every pull request that changes an interface.

## Styling

Each project chooses one styling approach, either Tailwind CSS or CSS Modules, and uses it throughout. Mixing approaches in one codebase makes every style change a search through three systems.

- Define colours, spacing, type sizes and radii once, as design tokens: CSS custom properties, or the Tailwind theme. Components use the tokens, not literal values.
- Support light and dark themes through the tokens, so that components do not need to know which theme is active.
- Use relative units (`rem`) for type and spacing, so that the interface respects the user's font size setting.
- Lay out with flexbox and grid. Design for the smallest screen first, and add breakpoints where the layout needs them.

A minimal global stylesheet sets the tokens, the base styles and the utility class used by the components in this handbook:

```css
/* app/globals.css */
:root {
  --color-text: #0e0f11;
  --color-text-secondary: #6b6f76;
  --color-background: #f6f5f2;
  --color-border: #e3e2de;
  --color-accent: #e5482d;
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-4: 1rem;
  --radius: 0.375rem;
  color-scheme: light dark;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-text: #f6f5f2;
    --color-text-secondary: #9ea2a9;
    --color-background: #0e0f11;
    --color-border: #26282c;
  }
}

body {
  margin: 0;
  padding: var(--space-4);
  font-family: system-ui, sans-serif;
  line-height: 1.5;
  color: var(--color-text);
  background: var(--color-background);
}

:focus-visible {
  outline: 2px solid var(--color-accent);
  outline-offset: 2px;
}

/* Hidden visually, still read by screen readers. */
.visually-hidden {
  position: absolute;
  width: 1px;
  height: 1px;
  overflow: hidden;
  clip-path: inset(50%);
  white-space: nowrap;
}

@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

## Performance

Pages meet the "good" thresholds of Google's [Core Web Vitals](https://web.dev/articles/vitals), measured at the 75th percentile of real page loads:

| Metric | Measures | Good |
| :-- | :-- | :-- |
| Largest Contentful Paint (LCP) | How soon the main content appears | 2.5 s or less |
| Interaction to Next Paint (INP) | How quickly the page responds to input | 200 ms or less |
| Cumulative Layout Shift (CLS) | How much the layout moves while loading | 0.1 or less |

The practices that matter most are in [Performance](../quality/performance.md#frontend). In short: render on the server, send less JavaScript, size images, and reserve space for content that loads later.

## Browser support

Support the current and previous major versions of Chrome, Edge, Firefox and Safari, on desktop and mobile. Use features that all of them support, and check [Baseline](https://web.dev/baseline) status before using a newer one. Polyfill only what a supported browser lacks.

## Internationalisation

- Format dates, numbers and currencies with `Intl` APIs and an explicit locale, never by building strings.
- Keep user-facing text out of logic, so that it can be translated later without rewriting components.
- Store and transmit times in UTC. Convert to the user's time zone only for display.

## Related

- [TypeScript](typescript.md)
- [React](react.md)
- [Next.js](nextjs.md)
- [Performance](../quality/performance.md)
