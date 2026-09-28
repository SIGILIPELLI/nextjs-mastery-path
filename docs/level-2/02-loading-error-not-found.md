---
description: "Add loading.tsx, error.tsx, not-found.tsx and global-error.tsx to App Router segments, and understand how streaming affects the HTTP status of error pages."
---

# 02 · Loading, Error & Not-Found UI

Every data-driven page has three states besides "success": still loading, failed, and
"this thing doesn't exist". In a client-only React app you handle them with state
variables in each component. In the App Router you handle them with three special
files per route segment — and the framework wires them into React's Suspense and error
boundaries for you.

## `loading.tsx`: instant feedback

```tsx title="app/dashboard/loading.tsx"
export default function Loading() {
  return (
    <div aria-busy="true" aria-live="polite">
      <p>Loading dashboard…</p>
      <div className="skeleton" style={{ height: 120 }} />
    </div>
  );
}
```

While `app/dashboard/page.tsx` is awaiting its data, this component is shown in its
place. The layouts above it are already on screen and interactive. Two effects:

- **Navigation feels instant.** When you click a `<Link>` to `/dashboard`, the loading
  UI is part of what was prefetched, so it appears immediately.
- **The server can stream.** The HTML up to the loading state is sent at once; the
  page's content follows in the same response when ready.

Prefer skeletons shaped like the real content over a spinner — they reduce layout shift
when the content arrives. For finer-grained control (several independently loading
sections in one page), use `<Suspense>` directly; Level 3 · 02 covers that.

## `error.tsx`: contain failures

```tsx title="app/dashboard/error.tsx"
"use client"; // error boundaries must be Client Components

import { useEffect } from "react";

export default function DashboardError({
  error,
  retry,
}: {
  error: Error & { digest?: string };
  retry: () => void;
}) {
  useEffect(() => {
    // report to your error tracker here
    console.error(error);
  }, [error]);

  return (
    <div role="alert">
      <h2>We couldn’t load your dashboard.</h2>
      <p>Reference: {error.digest ?? "n/a"}</p>
      <button onClick={() => retry()}>Try again</button>
    </div>
  );
}
```

If anything in the segment throws while rendering — a failed fetch, a bug — this
component replaces the page, while the layout and the rest of the app keep working.

- `retry()` re-fetches and re-renders the segment. It became a stable prop in Next.js
  16.3 (added as `unstable_retry` in 16.2). On earlier versions use `reset()`, which
  re-renders **without** re-fetching server data — so for a server-side failure you'd
  combine it with `router.refresh()`.
- In production, error messages from Server Components are **replaced with a generic
  message** so internals don't leak to users; `error.digest` is a hash you can match
  against server logs, where the full error is printed.

### Where the boundary sits

`error.tsx` wraps the segment's `page.tsx`, nested layouts, `loading.tsx` and
`not-found.tsx` — but **not the `layout.tsx` in the same folder**. An error in
`app/dashboard/layout.tsx` is caught by `app/error.tsx`, one level up. For the root
layout itself, add `app/global-error.tsx`, which must render its own `<html>` and
`<body>` because it replaces the root layout when active:

```tsx title="app/global-error.tsx"
"use client";

export default function GlobalError({ retry }: { error: Error & { digest?: string }; retry: () => void }) {
  return (
    <html lang="en">
      <body>
        <h1>Something went wrong</h1>
        <button onClick={() => retry()}>Reload</button>
      </body>
    </html>
  );
}
```

## `not-found.tsx` and `notFound()`

```tsx title="app/products/[id]/not-found.tsx"
import Link from "next/link";

export default function ProductNotFound() {
  return (
    <main>
      <h1>Product not found</h1>
      <p>It may have been discontinued. <Link href="/products">Browse all products</Link>.</p>
    </main>
  );
}
```

Call `notFound()` from `next/navigation` in a page, layout, `generateMetadata` or
Server Action, and the nearest `not-found.tsx` renders. Like `redirect()`, it throws
internally, so code after it doesn't run (and TypeScript knows the value can't be
`null` afterwards).

`app/not-found.tsx` at the root also handles URLs that match no route at all.

## The status-code subtlety: streaming vs 404

Here's something you only discover by testing. We built a test app with
`app/blog/[slug]/page.tsx` (calling `notFound()` for unknown slugs, with the default
`dynamicParams = true`) **and** an `app/blog/loading.tsx`. Then:

```bash
curl -s -o /dev/null -w "%{http_code}\n" localhost:3111/blog/nope
curl -s -o /dev/null -w "%{http_code}\n" localhost:3111/nothing-here
```

Observed on Next.js 16.3.6:

```text
200
404
```

The unknown blog slug returned **200**, while a URL matching no route returned 404.
The body of the 200 response contained the not-found UI and a
`<meta name="robots" content="noindex">` tag.

Why? The loading boundary let Next.js start streaming the response — status line and
headers included — before the page component ran. By the time `notFound()` was
called, the 200 had already been sent. Next.js compensates by injecting `noindex` so
search engines don't index the page, but API clients and uptime monitors that only look
at status codes will be misled.

When the correct status matters, resolve "does this exist?" **before** anything can
stream: don't wrap that route in a `loading.tsx` above the check, use
`dynamicParams = false` with a known list (we saw a clean 404 with that setup in the
Level 1 blog project), or check existence in Proxy for routes that must 404 strictly.

## Worked example: a resilient product page

```text
app/products/
├── page.tsx
├── error.tsx          ← catches failures in the list and in [id]
└── [id]/
    ├── page.tsx
    ├── loading.tsx    ← skeleton while one product loads
    └── not-found.tsx  ← friendly 404 for unknown ids
```

```tsx title="app/products/[id]/page.tsx"
import { notFound } from "next/navigation";
import { getProduct } from "@/lib/products";

export default async function ProductPage({ params }: PageProps<"/products/[id]">) {
  const { id } = await params;
  const product = await getProduct(Number(id));
  if (!product) notFound();
  if (product.discontinued) throw new Error(`Product ${id} is discontinued`); // demo: goes to error.tsx
  return <h1>{product.name}</h1>;
}
```

Walk through the states: a slow `getProduct` shows the skeleton; an unknown ID shows
the product-specific not-found page; a thrown error shows `app/products/error.tsx` with
a retry button, and the site header in the root layout stays usable throughout.

## How It Actually Works

For each segment, Next.js composes the special files into a fixed element hierarchy:

```tsx
<Layout>
  <Template>
    <ErrorBoundary fallback={<Error />}>
      <Suspense fallback={<Loading />}>
        <NotFoundBoundary fallback={<NotFound />}>
          <Page />
        </NotFoundBoundary>
      </Suspense>
    </ErrorBoundary>
  </Template>
</Layout>
```

That ordering explains every rule above: the error boundary is *inside* the layout (so
it can't catch the layout's own errors), the loading state is inside the error boundary
(so a failure while loading is still caught), and `notFound()` throws a special error
that the not-found boundary recognises. On the server, React's streaming renderer sends
the shell with `<Loading />` in place of anything still suspended, then later sends the
finished content with a small inline script that swaps it into position. Once the first
bytes are flushed, the HTTP status can no longer change — the root of the 200-vs-404
behaviour.

## Common mistakes

- **Forgetting `"use client"` in `error.tsx`.** Error boundaries are a client concept;
  the build will complain.
- **Expecting `error.tsx` to catch errors in the same folder's layout.** Put a boundary
  one level up or use `global-error.tsx`.
- **Showing `error.message` to users in development and assuming the same in
  production.** Production hides Server Component messages. Log by `digest`.
- **Catching `notFound()` or `redirect()` in a `try/catch`.** They work by throwing.
- **Relying on the status code of streamed 404s.** Test it with `curl`.
- **A single spinner for the whole app** in `app/loading.tsx`. Place loading UI as close
  as possible to the slow data.

## Exercise

1. Add `loading.tsx`, `error.tsx` and `not-found.tsx` to a dynamic route in your
   project. Use `await new Promise(r => setTimeout(r, 2000))` in the page to see the
   loading UI, and a query flag (`?fail=1`) to throw.
2. Build and start the app, then use `curl -w "%{http_code}"` to record the status for
   an unknown ID with and without the `loading.tsx` file. Explain what you see.
3. Add `app/global-error.tsx` and deliberately throw in the root layout (temporarily).
   Confirm your global error UI appears in a production build.
