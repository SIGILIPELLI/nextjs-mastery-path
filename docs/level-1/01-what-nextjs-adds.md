---
description: "What Next.js adds on top of React — routing, server rendering, data fetching, bundling and a production server — and when a plain React app is enough."
---

# 01 · What Next.js Adds to React

React is a library for describing UI as a function of state. It deliberately stops
there. It has no opinion about URLs, no way to fetch data before the page reaches the
browser, no build pipeline and no server. Every real React application therefore sits
inside *something* that supplies those pieces. Next.js is one such something — a
framework that bundles routing, rendering, data loading, code splitting and a server
into one set of conventions.

This lesson maps out exactly which problems Next.js takes over, so that later lessons
can each zoom in on one of them.

## The gaps a React-only app has to fill

Start from a single-page app built with a bundler such as Vite and nothing else:

| Concern | React alone | What you'd add yourself |
|---|---|---|
| Mapping URLs to screens | Nothing | A client router (React Router, TanStack Router) |
| First paint | Empty `<div id="root">` until JS runs | Server-side rendering setup, hydration wiring |
| Data loading | `useEffect` + `fetch` after mount | A data library, loaders, a backend API |
| Mutations | `fetch('/api/...')` from handlers | A separate API server, CORS, auth plumbing |
| SEO / social previews | One static `index.html` for all URLs | Per-route `<title>`, meta tags, sitemap |
| Code splitting | Manual `lazy()` | Route-based chunking |
| Images and fonts | Whatever you ship | Resizing, lazy loading, font subsetting |
| Production server | Static file host | Node server, caching headers, CDN rules |

None of these is impossible to do yourself. The cost is that every team assembles a
slightly different stack, and the pieces don't know about each other — your router
doesn't know which data a route needs, so it cannot start loading it before the code
arrives.

## What Next.js provides

Next.js answers each row with a convention:

- **Routing from the file system.** `app/blog/page.tsx` is the `/blog` page. Nested
  folders are nested routes; `layout.tsx` files wrap everything beneath them.
- **Rendering on the server by default.** Components in `app/` are *React Server
  Components*: they run on the server (or at build time) and send HTML plus a compact
  description of the UI to the browser. JavaScript is shipped only for components you
  mark with `"use client"`.
- **Data fetching where the UI is.** A Server Component can be `async` and `await` a
  database query or `fetch` call directly. No `useEffect`, no loading spinner for the
  initial content, no API layer needed just to read data.
- **Mutations without writing an API.** A function marked `"use server"` (a *Server
  Action*) can be passed to a `<form action={...}>`. Next.js creates the endpoint.
- **Static generation and caching.** Pages that don't depend on the request are rendered
  once at build time and served as files. Pages that do can be rendered per request,
  or partly static and partly streamed.
- **Built-in optimisations.** `next/image` resizes and lazy-loads images, `next/font`
  self-hosts fonts with no layout shift, and the Metadata API produces `<head>` tags.
- **A build tool and a server.** `next build` compiles everything (with Turbopack by
  default in version 16) and `next start` runs a production Node.js server. The output
  can also be deployed to platforms that understand Next.js, or exported as static
  files when you don't need a server.

## A concrete comparison

Here is a page that shows a list of products. In a client-only React app:

```tsx title="src/Products.tsx (client-only React)"
import { useEffect, useState } from "react";

type Product = { id: number; name: string };

export function Products() {
  const [products, setProducts] = useState<Product[] | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetch("/api/products")
      .then((r) => (r.ok ? r.json() : Promise.reject(r.statusText)))
      .then(setProducts)
      .catch((e) => setError(String(e)));
  }, []);

  if (error) return <p>Failed: {error}</p>;
  if (!products) return <p>Loading…</p>;
  return <ul>{products.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

The browser downloads JavaScript, runs it, renders "Loading…", *then* asks a separate
API for data, *then* renders the list. You also need that `/api/products` server.

The Next.js App Router version:

```tsx title="app/products/page.tsx"
import { db } from "@/db";

export default async function ProductsPage() {
  const products = await db.product.findMany(); // runs on the server
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>{p.name}</li>
      ))}
    </ul>
  );
}
```

(`db` stands for whatever data access you use; Level 2 lesson 08 sets up a real one.)
The query runs on the server, the HTML arrives already containing the list, and this
component adds **zero** bytes to the browser's JavaScript bundle. The loading and error
states move into sibling files (`loading.tsx`, `error.tsx`) covered in Level 2.

## When plain React is still the better choice

Next.js is not free. It adds a server (unless you use static export), a build step
with its own rules, and concepts — server/client boundaries, caching — that a pure
client app doesn't have. Plain React with Vite can be the better fit when:

- the app lives entirely behind a login and SEO and first paint don't matter much
  (internal dashboards, admin tools);
- you already have a separate backend and want the frontend to be static files only;
- you're embedding React widgets in a page that some other system renders.

Next.js pays off for public sites, content that must be indexable, apps where first
load matters on slow devices, and full-stack apps where one team owns both UI and data
access.

## App Router vs Pages Router

Next.js has two routers. The **Pages Router** (`pages/` directory) was the original,
built around `getServerSideProps` and `getStaticProps` functions that fetched data for
a whole page. The **App Router** (`app/` directory), introduced in version 13, is built
on React Server Components, nested layouts, streaming and Server Actions. Both are
still supported and can coexist in one project during a migration, but new features
land in the App Router and this course uses it exclusively. When you meet
`getServerSideProps` in an older codebase, think of it as "the Pages Router's way of
doing what an `async` Server Component does now".

## How It Actually Works

When a request for `/products` hits `next start`, roughly this happens:

1. **Route matching.** Next.js matches the URL against a manifest generated at build
   time from your `app/` folder. It finds `app/layout.tsx` (the root layout) and
   `app/products/page.tsx`.
2. **Server rendering.** React renders the tree on the server. Server Components run
   to completion (including their `await`s) and produce the **RSC payload** — a
   serialised description of the rendered UI in which Client Components appear as
   references ("render component X from chunk Y with these props") rather than code.
3. **HTML generation.** From that payload React also produces HTML, so the browser can
   paint before any JavaScript runs.
4. **Hydration.** The browser receives the HTML, the RSC payload (inlined in the page)
   and the JavaScript chunks for *only* the Client Components on the page. React
   attaches event handlers to the existing HTML.
5. **Client navigation.** When the user clicks a `<Link>`, the browser does not load a
   new HTML document. The Next.js client router fetches the RSC payload for the next
   route and React reconciles it into the page, preserving layouts that didn't change.

If the page doesn't depend on anything request-specific, steps 2–3 happen once during
`next build` and the result is stored as a file. That is "static rendering", and it is
the default whenever Next.js can prove it is safe.

## Common mistakes

- **Treating Next.js as "React with SSR bolted on".** Reaching for `useEffect` + `fetch`
  in every component throws away the main advantage. Fetch in Server Components first;
  use client fetching only for data that genuinely changes after load.
- **Marking the root layout `"use client"`.** This turns the whole tree into Client
  Components and ships all of it to the browser. Push `"use client"` down to the
  interactive leaves (lesson 05).
- **Mixing Pages Router tutorials into App Router code.** `getServerSideProps`,
  `next/router` and `_app.tsx` do nothing in `app/`. Check which router a snippet was
  written for.
- **Assuming "server rendered" means "rendered on every request".** Most App Router
  pages are rendered at build time. Lesson 09 and Level 2 lesson 04 explain what makes
  a page dynamic.

## Exercise

1. Take a small React app you have (or the product list above) and list, for each row
   of the "gaps" table, how that app currently solves it — or doesn't.
2. For each of these, decide whether you'd pick Next.js or a plain Vite React app, and
   write one sentence of justification: a public recipe site; an internal inventory
   screen used by five people; a marketing site with a signup form; a browser extension
   popup.
3. Predict, before reading lesson 05: in the Next.js `ProductsPage` above, would adding
   an `onClick` handler to the `<li>` work? Why or why not?
