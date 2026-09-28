---
description: "Next.js 16 caching explained: the previous model vs Cache Components, 'use cache', cacheLife, cacheTag, revalidatePath, revalidateTag, updateTag."
---

# 04 · Caching & Revalidation

Caching is where Next.js has changed most, and where most confusion comes from. Advice
written for version 14 is actively wrong for version 15, and version 16 introduced an
opt-in model that works differently again. This lesson gives you a mental model that
survives those changes, then the concrete APIs for Next.js 16.

## The durable mental model

Whatever the version, there are only a few questions:

1. **What is cached?** A data result (one query or `fetch`), or rendered UI (a page, a
   component).
2. **Where?** On the server (shared by all users), in the browser's router cache
   (per tab), or in a CDN in front of your server.
3. **For how long?** A lifetime, or until explicitly invalidated.
4. **Who invalidates it after a write?** Your code — with a path, a tag, or by
   time.

If you can answer these for each piece of data on a page, you understand its caching,
whatever the API names.

## A short history (so old tutorials make sense)

| Version | Default for `fetch` | Default for pages |
|---|---|---|
| 13–14 | Cached indefinitely (`force-cache`) unless opted out | Static when possible |
| 15 | **Not** cached by default | Static when possible; `GET` Route Handlers no longer cached by default |
| 16, default config | Same as 15 ("previous model" in the docs) | Same as 15 |
| 16 with `cacheComponents: true` | Nothing cached unless marked `"use cache"` | Partial prerendering: static shell + dynamic holes |

The official docs for version 16 now describe the Cache Components model first and
keep a separate guide, "Caching and Revalidating (Previous Model)", for apps that
haven't enabled it. Both are supported in 16; expect Cache Components to become the
recommended default over time.

## Model A: the default (previous) model

Without `cacheComponents`, the unit of caching is mostly the **route**:

- A route with no request-time APIs is prerendered at build time (`○`). Any `fetch` or
  DB call it makes runs during the build and the result is baked in.
- Time-based refresh (Incremental Static Regeneration): export `revalidate` from a page
  or layout, or set it per fetch.

```tsx title="app/prices/page.tsx"
export const revalidate = 300; // re-generate at most every 5 minutes

export default async function Prices() {
  const res = await fetch("https://api.example.com/prices");
  const prices: { sku: string; amount: number }[] = await res.json();
  return <ul>{prices.map((p) => <li key={p.sku}>{p.sku}: {p.amount}</li>)}</ul>;
}
```

- Per-request caching of a `fetch` result: `fetch(url, { cache: "force-cache" })` or
  `{ next: { revalidate: 60, tags: ["prices"] } }`.
- Non-`fetch` functions can be cached with `unstable_cache` (still prefixed
  `unstable_` in 16).
- Opting out: `export const dynamic = "force-dynamic"`, `await connection()`, or any
  request-time API.

## Model B: Cache Components (`cacheComponents: true`)

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = { cacheComponents: true };

export default nextConfig;
```

Now caching is **explicit and granular**:

- Nothing async is cached by default.
- `"use cache"` at the top of an async function or component caches its result. Its
  arguments (and captured variables) form the cache key.
- `cacheLife("hours")` (or `"minutes"`, `"days"`, `"max"`, a custom profile…) sets its
  lifetime. `cacheTag("products")` labels it for invalidation.
- Anything uncached or request-dependent must be inside a `<Suspense>` boundary, so the
  rest of the page can be prerendered as a static **shell** (this is Partial
  Prerendering, which Cache Components turns on).

Here is a page we built and compiled on Next.js 16.3.6:

```ts title="lib/catalog.ts"
import { cacheLife, cacheTag } from "next/cache";

export async function getProducts() {
  "use cache";
  cacheLife("hours");
  cacheTag("products");
  return [
    { id: 1, name: "Mug", price: 12 },
    { id: 2, name: "Shirt", price: 25 },
  ]; // imagine a database query here
}
```

```tsx title="app/page.tsx"
import { Suspense } from "react";
import { cookies } from "next/headers";
import { getProducts } from "@/lib/catalog";

async function Greeting() {
  const name = (await cookies()).get("name")?.value ?? "guest";
  return <p>Welcome back, {name}</p>;
}

export default async function Home() {
  const products = await getProducts();
  return (
    <main>
      <h1>Shop</h1>
      <Suspense fallback={<p>…</p>}>
        <Greeting />
      </Suspense>
      <ul>{products.map((p) => <li key={p.id}>{p.name}</li>)}</ul>
    </main>
  );
}
```

The build's route table:

```text
Route (app)      Revalidate  Expire
┌ ◐ /                    1h      1d
├ ○ /_not-found
└ ○ /products

○  (Static)             prerendered as static content
◐  (Partial Prerender)  prerendered as static HTML with dynamic server-streamed content
```

`/` is `◐`: the heading and cached product list are in the static shell (revalidated
per the `hours` profile), while the cookie-dependent greeting streams per request.

We then removed the `<Suspense>` around `<Greeting />` and rebuilt. The build **failed**,
with a message listing three possible fixes: wrap the access in
`<Suspense>`, cache it with `"use cache"`, or set `export const instant = false` to
allow a blocking route. Cache Components turns "this page accidentally became dynamic"
from a silent performance bug into a build error.

## Invalidating after a write

Both models share these functions from `next/cache`, called in Server Actions (and,
except `updateTag`, in Route Handlers):

| Function | Use when |
|---|---|
| `revalidatePath("/posts")` | You know which page(s) show the changed data |
| `revalidateTag("products", "max")` | Many pages share tagged data; stale-while-revalidate is fine |
| `updateTag("products")` | *Server Actions only*: the user who made the change must see it immediately ("read your own writes") |
| `refresh()` | Refresh the client router's view from a Server Action without invalidating a cache |

```ts title="app/admin/actions.ts"
"use server";

import { revalidateTag, updateTag } from "next/cache";

export async function editPrice(id: number, amount: number) {
  // ...write to the database
  updateTag("products"); // the editor sees the new price on the next render
}

export async function nightlyImport() {
  // ...bulk import
  revalidateTag("products", "max"); // visitors get fresh data on their next visit, served stale meanwhile
}
```

In Next.js 16 `revalidateTag` takes a second argument (a cache-life profile such as
`"max"`, or `{ expire: seconds }`); the one-argument form is deprecated. With `"max"`,
the next request is served the stale version while regeneration happens in the
background.

## The browser's router cache

Separately from the server, the client router keeps RSC payloads of visited and
prefetched routes so back/forward navigation is instant. A Server Action that
revalidates also refreshes this cache for the current user; `router.refresh()` forces
it from a Client Component. Other users' tabs are not notified — they see new data on
their next navigation that misses the cache, or you add real-time updates (Level 3 · 09).

## How It Actually Works

**Previous model:** during the build, rendering a static route stores its HTML and
RSC payload in the server's cache with a revalidate time. When a request arrives after
that time, the server returns the stored copy *and* triggers a background re-render
(stale-while-revalidate); the next visitor gets the new version. `revalidatePath` marks
a path's entry stale, so regeneration happens on the next request — nothing is rebuilt
at the moment you call it.

**Cache Components:** the compiler turns each `"use cache"` function into a wrapper
that serialises its arguments into a key, looks the key up in a cache handler
(in-memory by default for self-hosting; configurable), and on a miss runs the function
and stores the serialised result with its lifetime and tags. During the build Next.js
renders each route, letting cached and pure work complete, and stops at `<Suspense>`
boundaries whose children touch request data or uncached I/O; what completed becomes
the static shell. At request time, the shell is sent immediately and the holes are
rendered and streamed. Tags are just labels stored with entries — `revalidateTag`
marks every entry carrying the tag stale.

## Common mistakes

- **Copying version-14 advice** ("fetch is cached by default") into a 15/16 app.
- **"My data never updates"** — the page is static. Check the route table; add
  revalidation or make it dynamic deliberately.
- **Caching user-specific data under a shared key.** A `"use cache"` function reading a
  cookie is not allowed for this reason (`"use cache: private"` exists for per-user
  cases); in the previous model, never cache personalised fetches with `force-cache`.
- **Forgetting to revalidate after writes.**
- **Using the deprecated one-argument `revalidateTag(tag)`** in new code.
- **Mixing models**: segment configs like `export const dynamic` are not used with Cache
  Components; the migration guide maps each one to its replacement.

## Exercise

1. In a fresh app with the default model, build a page that shows `new Date().toISOString()`
   with `export const revalidate = 10`. Build, start, and reload repeatedly for 30
   seconds. Describe exactly when the timestamp changes and why the first reload after
   10 seconds still shows the old value.
2. In a second app, enable `cacheComponents`, reproduce the shop example, and observe the
   `◐` symbol. Remove the `<Suspense>` and read the build error in full.
3. Add a Server Action that calls `updateTag("products")` after changing a product,
   and verify the change is visible immediately to the user who made it.
