---
description: "Level 1 of Next.js: what it adds to React, project structure, routing, layouts, Server and Client Components, styling, images, metadata and data."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: build and ship a small multi-page Next.js site and be able to explain what the
framework did for you at every step — which files became routes, which components ran
on the server, which JavaScript was sent to the browser, and which pages were turned
into plain HTML at build time.

You already know React (if not, begin with
[React Mastery Path](https://sigilipelli.github.io/react-mastery-path/)). This level is
about the layer on top: the conventions Next.js adds and the build and server machinery
that make those conventions work.

Three ideas run through the whole level:

1. **The file system is the router.** A folder under `app/` is a URL segment; a
   `page.tsx` inside it makes the segment reachable; a `layout.tsx` wraps it and every
   segment below it. You will rarely write routing code.
2. **Components are Server Components unless you say otherwise.** They run on the
   server (or at build time), can `await` data directly, and send no JavaScript of their
   own to the browser. You opt into the browser with `"use client"` only where you need
   state, effects or event handlers.
3. **`next build` tells the truth.** Its route table shows which pages are static (`○`),
   pre-generated from parameters (`●`), or rendered per request (`ƒ`). Get used to
   reading it — it is the fastest way to catch a page that became dynamic by accident.

## Modules

1. [What Next.js Adds to React](01-what-nextjs-adds.md) — routing, rendering, data, bundling and the server that React alone does not provide
2. [create-next-app & Project Structure](02-create-next-app-structure.md) — scaffolding a project and what every generated file is for
3. [File-Based Routing & Layouts](03-file-routing-layouts.md) — folders, `page.tsx`, nested layouts, route groups and private folders
4. [Pages & Navigation with Link](04-pages-navigation-link.md) — `<Link>`, prefetching, client-side transitions and programmatic navigation
5. [Server vs Client Components](05-server-vs-client-components.md) — where code runs, the `"use client"` boundary and what can cross it
6. [Styling Options](06-styling.md) — global CSS, CSS Modules, Tailwind CSS and the trade-offs of CSS-in-JS
7. [Images, Fonts & Metadata](07-images-fonts-metadata.md) — `next/image`, `next/font` and the Metadata API
8. [Static Assets & the public Folder](08-static-assets-public.md) — what `public/` serves, caching, and when to import instead
9. [Fetching Data in Server Components](09-data-fetching-basics.md) — `async` components, parallel requests, and where secrets stay
10. [Project — A Markdown Blog](10-project-blog.md) — a statically generated blog with an index, post pages and per-post metadata

## What you need before starting

- **Node.js 20.9 or newer.** That is the minimum for Next.js 16. Check with `node -v`.
- **React fundamentals**: components, props, `useState`, `useEffect`, lists and keys.
- **Basic TypeScript** helps; every example uses `.tsx`, but the types are light. See
  [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/) if
  you need a refresher.
- A terminal and an editor. No accounts, cloud services or paid tools are needed for
  Levels 1–3.

## How the code is organized

Code blocks are titled with the file's path relative to the project root, for example
`app/blog/page.tsx`. In Next.js the path is part of the meaning — the same component
behaves differently as `page.tsx`, `layout.tsx` or `loading.tsx` — so always create the
file at exactly that path. Blocks labelled `text` show terminal output; where output is
shown, it was captured from a real run on Next.js 16.3.6.

!!! note "About versions"
    Next.js moves fast and some APIs changed shape between major versions (for example,
    `params` became a Promise in version 15, and `middleware.ts` was renamed
    `proxy.ts` in version 16). When a lesson depends on a version-specific detail it
    says so. If your installed version differs, the documentation shipped inside the
    package at `node_modules/next/dist/docs/` matches your version exactly.
