---
description: "Navigate between App Router pages with Link, prefetching and client transitions, highlight active links, and navigate from code with useRouter."
---

# 04 · Pages & Navigation with Link

A plain `<a href="/about">` works in Next.js — it just throws away most of what the
framework can do. The `<Link>` component turns a click into a *client-side
transition*: no full document reload, shared layouts kept alive, and the next page's
data often already downloaded before you click.

## `<Link>` basics

```tsx title="app/page.tsx"
import Link from "next/link";

export default function Home() {
  return (
    <main>
      <h1>Home</h1>
      <ul>
        <li><Link href="/about">About</Link></li>
        <li><Link href="/blog/routing">A blog post</Link></li>
        <li><Link href={{ pathname: "/search", query: { q: "layouts" } }}>Search for layouts</Link></li>
        <li><Link href="/about#team">Jump to the team section</Link></li>
      </ul>
    </main>
  );
}
```

`<Link>` renders a real `<a>` element, so it is crawlable, works with middle-click and
"open in new tab", and degrades gracefully if JavaScript fails. Any extra props
(`className`, `aria-current`, `target`) pass through to the `<a>`.

Useful props:

| Prop | Effect |
|---|---|
| `href` | Path string or URL object. Required. |
| `replace` | Replace the current history entry instead of pushing a new one. |
| `scroll` | Set `false` to keep the scroll position after navigation. |
| `prefetch` | `"auto"`/`null` (default), `true`, or `false` — see below. |

For links to *other* sites, use a normal `<a>`; `<Link>` is for routes in your app.

## Prefetching

In production, when a `<Link>` scrolls into the viewport, Next.js prefetches the
destination in the background. What it fetches depends on the route (as documented for
Next.js 16):

- **Static route** — the whole route, including its data. The click is effectively
  instant.
- **Dynamic route** — the part of the route down to the nearest `loading.tsx`
  boundary. The click shows that loading UI immediately while the rest streams in.
- `prefetch={false}` — nothing until the click. Useful for huge lists of rarely
  followed links.

Prefetching is **disabled in `next dev`**, which is one more reason to judge navigation
speed with `next build && next start`.

## Highlighting the active link

Layouts don't re-render on navigation and Server Components can't read the current
path, so active-link styling belongs in a small Client Component:

```tsx title="app/_components/NavLink.tsx"
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";

export function NavLink({ href, children }: { href: string; children: React.ReactNode }) {
  const pathname = usePathname();
  const active = href === "/" ? pathname === "/" : pathname.startsWith(href);
  return (
    <Link href={href} aria-current={active ? "page" : undefined} className={active ? "active" : undefined}>
      {children}
    </Link>
  );
}
```

```tsx title="app/layout.tsx (nav excerpt)"
import { NavLink } from "./_components/NavLink";

// inside <body>:
<nav>
  <NavLink href="/">Home</NavLink>
  <NavLink href="/blog">Blog</NavLink>
  <NavLink href="/about">About</NavLink>
</nav>
```

The layout stays a Server Component; only the tiny `NavLink` ships JavaScript.
`aria-current="page"` also tells screen readers which link is current.

## Showing that a navigation is in progress

If a destination is dynamic and has no `loading.tsx`, a click can appear to do nothing
for a moment. `useLinkStatus` (from `next/link`) lets a component *inside* a `<Link>`
know that its navigation is pending:

```tsx title="app/_components/PendingDot.tsx"
"use client";

import { useLinkStatus } from "next/link";

export function PendingDot() {
  const { pending } = useLinkStatus();
  return <span aria-hidden="true" style={{ opacity: pending ? 1 : 0 }}> ⏳</span>;
}
```

```tsx
<Link href="/reports">Reports<PendingDot /></Link>
```

Prefer fixing the cause — add a `loading.tsx` (Level 2 · 02) — and use this as polish.

## Navigating from code

In a Client Component, `useRouter` from **`next/navigation`** (not `next/router`, which
is the Pages Router) gives you imperative navigation:

```tsx title="app/_components/SearchBox.tsx"
"use client";

import { useRouter } from "next/navigation";
import { useState } from "react";

export function SearchBox() {
  const router = useRouter();
  const [q, setQ] = useState("");
  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        router.push(`/search?q=${encodeURIComponent(q)}`);
      }}
    >
      <label htmlFor="q">Search</label>
      <input id="q" value={q} onChange={(e) => setQ(e.target.value)} />
      <button>Go</button>
    </form>
  );
}
```

Methods: `push(href)`, `replace(href)`, `back()`, `forward()`, `refresh()` (re-fetch
the current route's Server Components without losing client state), and
`prefetch(href)`.

On the **server** — in a Server Component, Server Action or Route Handler — use
`redirect()` from `next/navigation` instead:

```tsx title="app/dashboard/page.tsx"
import { redirect } from "next/navigation";
import { getSession } from "@/lib/session"; // your own helper

export default async function Dashboard() {
  const session = await getSession();
  if (!session) redirect("/login");
  return <h1>Welcome back</h1>;
}
```

`redirect()` works by throwing a special error that Next.js catches, so code after it
never runs — and you must not wrap it in a `try/catch` that swallows it.
`permanentRedirect()` does the same with a permanent status for moved content.

## How It Actually Works

A `<Link>` renders an `<a>` and attaches a click handler. On a normal left-click
without modifier keys, the handler calls `preventDefault()` and asks the client router
to navigate. The router:

1. Looks in its **client-side cache** for the destination's RSC payload (filled by an
   earlier prefetch or visit).
2. If absent or incomplete, requests it from the server. This is an ordinary HTTP
   request to the same URL with special request headers that ask for the RSC payload
   instead of HTML. The server renders only the segments that differ from the current
   page.
3. Updates the browser history with `history.pushState`, and hands the payload to React,
   which reconciles it into the existing tree inside a transition — so the old UI stays
   interactive until the new one is ready (or a loading boundary is shown).
4. Scrolls to the top of the new segment (unless `scroll={false}`) and moves focus
   for accessibility; Next.js also announces the new page title through a hidden live
   region.

Prefetching uses an `IntersectionObserver` on each `<Link>`; entering the viewport
triggers step 2 early. Because the router merges payloads per segment, a prefetch of
`/blog/a` and a later visit to `/blog/b` can share the cached `blog` layout.

## Common mistakes

- **Importing `useRouter` from `next/router`.** In the App Router it throws; use
  `next/navigation`.
- **Using `router.push` for ordinary links.** You lose prefetching, crawlability and
  open-in-new-tab. Use `<Link>`; reserve `router.push` for navigation after logic.
- **Calling `redirect()` inside `try { } catch { }`.** The catch swallows the redirect.
  Call it outside the try, or rethrow.
- **Concluding "navigation is slow" from dev mode.** No prefetching happens there.
- **Wrapping `<Link>` around another `<a>`.** Since Next.js 13 `<Link>` renders the
  anchor itself; nesting produces invalid HTML.

## Exercise

1. Add a nav bar with `NavLink` to your project and verify with the browser's
   accessibility inspector that the current link has `aria-current="page"`.
2. Build and start the app (`npm run build && npm start`), open the Network panel, and
   scroll a page with several links. Identify the prefetch requests. Repeat with
   `prefetch={false}` on one link and confirm it no longer appears until clicked.
3. Write a Server Component page `/go` that reads a `to` search parameter and calls
   `redirect()` to `/blog/<to>`. (You'll learn to read `searchParams` properly in
   Level 2; for now, accept `searchParams: Promise<{ to?: string }>` as a prop and
   `await` it.) What should happen when `to` is missing?
