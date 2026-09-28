---
description: "Level 2 of Next.js: dynamic routes, loading and error UI, Server Actions, caching, Route Handlers, Proxy, authentication, databases and URL state."
---

# Level 2 · Intermediate <span class="level-badge">Full-stack</span>

Goal: turn a mostly static site into a full-stack application — one that reads and
writes a real database, validates forms on the server, protects pages behind a login,
exposes an API for other clients, and keeps its caches honest after data changes.

Level 1 covered what happens when a page is *read*. Level 2 is about what happens when
data *changes*: where the change is handled, how you validate it, and how every page
that showed the old data finds out.

Three ideas run through this level:

1. **The server is the source of truth, and mutations go through it.** Server Actions
   let a `<form>` call a server function directly. They are public HTTP endpoints in
   disguise, so every one validates its input and checks authorisation.
2. **Every cache needs an invalidation story.** Static pages and cached data are fast
   because they are old. When you write, you must say which cached results are now
   stale — with `revalidatePath`, `revalidateTag` or `updateTag`.
3. **Put checks where they can't be skipped.** Proxy is good for fast redirects, but
   authorisation belongs next to the data, in the code that reads or writes it.

## Modules

1. [Dynamic Routes & Params](01-dynamic-routes-params.md) — `[id]`, catch-all segments, async `params`, `generateStaticParams`
2. [Loading, Error & Not-Found UI](02-loading-error-not-found.md) — `loading.tsx`, `error.tsx`, `not-found.tsx` and status codes
3. [Server Actions & Forms](03-server-actions-forms.md) — `"use server"`, `useActionState`, validation, progressive enhancement
4. [Caching & Revalidation](04-caching-revalidation.md) — the previous model and Cache Components, `revalidatePath`, tags
5. [Route Handlers (API Endpoints)](05-route-handlers.md) — `route.ts`, the Web `Request`/`Response` API, webhooks
6. [Proxy (formerly Middleware)](06-proxy-middleware.md) — running code before routing: redirects, rewrites, headers
7. [Authentication Patterns](07-authentication.md) — sessions, cookies, Auth.js and a data access layer
8. [Databases with an ORM](08-databases-orm.md) — Drizzle ORM with SQLite, schemas, queries and migrations
9. [URL State & Search Params](09-url-state-search-params.md) — filters, sorting and pagination that live in the URL
10. [Project — Full-Stack Task Manager](10-project-crud-app.md) — CRUD with Drizzle + SQLite, validated Server Actions, and an end-to-end test

## Before you start

- Finish Level 1, or be comfortable with layouts, Server/Client Components and `async`
  data fetching in pages.
- Levels 2's database lessons use **SQLite** through `better-sqlite3`, which runs as a
  local file — no database server or account needed. On some systems npm asks you to
  approve the package's install script (it compiles or downloads a native binary);
  approve it for `better-sqlite3` only.
- A basic grasp of HTTP (methods, status codes, cookies, headers) helps a lot. The
  [REST API Mastery Path](https://sigilipelli.github.io/rest-api-mastery-path/) covers
  it if you need a refresher.

!!! note "Version-sensitive topics in this level"
    Two areas here changed significantly across recent versions: **caching** (lesson
    04) and **Middleware → Proxy** (lesson 06). Both lessons state which version each
    behaviour applies to. If you maintain a Next.js 14 or 15 app, read those lessons'
    "what changed" notes carefully before applying code from them.
