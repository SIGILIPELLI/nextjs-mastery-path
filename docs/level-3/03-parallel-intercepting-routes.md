---
description: "Parallel routes (@slots) and intercepting routes in Next.js: default.tsx, and a modal that has its own URL and becomes a full page on refresh."
---

# 03 · Parallel & Intercepting Routes

Two advanced routing features solve problems that are awkward with plain nested
layouts: showing **several independently routed sections** in one layout, and showing a
route **as an overlay** on top of the current page while still giving it a real URL.
Combined, they give you the "photo opens in a modal, but refresh shows the full photo
page" behaviour familiar from social sites.

## Parallel routes: named slots

A folder starting with `@` is a **slot**. It doesn't add a URL segment; instead, its
content is passed to the parent layout as a prop with the slot's name:

```text
app/dashboard/
├── layout.tsx
├── page.tsx
├── @activity/
│   ├── page.tsx
│   └── default.tsx
└── @stats/
    ├── page.tsx
    └── default.tsx
```

```tsx title="app/dashboard/layout.tsx"
export default function DashboardLayout({ children, activity, stats }: LayoutProps<"/dashboard">) {
  return (
    <div style={{ display: "grid", gridTemplateColumns: "2fr 1fr", gap: "1.5rem" }}>
      <section>{children}</section>
      <aside>
        {stats}
        {activity}
      </aside>
    </div>
  );
}
```

`LayoutProps<"/dashboard">` knows about the slots because they come from the file
system. Each slot can have its own `loading.tsx` and `error.tsx`, so the activity feed
failing doesn't affect the stats panel — a clean way to build dashboards.

Slots can also be rendered **conditionally**:

```tsx
export default async function Layout({ children, admin, user }: LayoutProps<"/panel">) {
  const session = await getSession();
  return <>{children}{session?.role === "admin" ? admin : user}</>;
}
```

### `default.tsx` is required

When you navigate to a sub-URL (say `/dashboard/settings`), Next.js needs to know what
each slot should show if it has no matching page for that URL. On a client-side
navigation it keeps the slot's previous content; on a full page load it can't know that,
so it renders the slot's `default.tsx`. Since Next.js 16, **every slot must have a
`default.tsx`**, or the build fails. Return `null` to render nothing, or call
`notFound()`.

## Intercepting routes

An intercepting route renders a different route's URL *in the context of the current
layout* — but only for client-side navigations. The folder prefix says how far up to
look for the route being intercepted:

| Prefix | Intercepts a route… |
|---|---|
| `(.)` | at the same level |
| `(..)` | one level up |
| `(..)(..)` | two levels up |
| `(...)` | from the app root |

These are based on **route segments**, not file-system folders — slots like `@modal`
don't count as a level.

## Worked example: a gallery with a modal

We built and ran this on Next.js 16.3.6.

```text
app/gallery/
├── layout.tsx
├── page.tsx                     → /gallery (grid of links)
├── photo/[id]/page.tsx          → /gallery/photo/1 (full page)
└── @modal/
    ├── default.tsx              → null when no modal is open
    └── (.)photo/[id]/
        ├── page.tsx             → intercepts /gallery/photo/1 as a modal
        └── Modal.tsx
```

```tsx title="app/gallery/layout.tsx"
export default function GalleryLayout({ children, modal }: LayoutProps<"/gallery">) {
  return (
    <>
      {children}
      {modal}
    </>
  );
}
```

```tsx title="app/gallery/@modal/default.tsx"
export default function Default() {
  return null;
}
```

```tsx title="app/gallery/page.tsx"
import Link from "next/link";

const ids = ["1", "2", "3"];

export default function Gallery() {
  return (
    <ul>
      {ids.map((id) => (
        <li key={id}><Link href={`/gallery/photo/${id}`}>Photo {id}</Link></li>
      ))}
    </ul>
  );
}
```

```tsx title="app/gallery/photo/[id]/page.tsx"
export default async function PhotoPage({ params }: PageProps<"/gallery/photo/[id]">) {
  const { id } = await params;
  return (
    <main>
      <h1>Photo {id}</h1>
      <p>Full page view (direct visit or refresh).</p>
    </main>
  );
}
```

```tsx title="app/gallery/@modal/(.)photo/[id]/page.tsx"
import { Modal } from "./Modal";

export default async function PhotoModal({ params }: PageProps<"/gallery/photo/[id]">) {
  const { id } = await params;
  return (
    <Modal>
      <h2>Photo {id}</h2>
      <p>Opened as a modal over the gallery.</p>
    </Modal>
  );
}
```

```tsx title="app/gallery/@modal/(.)photo/[id]/Modal.tsx"
"use client";

import { useRouter } from "next/navigation";

export function Modal({ children }: { children: React.ReactNode }) {
  const router = useRouter();
  return (
    <div role="dialog" aria-modal="true" style={{ position: "fixed", inset: 0, background: "rgb(0 0 0 / 0.5)" }}>
      <div style={{ background: "white", margin: "10vh auto", padding: 24, maxWidth: 480 }}>
        {children}
        <button onClick={() => router.back()}>Close</button>
      </div>
    </div>
  );
}
```

The build listed both routes:

```text
├ ○ /gallery
├ ƒ /gallery/(.)photo/[id]
├ ƒ /gallery/photo/[id]
```

Behaviour:

- Click "Photo 2" on `/gallery` → the URL becomes `/gallery/photo/2`, the gallery stays
  visible, and the modal slot renders the intercepted page on top.
- Refresh, or open the link in a new tab → no interception on a full load; the real
  `/gallery/photo/2` page renders, and the `@modal` slot uses `default.tsx` (`null`).
- Close (`router.back()`) → back to `/gallery`, modal gone. The browser back button does
  the same.

For production, make the modal a proper dialog: move focus into it on open, trap focus,
close on Escape, and return focus to the link on close (the native `<dialog>` element
with `showModal()` handles most of this).

## How It Actually Works

The client router tracks, per layout, the active segment for every slot, not just for
`children`. On a soft navigation to `/gallery/photo/2`, the router asks the server for
the new tree; the server, knowing the navigation originated from `/gallery` (the
request carries the current router state), checks whether any slot contains an
intercepting route matching the target. It does — `@modal/(.)photo/[id]` — so it
renders that into the `modal` slot and keeps `children` as the gallery. On a hard
navigation there is no previous state, so interception is skipped: the URL is matched
normally, `children` becomes the photo page, and unmatched slots fall back to
`default.tsx`. That's the whole trick: the same URL resolves to different trees
depending on where you came from.

## Common mistakes

- **Missing `default.tsx`** in a slot — a build error in Next.js 16.
- **Counting `@slot` folders as a level** when choosing `(.)` vs `(..)`.
- **Closing a modal with `router.push("/gallery")`**, which adds a history entry;
  `router.back()` matches user expectations.
- **Modal content that duplicates the full page's data fetching in a different way.**
  Share the data function and a presentational component.
- **Forgetting accessibility**: focus management and Escape to close.

## Exercise

1. Build the gallery. Confirm the three behaviours (soft navigation, refresh, back).
2. Replace the `div` modal with a native `<dialog>` opened via `showModal()` in an
   effect, closing on the `close` event with `router.back()`.
3. Add a dashboard with `@stats` and `@activity` slots where `@activity` has its own
   `loading.tsx` with a 2-second delay. Confirm the stats appear first.
