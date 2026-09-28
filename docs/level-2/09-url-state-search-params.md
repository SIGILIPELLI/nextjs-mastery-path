---
description: "Keep filters, sorting, search and pagination in the URL with searchParams, useSearchParams and router.replace: shareable and server-rendered."
---

# 09 · URL State & Search Params

Where should a list's current filter, sort order and page number live? The common
instinct is `useState`. The better answer, most of the time, is **the URL**:
`/products?category=kitchen&sort=price&page=2`. Then the view is shareable, survives a
refresh, works with the back button, and — in Next.js — can be rendered on the server.

## Reading search params on the server

Pages receive `searchParams` as a Promise:

```tsx title="app/products/page.tsx"
import Link from "next/link";
import { listProducts } from "@/lib/products";

const SORTS = ["name", "price"] as const;
type Sort = (typeof SORTS)[number];
const PAGE_SIZE = 10;

export default async function ProductsPage({ searchParams }: PageProps<"/products">) {
  const sp = await searchParams;
  const category = typeof sp.category === "string" ? sp.category : undefined;
  const sort: Sort = SORTS.includes(sp.sort as Sort) ? (sp.sort as Sort) : "name";
  const page = Math.max(1, Number.parseInt(String(sp.page ?? "1"), 10) || 1);

  const { items, total } = await listProducts({ category, sort, offset: (page - 1) * PAGE_SIZE, limit: PAGE_SIZE });
  const pages = Math.max(1, Math.ceil(total / PAGE_SIZE));

  const withParams = (patch: Record<string, string | undefined>) => {
    const next = new URLSearchParams();
    const merged = { category, sort, page: String(page), ...patch };
    for (const [k, v] of Object.entries(merged)) if (v) next.set(k, v);
    return `/products?${next}`;
  };

  return (
    <main>
      <p>
        Sort: <Link href={withParams({ sort: "name", page: "1" })}>name</Link> ·{" "}
        <Link href={withParams({ sort: "price", page: "1" })}>price</Link>
      </p>
      <ul>{items.map((p) => <li key={p.id}>{p.name} — ${p.price}</li>)}</ul>
      <nav aria-label="Pagination">
        {page > 1 && <Link href={withParams({ page: String(page - 1) })}>← Previous</Link>}{" "}
        Page {page} of {pages}{" "}
        {page < pages && <Link href={withParams({ page: String(page + 1) })}>Next →</Link>}
      </nav>
    </main>
  );
}
```

Things to notice:

- Each value can be `string`, `string[]` (for `?tag=a&tag=b`) or `undefined`. Validate
  and default every one — this is user input.
- Invalid values (like `sort=drop table`) fall back to a safe default instead of
  reaching the query.
- Sorting and pagination are plain `<Link>`s: they work without JavaScript and get
  prefetched.
- Using `searchParams` makes the page **dynamic** — each combination is rendered on
  request (or, under Cache Components, the part that reads them streams inside a
  Suspense boundary).

## Updating the URL from a Client Component

For a live search box you want to update the URL as the user types, without a full
navigation per keystroke:

```tsx title="app/products/SearchBox.tsx"
"use client";

import { usePathname, useRouter, useSearchParams } from "next/navigation";
import { useEffect, useState, useTransition } from "react";

export function SearchBox() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();
  const [value, setValue] = useState(searchParams.get("q") ?? "");
  const [pending, startTransition] = useTransition();

  useEffect(() => {
    const id = setTimeout(() => {
      const next = new URLSearchParams(searchParams);
      if (value) next.set("q", value);
      else next.delete("q");
      next.delete("page"); // a new query starts at page 1
      startTransition(() => router.replace(`${pathname}?${next}`, { scroll: false }));
    }, 300); // debounce
    return () => clearTimeout(id);
  }, [value]); // eslint-disable-line react-hooks/exhaustive-deps

  return (
    <div>
      <label htmlFor="q">Search products</label>{" "}
      <input id="q" type="search" value={value} onChange={(e) => setValue(e.target.value)} />
      {pending && <span aria-live="polite"> Searching…</span>}
    </div>
  );
}
```

- `router.replace` (not `push`) avoids a history entry per keystroke.
- `scroll: false` keeps the page from jumping to the top.
- The transition keeps the current results visible while the server renders new ones.

### The Suspense requirement

`useSearchParams()` in a Client Component on a statically rendered route forces
everything up to the nearest `<Suspense>` boundary to render on the client, and the
production build complains if there's no boundary. Wrap the component:

```tsx
import { Suspense } from "react";
import { SearchBox } from "./SearchBox";

<Suspense fallback={<div style={{ height: 32 }} />}>
  <SearchBox />
</Suspense>
```

## A plain HTML alternative

A `GET` form writes its fields into the query string with no JavaScript at all:

```tsx
<form action="/products">
  <label htmlFor="cat">Category</label>
  <select id="cat" name="category" defaultValue={category}>
    <option value="">All</option>
    <option value="kitchen">Kitchen</option>
    <option value="office">Office</option>
  </select>
  <button>Filter</button>
</form>
```

Next.js also provides a `<Form>` component (`next/form`) that behaves like this but
performs a client-side navigation and can prefetch the target once hydrated.

## What belongs in the URL?

| In the URL | Not in the URL |
|---|---|
| Filters, sort, page, search query | Form drafts mid-edit |
| Selected tab, open item ID | Hover/animation state |
| Map viewport, date range | Secrets or personal data |

A useful test: "If I send this link to a colleague, should they see the same thing?"

## How It Actually Works

On the server, `searchParams` is parsed from the request URL and passed into the page;
because it can only be known per request, touching it marks the route dynamic (Level 1
· 09). On the client, `useSearchParams` subscribes to the router's current URL.
`router.replace` updates `history` and fetches the RSC payload for the new URL; the
server re-renders the page with the new `searchParams` and React reconciles the result
into the existing tree, preserving client state such as the input's focus and its local
`value`. The search box doesn't need to "pass" the query to the list — the URL is the
shared state, and the server re-derives everything from it.

## Common mistakes

- **Duplicating URL state in `useState`** and letting the two drift. Derive from the URL.
- **Not validating params** — using `sp.page` directly in an offset.
- **`router.push` on every keystroke**, filling history.
- **Missing Suspense around `useSearchParams`** on static routes.
- **Forgetting to reset `page`** when filters change, leaving users on an empty page 7.

## Exercise

1. Build the products page with category filter, sort and pagination entirely with
   `<Link>` and a `GET` form. Verify it works with JavaScript disabled.
2. Add the debounced `SearchBox`. In the Network panel, count requests while typing a
   ten-letter word with and without the debounce.
3. Add a multi-select tag filter (`?tag=a&tag=b`) and handle the `string[]` case.
