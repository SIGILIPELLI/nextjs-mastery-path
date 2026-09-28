---
description: "Test Next.js App Router apps: Vitest for functions, Client Components and Server Actions, plus Playwright end-to-end tests on a production build."
---

# 07 · Testing with Vitest & Playwright

A Next.js app has code running in three places — the browser, the server during
rendering, and the server handling actions and requests — plus a build step that can
fail on its own. No single test tool covers all of it. The practical split:

| Layer | Tool | What it catches |
|---|---|---|
| Pure logic (validation, formatting, data mapping) | **Vitest** | Logic bugs, fast |
| Client Components | **Vitest + Testing Library** in jsdom | Rendering and interaction bugs |
| Server Actions / data layer | **Vitest** calling the functions directly (with a test DB) | Validation and authorisation bugs |
| Async Server Components, routing, caching, the real build | **Playwright** against `next build && next start` | Integration bugs, the things that only exist in the framework |

Everything below was run while writing this lesson (Vitest 5.0, Playwright 1.61,
Next.js 16.3.6), and the outputs are the real ones.

## Vitest setup

```bash
npm install --save-dev vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/dom
```

```ts title="vitest.config.mts"
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  resolve: { tsconfigPaths: true }, // understand the "@/..." alias
  test: {
    environment: "jsdom",
    exclude: ["e2e/**", "node_modules/**"], // keep Playwright specs out of Vitest
  },
});
```

Older guides add the `vite-tsconfig-paths` plugin for the alias; with the Vite version
Vitest 5 uses, it printed a notice that `resolve.tsconfigPaths: true` now does this
natively, so the plugin isn't needed.

Add `"test": "vitest"` to `package.json` scripts.

## Unit tests

```ts title="lib/slug.ts"
export function slugify(title: string): string {
  return title
    .normalize("NFKD")
    .replace(/[̀-ͯ]/g, "")
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, "-")
    .replace(/^-+|-+$/g, "");
}
```

```ts title="lib/__tests__/slug.test.ts"
import { describe, expect, it } from "vitest";
import { slugify } from "@/lib/slug";

describe("slugify", () => {
  it("lowercases and hyphenates", () => {
    expect(slugify("Hello, Next.js World")).toBe("hello-next-js-world");
  });
  it("strips accents and trims dashes", () => {
    expect(slugify("  Café Crème! ")).toBe("cafe-creme");
  });
});
```

## Client Component tests

```tsx title="components/Counter.tsx"
"use client";
import { useState } from "react";

export function Counter({ start = 0 }: { start?: number }) {
  const [n, setN] = useState(start);
  return <button onClick={() => setN(n + 1)}>Clicked {n} times</button>;
}
```

```tsx title="components/Counter.test.tsx"
import { expect, test } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";
import { Counter } from "./Counter";

test("increments on click", () => {
  render(<Counter start={2} />);
  const button = screen.getByRole("button", { name: "Clicked 2 times" });
  fireEvent.click(button);
  expect(screen.getByRole("button").textContent).toBe("Clicked 3 times");
});
```

```bash
npx vitest run
```

```text
 Test Files  2 passed (2)
      Tests  3 passed (3)
```

## Server Components and Server Actions

**Async Server Components** aren't well supported by jsdom-based renderers at the time
of writing; the Next.js docs recommend end-to-end tests for them. Keep them thin — fetch
with a data function, render with presentational components — and unit-test those parts
separately.

**Server Actions** are just async functions. Test the logic by calling them with a
`FormData`, pointing the data layer at a test database:

```ts title="app/tasks/__tests__/actions.test.ts"
import { beforeEach, expect, test, vi } from "vitest";

vi.mock("server-only", () => ({}));                 // the guard throws outside Next's server build
vi.mock("next/cache", () => ({ revalidatePath: vi.fn() }));

beforeEach(() => {
  process.env.DATABASE_PATH = ":memory:";
  vi.resetModules();                                  // fresh in-memory DB per test
});

test("rejects an empty title", async () => {
  const { createTask } = await import("../actions");
  const fd = new FormData();
  fd.set("title", "   ");
  const state = await createTask({}, fd);
  expect(state.errors?.title?.[0]).toBe("Title is required");
});
```

Mocking `next/cache` and `server-only` is necessary because those modules rely on the
Next.js runtime. We ran this test against the Level 2 task manager's real
`actions.ts` and in-memory SQLite, and it passed. On our first run, though, Vitest also
picked up `e2e/tasks.spec.ts` and failed with "Playwright Test did not expect test() to
be called here" — hence the `exclude` in the config above. With it:

```text
 Test Files  3 passed (3)
      Tests  4 passed (4)
```

## End-to-end with Playwright

```bash
npm install --save-dev @playwright/test
npx playwright install chromium
```

The configuration and test in Level 2 · 10 build the app, start it on a spare port with
an isolated database, and drive a real browser through create, complete and delete:

```text
  ✓  1 [chromium] › e2e/tasks.spec.ts:3:5 › create, complete and delete a task (573ms)

  1 passed (6.3s)
```

Guidelines for stable e2e tests:

- **Test the production build**, not `next dev` — that's what users get, and it's
  where caching and prerendering behave for real.
- **Locate by role and label** (`getByRole("button", { name: "Add task" })`), which also
  checks accessibility. Be aware of framework elements: the route announcer
  `<div role="alert">` that Next.js adds made a bare `getByRole("alert")` ambiguous in
  our first run.
- **Unique data per test** (`Buy milk ${Date.now()}`) so tests don't collide.
- **Wait on outcomes, not timeouts** — `expect(...).toBeVisible()` retries.

## What to test where: a worked plan

For the task manager:

| Behaviour | Test |
|---|---|
| Title validation rules | Vitest on the Zod schema / action |
| `TaskForm` shows the error message it receives | Vitest + Testing Library |
| User A can't delete user B's task | Vitest on the action with two sessions |
| Adding a task shows it in the list (revalidation works) | Playwright |
| Unknown task ID returns 404 | Playwright `request.get` and status check |
| Build succeeds, types check | CI runs `next build` |

## How It Actually Works

Vitest runs test files in Node with Vite's module pipeline; the `jsdom` environment
installs a fake `window` and `document` so React DOM can render Client Components.
But it's not the Next.js compiler: `"use client"`/`"use server"` directives are plain
strings to it, there's no RSC renderer, and framework modules such as `next/cache`
expect request context. That's why framework-dependent behaviour belongs in Playwright,
which tests the real thing: Playwright's `webServer` option starts `next start`, waits
for the URL to respond, then launches a browser that talks to it over HTTP exactly as a
user would.

## Common mistakes

- **Trying to unit-test everything**, including async Server Components, with heavy
  mocking that tests the mocks.
- **Running e2e tests against `next dev`** and chasing dev-only timing issues.
- **Sharing a database between tests and development.**
- **CSS-selector-based locators** that break on every style change.
- **Skipping `next build` in CI**; type and prerender errors only show there.

## Exercise

1. Add Vitest to your project and write tests for three pure functions and one Client
   Component.
2. Write a Vitest test for a Server Action's validation using the mocking pattern above,
   and get it passing against an in-memory SQLite database.
3. Add a Playwright test that requests `/tasks/999999/edit` and asserts a 404 status
   using `page.goto` and the returned response.
