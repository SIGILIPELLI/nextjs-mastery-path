---
description: "How the App Router maps folders to URLs: page.tsx, nested layouts that persist across navigation, templates, route groups and private folders."
---

# 03 · File-Based Routing & Layouts

In the App Router you never write a route table. The folder structure under `app/` *is*
the route table, and a small set of reserved file names decide what each folder does.

## Folders are segments, `page.tsx` makes them public

```text
app/
├── page.tsx                → /
├── about/
│   └── page.tsx            → /about
├── blog/
│   ├── page.tsx            → /blog
│   └── archive/
│       └── page.tsx        → /blog/archive
└── components/
    └── Header.tsx          → (not a route: no page.tsx)
```

A folder becomes reachable only when it contains a `page.tsx` (or a `route.ts` for API
endpoints — Level 2 lesson 05). That means you can colocate components, tests and
helpers next to the route that uses them without accidentally creating URLs.

A page is just a component exported as `default`:

```tsx title="app/about/page.tsx"
export default function AboutPage() {
  return (
    <main>
      <h1>About</h1>
      <p>We teach Next.js one folder at a time.</p>
    </main>
  );
}
```

## Layouts wrap everything beneath them

A `layout.tsx` receives the rendered child segment as `children` and wraps it. Layouts
nest: the root layout wraps a section layout, which wraps the page.

```tsx title="app/layout.tsx"
import Link from "next/link";
import "./globals.css";

export default function RootLayout({ children }: LayoutProps<"/">) {
  return (
    <html lang="en">
      <body>
        <header>
          <nav>
            <Link href="/">Home</Link> · <Link href="/blog">Blog</Link> ·{" "}
            <Link href="/about">About</Link>
          </nav>
        </header>
        {children}
        <footer>© Acme</footer>
      </body>
    </html>
  );
}
```

```tsx title="app/blog/layout.tsx"
export default function BlogLayout({ children }: LayoutProps<"/blog">) {
  return (
    <div className="blog-shell">
      <aside>
        <h2>Categories</h2>
        <ul>
          <li>Routing</li>
          <li>Data</li>
        </ul>
      </aside>
      <section>{children}</section>
    </div>
  );
}
```

Visiting `/blog/archive` renders:

```text
RootLayout
└── BlogLayout
    └── app/blog/archive/page.tsx
```

Only the root layout may (and must) render `<html>` and `<body>`.

### Layouts persist across navigation

When you navigate from `/blog` to `/blog/archive` with a `<Link>`, `RootLayout` and
`BlogLayout` are **not re-rendered or re-mounted** — only the page segment changes. Any
state inside a Client Component in the layout (an open sidebar, a search input, a
playing video) survives. This is a major difference from the Pages Router, where the
whole page component re-rendered.

The flip side: a layout does not know the current page. It can't read the pathname on
the server, and it won't re-run when a child changes. If you need the active path (to
highlight a nav link), do it in a small Client Component with `usePathname()` (next
lesson).

### `template.tsx`: a layout that re-mounts

A `template.tsx` has the same shape as a layout but gets a fresh instance on every
navigation. Use it only when you *want* state reset or an enter animation to replay per
page. Most apps never need one.

## Route groups: organise without changing URLs

Wrapping a folder name in parentheses removes it from the URL:

```text
app/
├── (marketing)/
│   ├── layout.tsx          ← marketing header/footer
│   ├── page.tsx            → /
│   └── pricing/page.tsx    → /pricing
└── (app)/
    ├── layout.tsx          ← app shell with sidebar
    └── dashboard/page.tsx  → /dashboard
```

Groups let different parts of the site have different layouts while sharing a URL
space. You can even give each group its own root layout (each with `<html>` and
`<body>`), but navigating between two different root layouts causes a full page load.

## Private folders: opt a folder out of routing

A folder whose name starts with an underscore — `app/_components/`, `app/_lib/` — and
everything inside it is ignored by the router, even if it contains a file called
`page.tsx`. It's a clear signal that the folder is implementation detail.

## The special files at a glance

| File | Role | Covered in |
|---|---|---|
| `page.tsx` | UI for the route; makes it public | this lesson |
| `layout.tsx` | Shared, persistent wrapper | this lesson |
| `template.tsx` | Wrapper re-mounted per navigation | this lesson |
| `loading.tsx` | Instant loading UI (a Suspense fallback) | Level 2 · 02 |
| `error.tsx` | Error boundary for the segment | Level 2 · 02 |
| `not-found.tsx` | UI for `notFound()` and unmatched URLs | Level 2 · 02 |
| `route.ts` | HTTP endpoint instead of a page | Level 2 · 05 |
| `default.tsx` | Fallback for parallel routes | Level 3 · 03 |

Dynamic segments — `[slug]`, `[...path]`, `[[...path]]` — come in Level 2 lesson 01.

## Worked example: a docs section with its own layout

Build this tree:

```text
app/
├── layout.tsx
├── page.tsx
└── docs/
    ├── layout.tsx
    ├── page.tsx
    ├── install/page.tsx
    └── _components/
        └── DocsNav.tsx
```

```tsx title="app/docs/_components/DocsNav.tsx"
import Link from "next/link";

const items = [
  { href: "/docs", label: "Overview" },
  { href: "/docs/install", label: "Install" },
];

export function DocsNav() {
  return (
    <nav aria-label="Docs">
      <ul>
        {items.map((i) => (
          <li key={i.href}>
            <Link href={i.href}>{i.label}</Link>
          </li>
        ))}
      </ul>
    </nav>
  );
}
```

```tsx title="app/docs/layout.tsx"
import { DocsNav } from "./_components/DocsNav";

export default function DocsLayout({ children }: LayoutProps<"/docs">) {
  return (
    <div style={{ display: "grid", gridTemplateColumns: "12rem 1fr", gap: "2rem" }}>
      <DocsNav />
      <article>{children}</article>
    </div>
  );
}
```

```tsx title="app/docs/install/page.tsx"
export default function InstallPage() {
  return (
    <>
      <h1>Install</h1>
      <pre>npm install next react react-dom</pre>
    </>
  );
}
```

Run `npm run build`. The table lists `/docs` and `/docs/install` but nothing for
`_components` — the underscore kept it out of the router.

## How It Actually Works

At build time Next.js walks `app/` and produces a **loader tree**: for each segment, a
record of which special files exist (layout, page, loading, error, not-found…). A URL
is matched by walking the tree segment by segment. The matched chain of files is then
turned into nested React elements, roughly:

```tsx
<RootLayout>
  <ErrorBoundary fallback={<RootError />}>     {/* if error.tsx exists */}
    <Suspense fallback={<RootLoading />}>      {/* if loading.tsx exists */}
      <BlogLayout>
        <Page />
      </BlogLayout>
    </Suspense>
  </ErrorBoundary>
</RootLayout>
```

On the client, the router keeps a cache of the RSC payload for each segment. When you
navigate, it asks the server only for the segments below the deepest layout the two
URLs share, and React reconciles just that subtree. Because the shared layouts are the
same element at the same position, React keeps their DOM and state — that's the
mechanism behind "layouts persist".

## Common mistakes

- **Forgetting `page.tsx`.** A folder with only a layout gives a 404 for that URL.
- **Expecting a layout to re-render with new data on child navigation.** It won't;
  fetch page-specific data in the page.
- **Putting `<html>`/`<body>` in a nested layout.** Only root layouts render them.
- **Using route groups to "hide" pages.** `(admin)/users/page.tsx` is still public at
  `/users`. Groups change organisation, not access control.
- **Two routes resolving to the same URL** (for example `(a)/about/page.tsx` and
  `(b)/about/page.tsx`). Next.js reports this as an error.

## Exercise

1. Create `(marketing)` and `(app)` route groups with different layouts. Put `/` and
   `/pricing` in the first and `/dashboard` and `/settings` in the second.
2. Add a Client Component with a counter (`"use client"`, `useState`) to the `(app)`
   layout. Increment it, then navigate between `/dashboard` and `/settings`. Does the
   count survive? Now rename the layout file to `template.tsx` and repeat. Explain the
   difference.
3. Add `app/_drafts/page.tsx`. Confirm with `next build` that no `/_drafts` route exists.
