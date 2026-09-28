---
description: "Connect Next.js to a database with Drizzle ORM and SQLite: schema, typed queries, drizzle-kit migrations, and avoiding build-time queries."
---

# 08 · Databases with an ORM

Server Components and Server Actions can talk to a database directly, so a Next.js app
doesn't need a separate API layer to be full-stack. This lesson wires up **Drizzle ORM**
with **SQLite** — a combination that runs from a local file with no server or account —
and covers the habits that carry over to Postgres or MySQL in production.

## Why an ORM (and why Drizzle here)

You can use a plain driver and SQL strings. An ORM or query builder adds:

- a **schema in code** that TypeScript understands, so query results are typed;
- **migrations** generated from schema changes;
- **parameterised queries** by default, which removes SQL injection by construction.

Drizzle is a thin, SQL-shaped TypeScript layer; Prisma is the other popular choice,
with its own schema language and generated client. Both work well with Next.js. This
course uses Drizzle because its queries map one-to-one to SQL, which makes it easy to
see what runs. Versions used while writing: `drizzle-orm` 0.45, `drizzle-kit` 0.31,
`better-sqlite3` 13 — the API has been changing toward a 1.0 release, so check the
Drizzle docs if something differs.

## Install

```bash
npm install drizzle-orm better-sqlite3 server-only
npm install --save-dev drizzle-kit @types/better-sqlite3
```

`better-sqlite3` is a native module. Recent npm versions may ask you to approve its
install script (`npm approve-scripts better-sqlite3`); without it the binary isn't
built and you'll get a load error at runtime.

## Schema

```ts title="db/schema.ts"
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";

export const tasks = sqliteTable("tasks", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  title: text("title").notNull(),
  notes: text("notes").notNull().default(""),
  done: integer("done", { mode: "boolean" }).notNull().default(false),
  createdAt: integer("created_at", { mode: "timestamp" }).notNull().$defaultFn(() => new Date()),
});

export type Task = typeof tasks.$inferSelect;
export type NewTask = typeof tasks.$inferInsert;
```

SQLite has no boolean or date types; the `mode` options store them as integers and
convert on the way in and out, so your code sees `boolean` and `Date`.

## Connection

```ts title="db/index.ts"
import "server-only";
import Database from "better-sqlite3";
import { drizzle } from "drizzle-orm/better-sqlite3";
import * as schema from "./schema";

const sqlite = new Database(process.env.DATABASE_PATH ?? "tasks.db");
export const db = drizzle(sqlite, { schema });
```

`server-only` makes it a build error to import the database from client code.

Tell Next.js not to bundle the native module:

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = { serverExternalPackages: ["better-sqlite3"] };

export default nextConfig;
```

(Next.js already keeps a list of well-known native packages external, so this is often
a no-op, but it documents intent and matters for less common drivers.)

## Migrations with drizzle-kit

```ts title="drizzle.config.ts"
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  dialect: "sqlite",
  schema: "./db/schema.ts",
  out: "./drizzle",
  dbCredentials: { url: process.env.DATABASE_PATH ?? "tasks.db" },
});
```

```bash
npx drizzle-kit generate   # write SQL migration files from schema changes
npx drizzle-kit migrate    # apply pending migrations to the database
```

Running `generate` against the schema above produced (file name is random per run):

```text
1 tables
tasks 5 columns 0 indexes 0 fks

[✓] Your SQL migration file ➜ drizzle/0000_flashy_stranger.sql 🚀
```

```sql title="drizzle/0000_….sql"
CREATE TABLE `tasks` (
	`id` integer PRIMARY KEY AUTOINCREMENT NOT NULL,
	`title` text NOT NULL,
	`notes` text DEFAULT '' NOT NULL,
	`done` integer DEFAULT false NOT NULL,
	`created_at` integer NOT NULL
);
```

Commit the `drizzle/` folder; migrations are part of your code. Run `migrate` as a
deploy step, **not** inside `next build` or at module load in production.

## Queries

```ts
import { and, desc, eq, like } from "drizzle-orm";
import { db } from "@/db";
import { tasks } from "@/db/schema";

// select
const open = db.select().from(tasks).where(eq(tasks.done, false)).orderBy(desc(tasks.createdAt)).all();

// search (parameterised — the % pattern is a bound value, not SQL text)
const found = db.select().from(tasks).where(and(eq(tasks.done, false), like(tasks.title, `%${q}%`))).all();

// insert, update, delete
db.insert(tasks).values({ title: "Write tests" }).run();
db.update(tasks).set({ done: true }).where(eq(tasks.id, 7)).run();
db.delete(tasks).where(eq(tasks.id, 7)).run();
```

With `better-sqlite3` these calls are **synchronous** (`.all()`, `.get()`, `.run()`),
because SQLite runs in-process. Async drivers (Postgres, libSQL) return Promises
instead, and you `await` them — the query-building syntax stays the same.

## Using it from pages and actions

```tsx title="app/tasks/page.tsx (excerpt)"
import { connection } from "next/server";
import { desc } from "drizzle-orm";
import { db } from "@/db";
import { tasks } from "@/db/schema";

export default async function TasksPage() {
  await connection(); // live data: render per request
  const rows = db.select().from(tasks).orderBy(desc(tasks.createdAt)).all();
  return <ul>{rows.map((t) => <li key={t.id}>{t.title}</li>)}</ul>;
}
```

Why `connection()`? A database call isn't a request-time API, so without it Next.js
would happily run the query **at build time** and serve that snapshot forever. Either
mark the page dynamic like this, or cache the query deliberately and revalidate it after
writes (lesson 04).

## Production notes

- **SQLite in production** is a fine choice for a single server with a persistent disk.
  It is *not* suitable where each request may run on a different, short-lived instance
  (typical serverless hosting) — the file wouldn't be shared. There, use a network
  database (Postgres, MySQL) or a hosted SQLite-compatible service.
- **Connection limits.** Serverless functions can open many connections to Postgres at
  once; use a pooler or a driver designed for it.
- **File uploads** go to object storage (S3-compatible), with only the URL/key in the
  database — never into `public/` (Level 1 · 08).
- **Never expose raw rows to the client.** Map to the fields the UI needs.

## How It Actually Works

Drizzle builds an internal representation of your query from the chained calls, then
compiles it to a SQL string with `?` placeholders and a separate array of values:
`select ... from "tasks" where "tasks"."done" = ?` with `[0]`. The driver sends the
statement and values to SQLite separately, so a value can never change the query's
structure — that is what defeats SQL injection. For the Next.js side: the `db` module
is evaluated once per server process and cached by the module system, so every request
in that process shares one connection. In `next dev`, hot reloading can re-evaluate
modules and create extra connections; with network databases people often stash the
client on `globalThis` in development to avoid exhausting connection limits.

## Common mistakes

- **Querying at build time by accident** (see `connection()` above).
- **Importing `db` into a Client Component** — `server-only` catches it.
- **Building SQL with template strings** (`sql.raw(\`... ${input}\`)`). Use the query
  builder or `sql` tagged templates, which parameterise.
- **Running migrations on app start** in multi-instance deploys (races). Migrate once,
  as a release step.
- **Using SQLite on ephemeral serverless storage.**

## Exercise

1. Set up Drizzle + SQLite with the `tasks` schema and generate the first migration.
2. Add a `priority` column (`integer`, default 2). Run `generate` again and read the
   new migration before applying it.
3. Write a page listing tasks filtered by `?priority=1`. Log the SQL Drizzle runs
   (`drizzle(sqlite, { schema, logger: true })`) and confirm the value is a bound
   parameter, not part of the SQL text.
