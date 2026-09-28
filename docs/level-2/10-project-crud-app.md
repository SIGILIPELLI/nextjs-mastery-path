---
description: "Level 2 project: a full-stack Next.js task manager with Drizzle and SQLite, Zod-validated Server Actions, revalidation and a Playwright test."
---

# 10 · Project — Full-Stack Task Manager

This project pulls Level 2 together into a small but complete application: create,
list, edit, complete and delete tasks, stored in SQLite through Drizzle, with every
write going through a validated Server Action. It finishes with an end-to-end test.

Everything here was built with `next build` on Next.js 16.3.6, run with `next start`,
clicked through in a browser, and tested with Playwright; outputs shown are from those
runs.

## Features and where each lesson appears

| Feature | Lesson |
|---|---|
| `/tasks/[id]/edit` with `notFound()` for bad IDs | 01, 02 |
| Create/update/toggle/delete via Server Actions + `useActionState` | 03 |
| `revalidatePath` after every write; `connection()` for live data | 04 |
| Drizzle schema and queries on SQLite | 08 |

(Authentication from lesson 07 is left as the exercise so the core stays readable.)

## 1. Setup

```bash
npx create-next-app@latest tasks-app --ts --app --eslint --tailwind --import-alias "@/*" --use-npm
cd tasks-app
npm install drizzle-orm better-sqlite3 zod server-only
npm install --save-dev drizzle-kit @types/better-sqlite3
```

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = { serverExternalPackages: ["better-sqlite3"] };

export default nextConfig;
```

## 2. Database

Use the `db/schema.ts` from lesson 08 unchanged. For a self-contained demo, the
connection module creates the table if it's missing; in a real project you'd run
`drizzle-kit migrate` instead.

```ts title="db/index.ts"
import "server-only";
import Database from "better-sqlite3";
import { drizzle } from "drizzle-orm/better-sqlite3";
import * as schema from "./schema";

const sqlite = new Database(process.env.DATABASE_PATH ?? "tasks.db");
sqlite.exec(`CREATE TABLE IF NOT EXISTS tasks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  notes TEXT NOT NULL DEFAULT '',
  done INTEGER NOT NULL DEFAULT 0,
  created_at INTEGER NOT NULL
)`);

export const db = drizzle(sqlite, { schema });
```

## 3. Server Actions

```ts title="app/tasks/actions.ts"
"use server";

import { z } from "zod";
import { eq } from "drizzle-orm";
import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { db } from "@/db";
import { tasks } from "@/db/schema";

const TaskInput = z.object({
  title: z.string().trim().min(1, "Title is required").max(120),
  notes: z.string().trim().max(2000).default(""),
});

export type FormState = { errors?: Record<string, string[]>; message?: string };

export async function createTask(_prev: FormState, formData: FormData): Promise<FormState> {
  const parsed = TaskInput.safeParse({
    title: formData.get("title"),
    notes: formData.get("notes") ?? "",
  });
  if (!parsed.success) return { errors: z.flattenError(parsed.error).fieldErrors };
  db.insert(tasks).values(parsed.data).run();
  revalidatePath("/tasks");
  return { message: "Task added" };
}

export async function updateTask(id: number, _prev: FormState, formData: FormData): Promise<FormState> {
  const parsed = TaskInput.safeParse({ title: formData.get("title"), notes: formData.get("notes") ?? "" });
  if (!parsed.success) return { errors: z.flattenError(parsed.error).fieldErrors };
  db.update(tasks).set(parsed.data).where(eq(tasks.id, id)).run();
  revalidatePath("/tasks");
  redirect("/tasks");
}

export async function toggleTask(id: number, done: boolean) {
  db.update(tasks).set({ done: !done }).where(eq(tasks.id, id)).run();
  revalidatePath("/tasks");
}

export async function deleteTask(id: number) {
  db.delete(tasks).where(eq(tasks.id, id)).run();
  revalidatePath("/tasks");
}
```

`updateTask` takes the ID as its first parameter so the edit page can `bind` it;
`useActionState` then supplies `(prevState, formData)` after it.

## 4. A reusable form

```tsx title="app/tasks/TaskForm.tsx"
"use client";

import { useActionState } from "react";
import type { FormState } from "./actions";

type Props = {
  action: (prev: FormState, formData: FormData) => Promise<FormState>;
  initial?: { title: string; notes: string };
  submitLabel: string;
};

export function TaskForm({ action, initial, submitLabel }: Props) {
  const [state, formAction, pending] = useActionState(action, {});
  return (
    <form action={formAction}>
      <label htmlFor="title">Title</label>
      <input id="title" name="title" defaultValue={initial?.title} aria-describedby="title-error" />
      {state.errors?.title && <p id="title-error" role="alert">{state.errors.title[0]}</p>}
      <label htmlFor="notes">Notes</label>
      <textarea id="notes" name="notes" defaultValue={initial?.notes} />
      <button type="submit" disabled={pending}>{pending ? "Saving…" : submitLabel}</button>
      {state.message && <p role="status">{state.message}</p>}
    </form>
  );
}
```

The same component serves "create" and "edit" — only the action and initial values
differ.

## 5. Pages

```tsx title="app/tasks/page.tsx"
import Link from "next/link";
import { desc } from "drizzle-orm";
import { connection } from "next/server";
import { db } from "@/db";
import { tasks } from "@/db/schema";
import { createTask, deleteTask, toggleTask } from "./actions";
import { TaskForm } from "./TaskForm";

export const metadata = { title: "Tasks" };

export default async function TasksPage() {
  await connection(); // always render at request time: this page shows live DB rows
  const rows = db.select().from(tasks).orderBy(desc(tasks.createdAt)).all();
  return (
    <main>
      <h1>Tasks</h1>
      <TaskForm action={createTask} submitLabel="Add task" />
      <ul>
        {rows.map((t) => (
          <li key={t.id}>
            <form action={toggleTask.bind(null, t.id, t.done)} style={{ display: "inline" }}>
              <button aria-label={t.done ? `Mark ${t.title} as not done` : `Mark ${t.title} as done`}>
                {t.done ? "☑" : "☐"}
              </button>
            </form>{" "}
            <Link href={`/tasks/${t.id}/edit`}>{t.title}</Link>
            <form action={deleteTask.bind(null, t.id)} style={{ display: "inline" }}>
              <button>Delete</button>
            </form>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

```tsx title="app/tasks/[id]/edit/page.tsx"
import { notFound } from "next/navigation";
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { tasks } from "@/db/schema";
import { updateTask } from "../../actions";
import { TaskForm } from "../../TaskForm";

export default async function EditTask({ params }: PageProps<"/tasks/[id]/edit">) {
  const { id } = await params;
  const taskId = Number(id);
  if (!Number.isInteger(taskId)) notFound();
  const task = db.select().from(tasks).where(eq(tasks.id, taskId)).get();
  if (!task) notFound();
  return (
    <main>
      <h1>Edit task</h1>
      <TaskForm action={updateTask.bind(null, task.id)} initial={task} submitLabel="Save" />
    </main>
  );
}
```

## 6. Build

```text
├ ƒ /tasks
└ ƒ /tasks/[id]/edit
```

Both are dynamic — correct for pages showing live database rows.

## 7. Try it

Clicking **Add task** with an empty title showed "Title is required" under the field.
Typing "Write lesson 3" and pressing Enter showed the "Task added" status and the new
row, with the input cleared. Clicking the checkbox button changed its accessible name
from "Mark Write lesson 3 as done" to "Mark Write lesson 3 as not done".

## 8. An end-to-end test

```bash
npm install --save-dev @playwright/test
npx playwright install chromium
```

```ts title="playwright.config.ts"
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "e2e",
  use: { baseURL: "http://localhost:3100" },
  webServer: {
    command: "npm run build && npx next start -p 3100",
    url: "http://localhost:3100",
    reuseExistingServer: !process.env.CI,
    timeout: 120_000,
    env: { DATABASE_PATH: "e2e.db" },
  },
  projects: [{ name: "chromium", use: { ...devices["Desktop Chrome"] } }],
});
```

```ts title="e2e/tasks.spec.ts"
import { test, expect } from "@playwright/test";

test("create, complete and delete a task", async ({ page }) => {
  const title = `Buy milk ${Date.now()}`;
  await page.goto("/tasks");

  await page.getByRole("button", { name: "Add task" }).click();
  await expect(page.getByRole("alert").filter({ hasText: "Title is required" })).toBeVisible();

  await page.getByLabel("Title").fill(title);
  await page.getByRole("button", { name: "Add task" }).click();
  await expect(page.getByRole("link", { name: title })).toBeVisible();

  await page.getByRole("button", { name: `Mark ${title} as done` }).click();
  await expect(page.getByRole("button", { name: `Mark ${title} as not done` })).toBeVisible();

  const row = page.getByRole("listitem").filter({ hasText: title });
  await row.getByRole("button", { name: "Delete" }).click();
  await expect(page.getByRole("link", { name: title })).toHaveCount(0);
});
```

```text
  ✓  1 [chromium] › e2e/tasks.spec.ts:3:5 › create, complete and delete a task (573ms)

  1 passed (6.3s)
```

The first version of this test used `page.getByRole("alert")` alone and **failed** with
a strict-mode violation: it matched two elements — our error paragraph and a
`<div role="alert" id="__next-route-announcer__">` that Next.js adds to every page to
announce route changes to screen readers. Filtering by text fixed it. It's a good
reminder that the framework adds its own accessible elements.

## How It Actually Works

Trace one click on **Delete**: the button submits a form whose action is
`deleteTask` bound to the row's ID. React serialises the bound argument, POSTs to
`/tasks` with the action's ID header, and the server runs `deleteTask(7)`. The
`revalidatePath("/tasks")` call marks the route stale; because the action was invoked
from that very page, Next.js renders the fresh `/tasks` RSC payload and returns it in
the same response. React reconciles it: the `<li>` disappears, while the `TaskForm`
(a Client Component at the same position) keeps its state. No client-side list state,
no refetch logic — the server's render is the single source of truth.

## Common mistakes

- **No authorisation.** Anyone can call `deleteTask(anyId)`. Fix it in the exercise.
- **Treating `id` from the URL as a number without checking.**
- **Forgetting `connection()`**, then seeing an empty list forever because the page was
  prerendered at build time before any tasks existed.
- **Sharing one SQLite file between the dev server and e2e tests** — the config above
  uses `DATABASE_PATH` to isolate them.

## Exercise

1. Add authentication from lesson 07: a `userId` column, tasks scoped to the signed-in
   user in every query, and actions that check ownership before updating or deleting.
2. Add filtering by status (`?status=open|done`) using URL state (lesson 09).
3. Use `useOptimistic` so toggling feels instant, and verify with network throttling.
4. Extend the Playwright test to cover editing a task.
