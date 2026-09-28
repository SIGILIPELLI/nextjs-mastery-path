---
description: "Server vs Client Components in Next.js: where each runs, the 'use client' boundary, what props can cross it, and composing both to ship less JS."
---

# 05 · Server vs Client Components

Every component in the App Router is a **Server Component** unless a file says
otherwise. This one default changes how you structure an app more than any other
Next.js feature, so it's worth understanding precisely.

## What each kind can do

| | Server Component | Client Component |
|---|---|---|
| Runs | On the server (at build or request time) | On the server for the initial HTML, then in the browser |
| `async` / `await` data | Yes | No (not as a component) |
| Access DB, file system, secrets | Yes | No — code is shipped to the browser |
| `useState`, `useEffect`, `useReducer` | No | Yes |
| Event handlers (`onClick`, `onChange`) | No | Yes |
| Browser APIs (`window`, `localStorage`) | No | Yes (inside effects / handlers) |
| Adds its code to the JS bundle | No | Yes |

A useful rule of thumb: **fetch and compose on the server; interact on the client.**

## The `"use client"` directive

Put `"use client"` at the very top of a file (before imports) to declare that file an
entry point into client code:

```tsx title="app/_components/LikeButton.tsx"
"use client";

import { useState } from "react";

export function LikeButton({ initialLikes }: { initialLikes: number }) {
  const [likes, setLikes] = useState(initialLikes);
  const [liked, setLiked] = useState(false);
  return (
    <button
      aria-pressed={liked}
      onClick={() => {
        setLiked(!liked);
        setLikes((n) => n + (liked ? -1 : 1));
      }}
    >
      ♥ {likes}
    </button>
  );
}
```

```tsx title="app/posts/[id]/page.tsx"
import { LikeButton } from "@/app/_components/LikeButton";
import { getPost } from "@/lib/posts";

export default async function PostPage({ params }: PageProps<"/posts/[id]">) {
  const { id } = await params;
  const post = await getPost(id);            // server-only work
  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.body}</div>
      <LikeButton initialLikes={post.likes} /> {/* the only JS shipped */}
    </article>
  );
}
```

The directive marks a **boundary**, not a single component. Everything the client file
imports becomes client code too. You don't need `"use client"` in every interactive
file — only at the point where server code first imports client code.

!!! warning "Client Components still render on the server first"
    "Client" does not mean "browser only". For the first page load a Client Component
    is also rendered to HTML on the server, then *hydrated* in the browser. So
    `window.localStorage` at the top level of a Client Component throws during that
    server render. Access browser APIs inside `useEffect` or event handlers.

## What can cross the boundary

When a Server Component renders a Client Component, the props are **serialised** and
sent over the network. So props must be serialisable:

- ✅ strings, numbers, booleans, `null`, `undefined`, plain objects and arrays, `Date`,
  `Map`, `Set`, `BigInt`, typed arrays, Promises (resolved on the client with `use`)
- ✅ JSX / React elements (including other Server Components) — as `children` or any prop
- ✅ Server Actions (functions marked `"use server"` — Level 2 · 03)
- ❌ ordinary functions (`onClick={() => …}` from a Server Component)
- ❌ class instances with methods, database connections, symbols not registered globally

Trying to pass a normal function fails with an error explaining that functions cannot be
passed to Client Components unless they are Server Actions.

## Composition: pass Server Components *through* Client Components

A Client Component can't `import` a Server Component (the import would make it client
code). But it can **receive** one as `children` or another prop:

```tsx title="app/_components/Collapsible.tsx"
"use client";

import { useState } from "react";

export function Collapsible({ title, children }: { title: string; children: React.ReactNode }) {
  const [open, setOpen] = useState(false);
  return (
    <section>
      <button aria-expanded={open} onClick={() => setOpen(!open)}>{title}</button>
      {open && <div>{children}</div>}
    </section>
  );
}
```

```tsx title="app/faq/page.tsx"
import { Collapsible } from "@/app/_components/Collapsible";
import { getFaqAnswer } from "@/lib/faq"; // reads from a DB — server only

async function Answer({ id }: { id: string }) {
  const text = await getFaqAnswer(id);
  return <p>{text}</p>;
}

export default function FaqPage() {
  return (
    <Collapsible title="How do refunds work?">
      <Answer id="refunds" /> {/* rendered on the server, passed in as children */}
    </Collapsible>
  );
}
```

The collapsible's toggle logic ships to the browser; the answer's data-fetching code
does not. This "donut" pattern — client shell, server filling — is how you keep the
client bundle small without giving up interactivity.

## Keeping server code out of the client

It's easy to import a module that reads secrets into a client file by accident. The
`server-only` package turns that mistake into a build error:

```bash
npm install server-only
```

```ts title="lib/secrets.ts"
import "server-only";

export function paymentsApiKey() {
  const key = process.env.PAYMENTS_API_KEY;
  if (!key) throw new Error("PAYMENTS_API_KEY is not set");
  return key;
}
```

If any Client Component's import graph reaches `lib/secrets.ts`, the build fails.
There is a matching `client-only` package for browser-only modules.

Environment variables add a second guard: only variables prefixed `NEXT_PUBLIC_` are
inlined into client bundles. `process.env.PAYMENTS_API_KEY` in client code evaluates to
`undefined`.

## Worked example: turning an all-client page into a mixed one

Before (everything client-side, because the author needed one button):

```tsx title="app/products/page.tsx (before)"
"use client";
import { useEffect, useState } from "react";
// ...fetches /api/products in useEffect, renders list, has an "Add to cart" button
```

After:

```tsx title="app/products/page.tsx"
import { AddToCart } from "./AddToCart";
import { listProducts } from "@/lib/products";

export default async function ProductsPage() {
  const products = await listProducts();
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>
          {p.name} — ${p.price.toFixed(2)} <AddToCart productId={p.id} />
        </li>
      ))}
    </ul>
  );
}
```

```tsx title="app/products/AddToCart.tsx"
"use client";

import { useState } from "react";

export function AddToCart({ productId }: { productId: number }) {
  const [added, setAdded] = useState(false);
  return (
    <button disabled={added} onClick={() => setAdded(true)} data-id={productId}>
      {added ? "Added" : "Add to cart"}
    </button>
  );
}
```

The list renders with data on first paint, the `/api/products` endpoint is no longer
needed, and the client bundle holds only `AddToCart`.

## How It Actually Works

The bundler builds **two module graphs**: a server graph and a client graph. Starting
from each page, it follows imports in the server graph until it reaches a file marked
`"use client"`. At that point it stops treating the module as server code and instead
records a **client reference** — an ID pointing to the module in the client graph, where
it is bundled as a normal JavaScript chunk.

When React renders on the server, Server Components are called like functions and
disappear into their output. Client Components are not called in the RSC render;
they're emitted into the RSC payload as `{ reference to chunk, props }`. That's why
props must be serialisable — they are literally written into that payload. A separate
server-side render of the same tree (which *does* execute Client Components) produces
the initial HTML. In the browser, React reads the payload, loads the referenced chunks,
and hydrates. Server Components never exist in the browser at all, which is why they
can't have state or effects: there is nothing on the client to hold the state.

## Common mistakes

- **Adding `"use client"` to fix an error without understanding it.** Often the right
  fix is extracting the interactive part into a small client file.
- **Putting `"use client"` in a shared barrel file** (`components/index.ts`) that
  server code imports. Now everything exported from it is client code.
- **Reading `window` during render in a Client Component** — it runs on the server
  first. Use `useEffect`.
- **Passing a whole database record with non-serialisable fields** to a Client
  Component. Map it to a plain object with just the fields the client needs (this also
  avoids leaking private columns).
- **Using React Context in a Server Component.** Context providers are client-only;
  render the provider in a Client Component and wrap `children` with it.

## Exercise

1. Build a page that lists five quotes from an array in a Server Component and adds a
   "Copy" button per quote that uses `navigator.clipboard.writeText` (a Client
   Component). Ensure the array is *not* in the client bundle.
2. Try passing `onCopied={() => console.log("done")}` from the Server Component to the
   button. Read the error message and explain it in your own words.
3. Create `lib/db-config.ts` with `import "server-only"` and import it from your Client
   Component. Run `npm run build` and note the error. Then fix the design.
