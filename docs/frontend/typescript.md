[Engineering handbook](../../README.md) / Frontend

# TypeScript

All Buildruntime frontend code is TypeScript in strict mode. Types are there to catch mistakes before users do, and to document what a value can be. This page covers compiler settings, how to model data with types, and how to validate data that comes from outside the program.

## Compiler settings

Start from the settings Next.js generates, and make them stricter:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "module": "esnext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "allowJs": false,
    "noEmit": true,
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./*"] }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts"
  ],
  "exclude": ["node_modules"]
}
```

| Setting | Why |
| :-- | :-- |
| `strict` | Enables the checks that make types trustworthy, including `strictNullChecks`. Non-negotiable. |
| `noUncheckedIndexedAccess` | `items[0]` has type `Item \| undefined`, because the array may be empty. Catches a common class of runtime errors. |
| `noImplicitOverride` | A method that overrides a base class method must say `override`, so that renaming the base method is caught. |
| `noFallthroughCasesInSwitch` | A `case` without `break` or `return` is an error. |
| `allowJs: false` | No untyped JavaScript in the project. |

CI runs `tsc --noEmit` on every pull request. Type errors fail the build; they are not warnings.

## Rules

- **No `any`.** Use `unknown` for values whose type you do not know yet, and narrow it. If a library forces `any` on you, contain it in one function with a proper return type.
- **No non-null assertions (`value!`)** outside tests. If you know a value is defined, the types should know it too; if they cannot, check it and throw a meaningful error.
- **Prefer type inference for local variables**, and write types explicitly at boundaries: exported functions, component props, API responses.
- **Use `type` for object shapes and unions.** Use `interface` only when you need declaration merging, which is rare in application code. The point is consistency.
- **Do not use `enum`.** Use a union of string literals, which needs no runtime code and works with `isolatedModules`.
- **Use `satisfies`** to check that a value matches a type without widening it.

```ts
// A union of literals instead of an enum.
type ProjectStatus = "active" | "archived";

// satisfies checks the object against the type, and keeps the literal keys,
// so statusLabel.active is known to exist.
const statusLabel = {
  active: "Active",
  archived: "Archived",
} satisfies Record<ProjectStatus, string>;

export function label(status: ProjectStatus): string {
  return statusLabel[status];
}
```

## Modelling with types

Make invalid states unrepresentable. A type with several optional fields that are only valid in some combinations lets the compiler accept combinations that make no sense.

```ts
// Bad: nothing stops { status: "success", error: "..." } or a success with no data.
type LoadStateLoose<T> = {
  status: "idle" | "loading" | "success" | "error";
  data?: T;
  error?: string;
};

// Good: a discriminated union. Each state carries exactly the data it has.
type LoadState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; error: string };

export function describe<T>(
  state: LoadState<T>,
  render: (data: T) => string,
): string {
  switch (state.status) {
    case "idle":
      return "Not started";
    case "loading":
      return "Loading…";
    case "success":
      return render(state.data);
    case "error":
      return `Failed: ${state.error}`;
    default:
      return assertNever(state);
  }
}

// If a new status is added and not handled above, this call stops compiling.
export function assertNever(value: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(value)}`);
}
```

Give identifiers that must not be mixed up their own types. A "branded" type costs nothing at runtime and stops a project ID from being passed where a user ID is expected:

```ts
type Brand<T, Name extends string> = T & { readonly __brand: Name };

export type ProjectId = Brand<string, "ProjectId">;
export type UserId = Brand<string, "UserId">;

export function removeMember(projectId: ProjectId, userId: UserId): void {
  console.log(`Removing ${userId} from ${projectId}`);
}

const projectId = "6a0d3c4e" as ProjectId;
const userId = "91b2f7aa" as UserId;

removeMember(projectId, userId);
// @ts-expect-error: the arguments are the wrong way round.
removeMember(userId, projectId);
```

## Validating data at the boundary

Types describe what the program expects. They do not check what arrives at runtime. Data from outside the program, such as API responses, form input, environment variables, URL parameters and `localStorage`, must be validated before the rest of the code relies on it.

Buildruntime projects use [Zod](https://zod.dev/) for this. A schema validates the data and produces the TypeScript type, so the two cannot drift apart.

```ts
// lib/env.ts
import "server-only";
import { z } from "zod";

const schema = z.object({
  API_URL: z.url(),
  API_TOKEN: z.string().min(1),
  NODE_ENV: z
    .enum(["development", "test", "production"])
    .default("development"),
});

export type Env = z.infer<typeof schema>;

let cached: Env | undefined;

// Throws on first use, listing every missing or invalid variable.
export function env(): Env {
  cached ??= schema.parse(process.env);
  return cached;
}
```

The variables are read on first use, not when the module is imported, because `next build` imports server modules too, and the build environment should not need runtime secrets. [Next.js](nextjs.md#environment-variables) shows how to check them when the server starts. `import "server-only"` makes the build fail if a Client Component imports this module, so that server secrets cannot end up in the browser bundle.

The same applies to API responses. Parse them where they enter the program, and let the rest of the code use the typed result:

```ts
// lib/api/schemas.ts
import { z } from "zod";

export const projectSchema = z.object({
  id: z.uuid(),
  slug: z.string(),
  name: z.string(),
  created_at: z.iso.datetime({ offset: true }),
  archived_at: z.iso.datetime({ offset: true }).nullable(),
});

export const projectPageSchema = z.object({
  items: z.array(projectSchema),
  next_cursor: z.string().nullable(),
});

export type Project = z.infer<typeof projectSchema>;
export type ProjectPage = z.infer<typeof projectPageSchema>;
```

If the API publishes an OpenAPI document, generate these types from it instead of writing them by hand, and run the generator in CI to catch drift (see [REST API design](../backend/api-design.md#documentation)).

## Errors

Throw `Error` objects, or subclasses, never strings. In a `catch` block the error has type `unknown`; narrow it before using it:

```ts
export async function loadOrFallback<T>(
  load: () => Promise<T>,
  fallback: T,
): Promise<T> {
  try {
    return await load();
  } catch (error: unknown) {
    if (error instanceof Error && error.name === "AbortError") {
      return fallback;
    }
    throw error;
  }
}
```

For expected failures that the caller must handle, such as a form submission that the API rejected, return a result type rather than throwing. The compiler then makes the caller deal with both outcomes. [Server Actions](nextjs.md#server-actions) use this pattern.

## Related

- [React](react.md)
- [Next.js](nextjs.md)
- [Frontend standards](frontend-standards.md)
