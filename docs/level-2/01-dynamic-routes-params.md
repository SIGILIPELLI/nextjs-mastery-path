---
description: "Dynamic segments in the App Router: [id], catch-all and optional catch-all routes, async params, generateStaticParams and dynamicParams."
---

# 01 · Dynamic Routes & Params

Most real URLs contain data: `/products/42`, `/blog/why-layouts-persist`,
`/docs/getting-started/install`. In the App Router you express that with a folder whose
name is in square brackets. One file then serves every matching URL.

## The three shapes

| Folder | Matches | `params` |
|---|---|---|
| `app/products/[id]/page.tsx` | `/products/42` | `{ id: "42" }` |
| `app/docs/[...path]/page.tsx` | `/docs/a`, `/docs/a/b/c` (not `/docs`) | `{ path: ["a","b","c"] }` |
| `app/shop/[[...filters]]/page.tsx` | `/shop`, `/shop/red`, `/shop/red/xl` | `{ filters: undefined }` or `["red","xl"]` |

Values are **always strings** (or string arrays). `/products/42` gives `"42"`, not `42`.

You can have several dynamic segments in one path:
`app/[org]/[repo]/issues/[number]/page.tsx` → `{ org, repo, number }`.

## `params` is a Promise

Since Next.js 15, `params` (and `searchParams`) are passed to pages and layouts as a
**Promise**. You `await` it in an `async` Server Component:

```tsx title="app/products/[id]/page.tsx"
import { notFound } from "next/navigation";
import { getProduct } from "@/lib/products";

export default async function ProductPage({ params }: PageProps<"/products/[id]">) {
  const { id } = await params;
  const productId = Number(id);
  if (!Number.isInteger(productId) || productId <= 0) notFound();

  const product = await getProduct(productId);
  if (!product) notFound();

  return (
    <main>
      <h1>{product.name}</h1>
      <p>${product.price.toFixed(2)}</p>
    </main>
  );
}
```

`PageProps<"/products/[id]">` is generated from your route tree, so `params` is typed
as `Promise<{ id: string }>` with no manual interface. If you rename the folder, the
type updates and TypeScript points at every place that's now wrong.

Why a Promise? It lets Next.js start rendering parts of the tree that *don't* need the
params before they're resolved, and it makes "this component depends on request data"
explicit. Next.js 14 passed plain objects; Next.js 15 allowed synchronous access with a
warning during a transition period; the version 16 upgrade guide treats synchronous
access as removed and offers a codemod (`next-async-request-api`).

In a **Client Component**, read params with the `useParams()` hook, or unwrap a
Promise passed from a Server Component with React's `use()`.

## Catch-all routes: a docs site

```tsx title="app/docs/[...path]/page.tsx"
import { notFound } from "next/navigation";
import { getDoc } from "@/lib/docs";

export default async function DocPage({ params }: PageProps<"/docs/[...path]">) {
  const { path } = await params;              // e.g. ["guides", "install"]
  const doc = await getDoc(path.join("/"));   // "guides/install"
  if (!doc) notFound();

  return (
    <article>
      <nav aria-label="Breadcrumb">
        {path.map((part, i) => (
          <span key={i}> / {decodeURIComponent(part)}</span>
        ))}
      </nav>
      <h1>{doc.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: doc.html }} />
    </article>
  );
}
```

Segments arrive URL-encoded for characters such as spaces; decode for display, and
validate before using them in a file path or query.

## Prerendering dynamic routes: `generateStaticParams`

By default a dynamic route is rendered when requested. If you know the valid values at
build time, list them and they're prerendered like static pages:

```tsx title="app/products/[id]/page.tsx (addition)"
import { listProductIds } from "@/lib/products";

export async function generateStaticParams() {
  const ids = await listProductIds();       // [1, 2, 3, ...]
  return ids.map((id) => ({ id: String(id) })); // values must be strings
}
```

What happens to an ID that wasn't in the list is controlled by `dynamicParams`:

```tsx
export const dynamicParams = true;  // default: render unknown values on demand
export const dynamicParams = false; // unknown values → 404
```

Common strategies:

- **Small, known set** (docs pages, blog posts): return all of them;
  `dynamicParams = false`.
- **Huge catalogue**: return the most popular few hundred; keep `dynamicParams = true`
  so the long tail renders on first request.
- **Nothing known in advance**: omit `generateStaticParams`.

For nested dynamic segments, a child's `generateStaticParams` receives the parent's
params:

```tsx title="app/[category]/[item]/page.tsx"
export async function generateStaticParams({ params }: { params: { category: string } }) {
  const items = await listItems(params.category);
  return items.map((item) => ({ item: item.slug }));
}
```

## Worked example: a product catalogue

```ts title="lib/products.ts"
import "server-only";

export type Product = { id: number; name: string; price: number; category: string };

const PRODUCTS: Product[] = [
  { id: 1, name: "Ceramic mug", price: 12, category: "kitchen" },
  { id: 2, name: "Linen apron", price: 28, category: "kitchen" },
  { id: 3, name: "Desk lamp", price: 45, category: "office" },
];

export async function listProductIds() {
  return PRODUCTS.map((p) => p.id);
}
export async function getProduct(id: number) {
  return PRODUCTS.find((p) => p.id === id) ?? null;
}
export async function listByCategory(category: string) {
  return PRODUCTS.filter((p) => p.category === category);
}
```

Add `generateStaticParams` returning all three IDs and run `npm run build`: you'll see
`/products/[id]` with `●` entries for `/products/1`, `/products/2` and `/products/3`.
Now visit `/products/abc` on `next start`: the integer check calls `notFound()`.

## How It Actually Works

At build time, Next.js compiles every dynamic folder into a pattern (`/products/[id]`
becomes something like a regular expression with a named group). Incoming URLs are
matched against static routes first, then dynamic ones, then catch-alls, so
`app/products/new/page.tsx` wins over `app/products/[id]/page.tsx` for `/products/new`.
The captured values populate `params`.

For `generateStaticParams`, the build calls the function, then renders the page once
per returned object, storing HTML and RSC payload under the concrete path. At runtime,
a request for a prerendered path is served from those files; a request for an unknown
path either invokes the page's render function (and, where the route is cacheable,
stores the result for next time) or returns 404 if `dynamicParams` is `false`.

## Common mistakes

- **Treating params as numbers.** They're strings; parse and validate.
- **Forgetting to `await params`.** In Next.js 16 `params.id` on the Promise is
  `undefined` and TypeScript flags it.
- **Returning numbers from `generateStaticParams`** (`{ id: 1 }`). Return strings.
- **Trusting params in file paths or raw SQL.** They are user input. Validate the shape
  and use parameterised queries.
- **Conflicting routes** such as `[id]` and `[slug]` as siblings. Next.js can't tell them
  apart and reports an error; use one name.
- **Prerendering millions of pages.** Builds become slow; prerender the popular subset.

## Exercise

1. Build `app/products/[id]/page.tsx` from the worked example, with
   `generateStaticParams` and `dynamicParams = false`. Confirm `/products/4` returns 404.
2. Add `app/shop/[[...filters]]/page.tsx` that shows "All products" at `/shop` and
   filters by category at `/shop/kitchen`. What does `/shop/kitchen/extra` do in your
   implementation? Decide what it *should* do and implement that.
3. Add `generateMetadata` to the product page so each product has its own `<title>`,
   and share the lookup with the page using React's `cache()`.
