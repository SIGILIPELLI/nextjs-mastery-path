---
description: "Fetch data in async Server Components: fetch and database calls, parallel vs sequential requests, deduplication, and what makes a page dynamic."
---

# 09 · Fetching Data in Server Components

In the App Router, the most common way to get data onto a page is also the simplest:
make the component `async` and `await` the data. No `useEffect`, no loading state for
the initial content, no separate API route.

## The basic pattern

```tsx title="app/users/page.tsx"
type User = { id: number; name: string; email: string };

export default async function UsersPage() {
  const res = await fetch("https://jsonplaceholder.typicode.com/users");
  if (!res.ok) throw new Error(`Failed to load users: ${res.status}`);
  const users: User[] = await res.json();

  return (
    <main>
      <h1>Users</h1>
      <ul>
        {users.map((u) => (
          <li key={u.id}>
            {u.name} — <a href={`mailto:${u.email}`}>{u.email}</a>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

(`jsonplaceholder.typicode.com` is a free public test API; any JSON endpoint works.)

Because this runs on the server:

- You can call **any** data source — a database client, the file system, an internal
  service on a private network — not just HTTP.
- Secrets used in the call (API keys, DB passwords) never reach the browser.
- Throwing an error is caught by the nearest `error.tsx` (Level 2 · 02).

Reading local files is just as direct:

```tsx title="app/changelog/page.tsx"
import { readFile } from "node:fs/promises";
import path from "node:path";

export default async function Changelog() {
  const text = await readFile(path.join(process.cwd(), "CHANGELOG.md"), "utf8");
  return <pre>{text}</pre>;
}
```

## Put data access in functions, not pages

Pages get messy if every one talks to the data source directly. Put access in a data
module and import it:

```ts title="lib/github.ts"
import "server-only";

export type Repo = { name: string; stargazers_count: number; html_url: string };

export async function getRepos(org: string): Promise<Repo[]> {
  const res = await fetch(`https://api.github.com/orgs/${org}/repos?per_page=5&sort=updated`, {
    headers: { Accept: "application/vnd.github+json" },
  });
  if (!res.ok) throw new Error(`GitHub responded ${res.status}`);
  return res.json();
}
```

`import "server-only"` guarantees this module never ends up in a client bundle (lesson
05). It also gives you one place to add auth headers, validation and error handling.

## Sequential vs parallel requests

Each `await` pauses the component. Two independent awaits in a row therefore take the
**sum** of their times:

```tsx
// Sequential: ~ time(A) + time(B)
const user = await getUser(id);
const posts = await getPostsFor(id);
```

If the second call doesn't need the first result, start both and wait for both:

```tsx title="app/profile/[id]/page.tsx"
import { getUser, getPostsFor } from "@/lib/data";

export default async function Profile({ params }: PageProps<"/profile/[id]">) {
  const { id } = await params;
  // Parallel: ~ max(time(A), time(B))
  const [user, posts] = await Promise.all([getUser(id), getPostsFor(id)]);
  return (
    <main>
      <h1>{user.name}</h1>
      <p>{posts.length} posts</p>
    </main>
  );
}
```

Sequential is correct only when there's a real dependency (for example, you need the
user's `teamId` to fetch the team). Level 3 · 02 shows a third option — streaming each
part independently with `<Suspense>` — which is often better than either.

## Deduplication: fetch where you need it

A layout, a page and `generateMetadata` often need the same record. You might think
you have to fetch once at the top and pass it down as props. You don't:

- Identical `fetch` calls (same URL and options) made during one render are
  **deduplicated** automatically — the request is sent once.
- For non-`fetch` data sources, wrap the function with React's `cache`:

```ts title="lib/posts.ts"
import "server-only";
import { cache } from "react";
import { db } from "@/db";

export const getPost = cache(async (slug: string) => {
  return db.query.posts.findFirst({ where: (p, { eq }) => eq(p.slug, slug) });
});
```

Now `generateMetadata` and the page can both call `getPost(slug)` and the query runs
once per request. React's `cache` is scoped to a single server request; it is **not** a
cross-request cache.

## Static or dynamic?

Whether this data is fetched once at build time or on every request is a separate
decision from *how* you fetch it. With the default configuration in Next.js 16, a
route is prerendered at build time unless it uses something only known at request time:

- `await cookies()`, `await headers()`, `await connection()`
- the `searchParams` prop of a page
- a `fetch` with `cache: "no-store"`, or segment config such as
  `export const dynamic = "force-dynamic"`

Notably, `fetch` responses are **not cached by default** since Next.js 15 — but a
`fetch` in a page that is otherwise static runs *during the build*, so its result is
baked into the HTML until you rebuild or revalidate. That surprises people: "my data
never updates" usually means "my page is static".

!!! note "Two caching models"
    Next.js 16 has an opt-in model called **Cache Components**
    (`cacheComponents: true`) in which you mark cached work explicitly with
    `"use cache"` and everything else is dynamic unless wrapped for prerendering. Level 2
    lesson 04 explains both models side by side. Until then, you only need: **static by
    default; request-time APIs make a route dynamic.**

To force fresh data on every request without cookies or headers:

```tsx title="app/status/page.tsx"
import { connection } from "next/server";

export default async function Status() {
  await connection(); // "render this per request"
  const res = await fetch("https://status.example.com/api/summary");
  const status = await res.json();
  return <p>Current status: {status.indicator}</p>;
}
```

## Worked example: a small GitHub dashboard

```tsx title="app/oss/page.tsx"
import { getRepos } from "@/lib/github";

export const metadata = { title: "Open source" };

export default async function OpenSource() {
  const [vercel, facebook] = await Promise.all([getRepos("vercel"), getRepos("facebook")]);
  return (
    <main>
      <h1>Recently updated repositories</h1>
      <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: "2rem" }}>
        {[{ org: "vercel", repos: vercel }, { org: "facebook", repos: facebook }].map(({ org, repos }) => (
          <section key={org}>
            <h2>{org}</h2>
            <ol>
              {repos.map((r) => (
                <li key={r.name}>
                  <a href={r.html_url}>{r.name}</a> ★ {r.stargazers_count}
                </li>
              ))}
            </ol>
          </section>
        ))}
      </div>
    </main>
  );
}
```

Build it and the route shows as `○` — the repositories were fetched during the build.
Add `await connection()` at the top and it becomes `ƒ`. The GitHub API is rate-limited
for unauthenticated requests, which is itself a good argument for *not* fetching on
every request unless you need to.

## How It Actually Works

An `async` Server Component is just a function returning a Promise of JSX. React's
server renderer awaits it; while it waits, it can keep rendering sibling subtrees, and
with Suspense boundaries it can even send finished parts of the page first. `fetch` on
the server is Node's standard `fetch`, extended by Next.js: during a render it keeps a
per-request memo table keyed by URL and options, which is how duplicates collapse into
one request. React's `cache()` does the same for any function, with a table that lives
exactly as long as the request.

During `next build`, Next.js tries to render every route. Request-time APIs such as
`cookies()` are instrumented: when one is called during prerendering, it signals that
the route can't be static, and Next.js marks it dynamic instead of baking the output.
That's why the route table can tell you — with no configuration — which pages depend on
the request.

## Common mistakes

- **Fetching your own Route Handler from a Server Component**
  (`fetch("http://localhost:3000/api/x")`). It's an extra network hop and breaks at
  build time when no server is running. Call the underlying function directly.
- **Accidental waterfalls** — a chain of awaits for independent data.
- **Expecting data to update on a static page.** Check the route table; revalidate or
  make it dynamic deliberately.
- **Swallowing errors** (`catch { return [] }`) so a broken API shows an empty page. Let
  it throw to an error boundary, or render an explicit error message.
- **Passing huge objects to Client Components.** Select only the fields the client needs.

## Exercise

1. Build the GitHub dashboard. Time the page with sequential awaits versus
   `Promise.all` by adding a `console.time` around the data calls (visible in the
   terminal running `next dev`).
2. Wrap a data function in `cache()` and call it from both `generateMetadata` and the
   page. Add a `console.log` inside it and confirm it logs once per request.
3. Create a page that shows the current server time. Build it and explain why the time
   never changes on reload. Then fix it with `connection()` and confirm the symbol in
   the route table changes.
