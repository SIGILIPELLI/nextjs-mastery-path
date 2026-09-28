---
description: "Styling a Next.js App Router app: global CSS, CSS Modules, Tailwind CSS v4, Sass, and why runtime CSS-in-JS libraries need care with Server Components."
---

# 06 · Styling Options

Next.js doesn't prescribe a styling approach, but the App Router's split between server
and client code makes some approaches much smoother than others. This lesson covers the
four you'll meet most, with the trade-offs that actually matter.

## 1. Global CSS

Import a stylesheet from a layout (normally the root layout) and it applies to every
page:

```tsx title="app/layout.tsx"
import "./globals.css";
```

```css title="app/globals.css"
:root {
  --bg: #fafaf7;
  --fg: #1f2328;
  --accent: #0b6bcb;
}
@media (prefers-color-scheme: dark) {
  :root { --bg: #111418; --fg: #e6e6e6; --accent: #6cb6ff; }
}
body {
  background: var(--bg);
  color: var(--fg);
  font-family: system-ui, sans-serif;
  line-height: 1.6;
  margin: 0;
}
a { color: var(--accent); }
```

Global CSS is right for resets, design tokens (custom properties) and base element
styles. It is wrong for component styles, because class names collide across the whole
app.

## 2. CSS Modules

A file ending in `.module.css` is scoped: each class name is rewritten to something
unique at build time.

```css title="app/_components/Card.module.css"
.card {
  border: 1px solid color-mix(in srgb, var(--fg) 15%, transparent);
  border-radius: 12px;
  padding: 1rem 1.25rem;
}
.title {
  margin: 0 0 0.5rem;
  font-size: 1.125rem;
}
.card:hover {
  border-color: var(--accent);
}
```

```tsx title="app/_components/Card.tsx"
import styles from "./Card.module.css";

export function Card({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <div className={styles.card}>
      <h3 className={styles.title}>{title}</h3>
      {children}
    </div>
  );
}
```

CSS Modules work in both Server and Client Components, need no dependencies, and
produce static CSS files. For many projects they're all you need.

## 3. Tailwind CSS

`create-next-app --tailwind` sets up **Tailwind CSS v4**: a `postcss.config.mjs` using
`@tailwindcss/postcss`, and `globals.css` beginning with:

```css title="app/globals.css"
@import "tailwindcss";

@theme {
  --color-brand: #0b6bcb;
}
```

In v4 the theme is configured in CSS with `@theme` rather than a
`tailwind.config.js` (v3 projects still use the JS config). Utility classes then go
straight into JSX:

```tsx title="app/_components/Badge.tsx"
export function Badge({ children }: { children: React.ReactNode }) {
  return (
    <span className="inline-flex items-center rounded-full bg-brand/10 px-2.5 py-0.5 text-sm font-medium text-brand">
      {children}
    </span>
  );
}
```

Tailwind scans your source files for class names and generates only the CSS you use,
so it works identically in Server and Client Components.

One rule: **class names must appear as complete strings in source**. Tailwind can't see
`` `bg-${color}-500` `` because it never runs your code. Map values to full class names
instead:

```tsx
const tone = { info: "bg-sky-100 text-sky-900", warn: "bg-amber-100 text-amber-900" } as const;
<div className={tone[kind]} />
```

## 4. Sass

Install `sass` and you can use `.scss` / `.module.scss` files exactly like the CSS
equivalents. Next.js 16 uses the modern Sass API.

```bash
npm install --save-dev sass
```

## CSS-in-JS and Server Components

Libraries that generate styles **at runtime** in the browser (styled-components,
Emotion) depend on React context and client-side effects. They work only in Client
Components, and need a small "style registry" component so styles generated during
server rendering are inserted into the HTML. Each library documents its App Router
setup and support evolves, so check the library's own docs for the current state.

The practical consequences:

- Every component styled that way must be a Client Component, which undermines the
  "ship less JS" benefit of Server Components.
- Zero-runtime alternatives (CSS Modules, Tailwind, or compile-time CSS-in-JS tools that
  extract static CSS) avoid the problem entirely.

## Choosing

| Situation | Reasonable choice |
|---|---|
| Small site, want zero dependencies | Global CSS for tokens + CSS Modules |
| Team already fluent in utility classes, many small components | Tailwind |
| Large existing SCSS codebase | Sass modules |
| Existing styled-components app being migrated | Keep it in Client Components for now; plan a gradual move |

Mixing is fine — Tailwind for layout plus a CSS Module for one complex component is a
common, sensible combination.

## Worked example: a themed post card

```css title="app/blog/PostCard.module.css"
.card { display: grid; gap: 0.25rem; padding: 1rem; border-radius: 10px; }
.card:focus-within { outline: 2px solid var(--accent); outline-offset: 2px; }
.meta { font-size: 0.875rem; opacity: 0.75; }
.link { text-decoration: none; font-weight: 600; }
.link::after { content: ""; position: absolute; inset: 0; } /* whole card clickable */
.wrapper { position: relative; }
```

```tsx title="app/blog/PostCard.tsx"
import Link from "next/link";
import styles from "./PostCard.module.css";

export function PostCard({ slug, title, date }: { slug: string; title: string; date: string }) {
  return (
    <div className={`${styles.wrapper} ${styles.card}`}>
      <Link className={styles.link} href={`/blog/${slug}`}>{title}</Link>
      <time className={styles.meta} dateTime={date}>
        {new Date(date).toLocaleDateString("en-GB", { dateStyle: "medium" })}
      </time>
    </div>
  );
}
```

The card is a Server Component, makes the whole area clickable with a pseudo-element
instead of wrapping block content in a link, and shows a focus ring for keyboard users.

## How It Actually Works

CSS in Next.js is handled by the bundler, not at runtime. When the server module graph
imports a `.css` or `.module.css` file, the bundler extracts it into CSS chunks and
records which chunks each route needs. For CSS Modules it also rewrites every class name
to a unique identifier and gives your component an object mapping the original names to
the rewritten ones — `styles.card` is just a string. The route's HTML then includes
`<link rel="stylesheet">` tags for its chunks, and the client router loads missing
chunks before revealing a new page on navigation.

Tailwind runs as a PostCSS plugin inside the same pipeline: it scans source files for
candidate class names, generates matching rules, and the result is just another CSS
file. Because all of this happens at build time, none of it depends on whether a
component is a Server or Client Component — which is exactly why runtime CSS-in-JS is
the odd one out.

## Common mistakes

- **Importing global CSS from a deeply nested component.** It still applies globally,
  and the load order becomes hard to reason about. Import global CSS from layouts.
- **Building Tailwind classes dynamically** (`text-${size}`). Use a lookup of full
  class names.
- **Using a runtime CSS-in-JS library in Server Components.** It won't work; either mark
  the component client or switch approach for that component.
- **Relying on CSS order between modules.** Chunk order can differ between dev and
  production; don't let two modules fight over the same element.
- **Forgetting dark mode and focus styles.** Put colours in custom properties so a theme
  change is one block of CSS.

## Exercise

1. Create a `Button` component with a CSS Module offering `primary` and `secondary`
   variants selected by a prop. It must be a Server Component.
2. Recreate the same button with Tailwind utility classes. Compare the two in terms of
   readability and how you'd change the brand colour everywhere.
3. Run `npm run build`, then look in `.next/static/` for the generated CSS. Find your
   rewritten CSS Module class names.
