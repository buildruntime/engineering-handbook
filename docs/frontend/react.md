[Engineering handbook](../../README.md) / Frontend

# React

How Buildruntime writes React components. The examples are function components in TypeScript, as used in a Next.js App Router project. Next.js-specific topics, such as Server Components and Server Actions, are in [Next.js](nextjs.md).

## Components

- One exported component per file, named after the file: `ProjectList` in `project-list.tsx`. Small helper components used only by that one may live in the same file.
- Props are a named type, `ProjectListProps`, declared next to the component.
- Keep components small enough to understand at a glance. A component that fetches data, manages a form and renders a table is three components.
- Prefer composition, passing `children` or components as props, to boolean props that switch between layouts. Five boolean props make thirty-two variants, most of them untested.
- Render lists with a `key` that identifies the item, such as its ID. Never the array index for a list that can be reordered or filtered: React would reuse the wrong component's state.

```tsx
// components/project-list.tsx
import Link from "next/link";
import type { Project } from "@/lib/api/schemas";

type ProjectListProps = {
  projects: Project[];
  emptyMessage?: string;
};

export function ProjectList({
  projects,
  emptyMessage = "No projects yet.",
}: ProjectListProps) {
  if (projects.length === 0) {
    return <p>{emptyMessage}</p>;
  }

  return (
    <ul>
      {projects.map((project) => (
        <li key={project.id}>
          <Link href={`/projects/${project.slug}`}>{project.name}</Link>
          {project.archived_at !== null && <span> (archived)</span>}
        </li>
      ))}
    </ul>
  );
}
```

`ProjectList` has no state and no effects, so it works as a Server Component and as a Client Component. Most components should be like this.

## State

Keep as little state as possible, as close as possible to where it is used.

| Kind of state | Where it lives |
| :-- | :-- |
| Data from the server | Fetched in a Server Component and passed down as props. In Client Components, use the framework's data tools, not hand-written effects. |
| What the URL should be able to restore: filters, sort order, current tab, pagination | The URL's search parameters. A user can then share and bookmark it, and the back button works. |
| Form input | The form itself, read with `FormData` on submit, unless something must react to each keystroke. |
| UI state used by one component | `useState` in that component. |
| UI state shared by a few nearby components | `useState` in their closest common parent, passed down as props. |
| Truly global UI state: theme, the current user | React context, provided near the root. |

Do not store what you can compute. Derived state goes out of date as soon as the value it was derived from changes:

```tsx
import { useState } from "react";

type Task = { id: string; title: string; done: boolean };

// Bad: remaining is copied into state and is wrong after the first change to tasks.
export function TaskSummaryBad({ tasks }: { tasks: Task[] }) {
  const [remaining] = useState(tasks.filter((task) => !task.done).length);
  return <p>{remaining} tasks left</p>;
}

// Good: compute it during render.
export function TaskSummary({ tasks }: { tasks: Task[] }) {
  const remaining = tasks.filter((task) => !task.done).length;
  return <p>{remaining} tasks left</p>;
}
```

Computing during render is fast for anything a user can see on one screen. Reach for `useMemo` when you have measured that a calculation is slow, not by default. If the project uses the React Compiler, it memoises automatically and manual `useMemo` and `useCallback` are rarely needed.

## Effects

`useEffect` is for synchronising a component with something outside React: a browser API, a timer, a subscription, a non-React widget. It is not for transforming data or for reacting to user events.

- **Responding to an event?** Do it in the event handler, not in an effect that watches for a state change.
- **Computing a value from props or state?** Compute it during render.
- **Fetching data?** Fetch it in a Server Component, or use the framework's data tools.
- **Every effect that starts something must stop it** in its clean-up function: clear the timer, remove the listener, abort the request.

A small hook shows the pattern. The clean-up function cancels the pending timer when `value` changes or the component unmounts:

```ts
// hooks/use-debounced-value.ts
import { useEffect, useState } from "react";

export function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(timer);
  }, [value, delayMs]);

  return debounced;
}
```

## Custom hooks

Extract a custom hook when the same stateful logic appears in two components, or when it makes one component easier to read. Name it `useSomething`, and give it a narrow, typed interface.

This search box keeps its text in local state, and writes it to the URL a moment after the user stops typing, so that the page can filter on the server:

```tsx
// components/project-search.tsx
"use client";

import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { useEffect, useState } from "react";
import { useDebouncedValue } from "@/hooks/use-debounced-value";

export function ProjectSearch() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [query, setQuery] = useState(searchParams.get("q") ?? "");
  const debouncedQuery = useDebouncedValue(query, 300);

  useEffect(() => {
    const params = new URLSearchParams(searchParams);
    if (debouncedQuery) {
      params.set("q", debouncedQuery);
    } else {
      params.delete("q");
    }
    if (params.toString() !== searchParams.toString()) {
      router.replace(`${pathname}?${params.toString()}`);
    }
  }, [debouncedQuery, pathname, router, searchParams]);

  return (
    <label>
      Search projects
      <input
        type="search"
        value={query}
        onChange={(event) => setQuery(event.target.value)}
      />
    </label>
  );
}
```

The effect synchronises the URL, an external system, with the debounced value. It compares before replacing, so that it does not loop.

## Interaction and feedback

Every action a user takes gets visible feedback, and nothing important depends on a hover.

```tsx
// components/copy-button.tsx
"use client";

import { useEffect, useState } from "react";

type CopyButtonProps = {
  text: string;
  label?: string;
};

export function CopyButton({ text, label = "Copy" }: CopyButtonProps) {
  const [status, setStatus] = useState<"idle" | "copied" | "failed">("idle");

  useEffect(() => {
    if (status === "idle") return;
    const timer = setTimeout(() => setStatus("idle"), 2000);
    return () => clearTimeout(timer);
  }, [status]);

  async function copy() {
    try {
      await navigator.clipboard.writeText(text);
      setStatus("copied");
    } catch {
      setStatus("failed");
    }
  }

  return (
    <button type="button" onClick={copy}>
      {status === "copied"
        ? "Copied"
        : status === "failed"
          ? "Copy failed"
          : label}
      <span role="status" className="visually-hidden">
        {status === "copied" ? "Copied to clipboard" : ""}
      </span>
    </button>
  );
}
```

- The button is a real `<button>` with `type="button"`, so it works with a keyboard and does not submit a surrounding form.
- The `role="status"` region announces the result to screen readers.
- The failure case is handled. The Clipboard API is not available in every context, such as a page served over plain HTTP.

## Forms

Forms in Next.js submit to a Server Action, with `useActionState` for the pending state and the result. The example is in [Server Actions](nextjs.md#server-actions). Whatever the framework:

- every input has a visible `<label>`;
- validation errors appear next to the field they belong to, and are linked to it with `aria-describedby`;
- the submit button is disabled while the form is submitting, and the form says so;
- the server validates everything again. Client-side validation is for convenience, not for security.

## Related

- [Next.js](nextjs.md)
- [TypeScript](typescript.md)
- [Frontend standards](frontend-standards.md)
