---
title: "Next.js Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Next.js cheat sheet: 26 quick-reference tables from every level of the Next.js Mastery Path, each linked to its full lesson."
---

# Next.js Cheat Sheet

Quick-reference tables from **26 lessons** of the free Next.js Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Next.js Adds to React](level-1/01-what-nextjs-adds.md)

*The gaps a React-only app has to fill*

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

### [02 · create-next-app &amp; Project Structure](level-1/02-create-next-app-structure.md)

*What was generated*

| File | Purpose |
|---|---|
| `app/layout.tsx` | The **root layout**. Must render `<html>` and `<body>`. Wraps every page. |
| `app/page.tsx` | The page for `/`. |
| `app/globals.css` | Global styles, imported once in the root layout. With Tailwind v4 it starts with `@import "tailwindcss";`. |
| `app/favicon.ico` | A special file: Next.js turns it into a `<link rel="icon">` automatically. |
| `public/` | Files served as-is from the site root (`/next.svg`). Lesson 08. |
| `next.config.ts` | Framework configuration (images, redirects, headers, output mode…). |
| `tsconfig.json` | TypeScript config, including the `@/*` path alias and the `next` plugin. |
| `next-env.d.ts` | Generated type references. Don't edit it; it's regenerated. |
| `eslint.config.mjs` | ESLint flat config with Next.js rules. |
| `postcss.config.mjs` | Loads the Tailwind PostCSS plugin. |
| `AGENTS.md` | Newer versions of `create-next-app` add notes for AI coding assistants pointing at the version-matched docs in `node_modules/next/dist/docs/`. Optional (`--agents-md` flag). |

### [03 · File-Based Routing &amp; Layouts](level-1/03-file-routing-layouts.md)

*The special files at a glance*

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

### [04 · Pages &amp; Navigation with Link](level-1/04-pages-navigation-link.md)

*`<Link>` basics*

| Prop | Effect |
|---|---|
| `href` | Path string or URL object. Required. |
| `replace` | Replace the current history entry instead of pushing a new one. |
| `scroll` | Set `false` to keep the scroll position after navigation. |
| `prefetch` | `"auto"`/`null` (default), `true`, or `false` — see below. |

### [05 · Server vs Client Components](level-1/05-server-vs-client-components.md)

*What each kind can do*

| | Server Component | Client Component |
|---|---|---|
| Runs | On the server (at build or request time) | On the server for the initial HTML, then in the browser |
| `async` / `await` data | Yes | No (not as a component) |
| Access DB, file system, secrets | Yes | No — code is shipped to the browser |
| `useState`, `useEffect`, `useReducer` | No | Yes |
| Event handlers (`onClick`, `onChange`) | No | Yes |
| Browser APIs (`window`, `localStorage`) | No | Yes (inside effects / handlers) |
| Adds its code to the JS bundle | No | Yes |

### [06 · Styling Options](level-1/06-styling.md)

*Choosing*

| Situation | Reasonable choice |
|---|---|
| Small site, want zero dependencies | Global CSS for tokens + CSS Modules |
| Team already fluent in utility classes, many small components | Tailwind |
| Large existing SCSS codebase | Sass modules |
| Existing styled-components app being migrated | Keep it in Client Components for now; plan a gradual move |

### [08 · Static Assets &amp; the `public` Folder](level-1/08-static-assets-public.md)

*The caching difference*

| | `public/` file | Imported asset |
|---|---|---|
| URL | Fixed, human-readable | Hashed, changes with content |
| Default `Cache-Control` from `next start` (observed on 16.3.6) | `public, max-age=0` plus an `ETag` — revalidated every time | `public, max-age=31536000, immutable` |
| Good for | Files with meaningful, stable URLs (PDF links, `robots.txt`, verification files, `.well-known/`) | Images and files used by components |

## Level 2 · Intermediate

### [01 · Dynamic Routes &amp; Params](level-2/01-dynamic-routes-params.md)

*The three shapes*

| Folder | Matches | `params` |
|---|---|---|
| `app/products/[id]/page.tsx` | `/products/42` | `{ id: "42" }` |
| `app/docs/[...path]/page.tsx` | `/docs/a`, `/docs/a/b/c` (not `/docs`) | `{ path: ["a","b","c"] }` |
| `app/shop/[[...filters]]/page.tsx` | `/shop`, `/shop/red`, `/shop/red/xl` | `{ filters: undefined }` or `["red","xl"]` |

### [04 · Caching &amp; Revalidation](level-2/04-caching-revalidation.md)

*A short history (so old tutorials make sense)*

| Version | Default for `fetch` | Default for pages |
|---|---|---|
| 13–14 | Cached indefinitely (`force-cache`) unless opted out | Static when possible |
| 15 | **Not** cached by default | Static when possible; `GET` Route Handlers no longer cached by default |
| 16, default config | Same as 15 ("previous model" in the docs) | Same as 15 |
| 16 with `cacheComponents: true` | Nothing cached unless marked `"use cache"` | Partial prerendering: static shell + dynamic holes |

### [05 · Route Handlers (API Endpoints)](level-2/05-route-handlers.md)

*Route Handler or Server Action?*

| Need | Use |
|---|---|
| Your own form or button changes data | Server Action |
| External client, mobile app, third party | Route Handler |
| Webhook receiver | Route Handler |
| Non-HTML/JSON output: RSS, CSV, images, files | Route Handler |
| Data for your own Server Components | Neither — call the data function directly |

### [06 · Proxy (formerly Middleware)](level-2/06-proxy-middleware.md)

*What a proxy can return*

| Return | Effect |
|---|---|
| `NextResponse.next()` | Continue to the matched route (optionally with modified headers) |
| `NextResponse.redirect(url)` | Send a 307 (or given status) to the browser; URL changes |
| `NextResponse.rewrite(url)` | Serve a different route **without** changing the URL |
| `new Response(...)` / `NextResponse.json(...)` | Answer directly; no route renders |

### [07 · Authentication Patterns](level-2/07-authentication.md)

*Where to check: a data access layer*

| Layer | Check | Purpose |
|---|---|---|
| Proxy | Cookie present? | Fast redirect of obviously signed-out users |
| Layout / page | `requireUser()` | Correct UI; redirect to login |
| **Data access layer** | Session valid *and* user may access this record | **Actual security** |

### [09 · URL State &amp; Search Params](level-2/09-url-state-search-params.md)

*What belongs in the URL?*

| In the URL | Not in the URL |
|---|---|
| Filters, sort, page, search query | Form drafts mid-edit |
| Selected tab, open item ID | Hover/animation state |
| Map viewport, date range | Secrets or personal data |

### [10 · Project — Full-Stack Task Manager](level-2/10-project-crud-app.md)

*Features and where each lesson appears*

| Feature | Lesson |
|---|---|
| `/tasks/[id]/edit` with `notFound()` for bad IDs | 01, 02 |
| Create/update/toggle/delete via Server Actions + `useActionState` | 03 |
| `revalidatePath` after every write; `connection()` for live data | 04 |
| Drizzle schema and queries on SQLite | 08 |

## Level 3 · Advanced

### [01 · Rendering Strategies](level-3/01-rendering-strategies.md)

*The strategies*

| Strategy | When HTML is produced | Freshness | Cost per request | Route table |
|---|---|---|---|---|
| **Static (SSG)** | At build | Until next build | Serve a file | `○` / `●` |
| **ISR** (time or on-demand revalidation) | At build, then in background after it goes stale | Seconds to hours | Mostly a file | `○` with Revalidate column |
| **Dynamic (SSR)** | Every request | Always fresh | A full render | `ƒ` |
| **Streaming** | Every request, sent in chunks | Always fresh | Full render, faster first byte | `ƒ` |
| **Partial Prerendering (PPR)** | Shell at build; holes per request | Mixed | Serve file + render holes | `◐` |

### [03 · Parallel &amp; Intercepting Routes](level-3/03-parallel-intercepting-routes.md)

*Intercepting routes*

| Prefix | Intercepts a route… |
|---|---|
| `(.)` | at the same level |
| `(..)` | one level up |
| `(..)(..)` | two levels up |
| `(...)` | from the app root |

### [06 · Performance &amp; Core Web Vitals](level-3/06-performance-web-vitals.md)

*The three metrics*

| Metric | Measures | "Good" threshold (at the 75th percentile of visits) |
|---|---|---|
| **LCP** — Largest Contentful Paint | When the main content (largest image or text block) appears | ≤ 2.5 s |
| **INP** — Interaction to Next Paint | How quickly the page responds visually to clicks, taps and keys, over the whole visit | ≤ 200 ms |
| **CLS** — Cumulative Layout Shift | How much visible content jumps around unexpectedly | ≤ 0.1 |

### [07 · Testing with Vitest &amp; Playwright](level-3/07-testing.md)

| Layer | Tool | What it catches |
|---|---|---|
| Pure logic (validation, formatting, data mapping) | **Vitest** | Logic bugs, fast |
| Client Components | **Vitest + Testing Library** in jsdom | Rendering and interaction bugs |
| Server Actions / data layer | **Vitest** calling the functions directly (with a test DB) | Validation and authorisation bugs |
| Async Server Components, routing, caching, the real build | **Playwright** against `next build && next start` | Integration bugs, the things that only exist in the framework |

### [09 · Client-Side Data &amp; Interactivity](level-3/09-client-data-interactivity.md)

*When client fetching is the right call*

| Situation | Why the server alone isn't enough |
|---|---|
| Data changes while the user watches (prices, job status, chat) | Server render is a snapshot |
| User-triggered loading of more items (infinite scroll) | Unknown in advance how much to render |
| Highly interactive widgets with frequent small queries (autocomplete) | Full route re-renders per keystroke are wasteful |
| Browser-only sources (geolocation, IndexedDB, device APIs) | Not available on the server |
| Third-party APIs the browser calls directly with a user token | The server shouldn't proxy them |

## Level 4 · Master

### [01 · Deployment: Managed &amp; Self-Hosted](level-4/01-deployment.md)

*Five deployment shapes*

| Shape | What you deploy | Supports |
|---|---|---|
| **Node server** (`next build` + `next start`) | The project + `node_modules` | Everything |
| **Standalone output** | `.next/standalone` folder (minimal files) | Everything |
| **Docker container** | An image built from standalone output | Everything |
| **Static export** (`output: "export"`) | Plain HTML/CSS/JS files | No server features (lesson 08) |
| **Managed platform** | Your repo; the platform builds it | Everything, plus platform extras |

### [02 · Node.js vs Edge Runtime](level-4/02-edge-vs-node-runtime.md)

*The two runtimes*

| | Node.js runtime | Edge runtime |
|---|---|---|
| APIs | Full Node.js (`fs`, `net`, `crypto`, native modules, all npm packages) | Web-standard subset (`fetch`, `Request`, `Response`, Web Crypto, streams) |
| Startup | Heavier process | Very light isolates (V8 isolates, not full processes) |
| Limits | Whatever your server allows | Small code-size limits, no file system, restricted packages |
| Where it ran (typically) | Your server or a regional serverless function | Platform's globally distributed network |

### [04 · Observability](level-4/04-observability.md)

*What to alert on*

| Signal | Why |
|---|---|
| Error rate per route type (render / action / route) | Regressions after deploys |
| p95 server duration for key routes | Slow queries, missing cache |
| Rate of `onRequestError` with the same digest | One bug hitting many users |
| Web Vitals p75 per page type | User-perceived performance |
| Cache hit ratio (if your cache handler exposes it) | Revalidation or cache-key bugs |

### [06 · Upgrading &amp; Codemods](level-4/06-upgrading-codemods.md)

*What changed in 15 → 16 (summary)*

| Change | Action |
|---|---|
| Node.js 20.9+ and TypeScript 5.1+ required | Upgrade runtime/CI images |
| Turbopack default for `dev` and `build` | Move custom `webpack` config to `turbopack` or opt out with `--webpack` |
| Synchronous `params`, `searchParams`, `cookies()`, `headers()` removed | `next-async-request-api` codemod, then fix leftovers |
| `middleware` → `proxy` (deprecated name) | `middleware-to-proxy` codemod |
| `revalidateTag(tag)` single-argument form deprecated | Add a profile (`"max"`) or use `updateTag` in actions |
| `experimental.ppr`, `dynamicIO`, `useCache` → `cacheComponents` | Opt in deliberately; see the Cache Components migration guide |
| Parallel route slots need `default.js` | Add them; the build fails otherwise |
| `next lint` removed; `next build` no longer lints | Run ESLint/Biome directly in CI |
| `serverRuntimeConfig` / `publicRuntimeConfig` removed | Environment variables |
| `next/image` defaults changed (qualities, cache TTL, local IP restriction…) | Review image config |
| AMP support removed | Remove AMP pages |

### [08 · Static Export &amp; CDN Caching](level-4/08-static-export-cdn.md)

*What you give up*

| Need | Static-export approach |
|---|---|
| Redirects | Configure them on the host/CDN, or meta-refresh pages |
| Forms | A third-party form endpoint or a separate API |
| Search | Build a search index at build time and search client-side |
| Images | `unoptimized`, pre-sized images, or a custom `loader` pointing to an image CDN |
| Fresh content | Rebuild and redeploy on content change (webhook-triggered CI) |

### [09 · Production Readiness Review](level-4/09-production-readiness.md)

*Build and configuration*

| Check | Verify |
|---|---|
| `next build` passes with no type errors or warnings you haven't triaged | CI log |
| Next.js and React on a supported, patched version | `npm view next version` vs `package.json`; security advisories |
| Route table matches intent (static pages are `○`/`●`, not `ƒ`) | Save and review build output |
| Required env vars validated at startup | Start with one missing; it should fail loudly |
| No secrets in `NEXT_PUBLIC_*` | `grep -r NEXT_PUBLIC_ .env*` and review |
| `poweredByHeader: false`, source maps policy decided | `curl -I` |

### [10 · Capstone — A Production Next.js App](level-4/10-capstone.md)

*Functional and technical requirements*

| # | Requirement | Lessons |
|---|---|---|
| R1 | Marketing pages are `○` in the route table; unique metadata; OG image | L1·07, L3·05 |
| R2 | Session in an `HttpOnly` cookie; every query/action in the DAL checks session **and** team membership | L2·07, L3·08 |
| R3 | Article create/edit via Server Actions with Zod validation and accessible errors | L2·03, L4·07 |
| R4 | After edits, the editor sees changes immediately (`updateTag` or `revalidatePath`) | L2·04 |
| R5 | Article search, filters and pagination live in the URL | L2·09 |
| R6 | Dashboard panels stream independently; each has an error boundary | L3·02, L3·10 |
| R7 | Article dialog via intercepting route; full page on refresh; focus returns on close | L3·03, L4·07 |
| R8 | Markdown rendered on the server and **sanitised** (user content!) | L1·10, L3·08 |
| R9 | Unknown article → real 404 status | L2·02 |
| R10 | Security headers; decided CSP policy documented | L3·08 |
| R11 | `instrumentation.ts` with `onRequestError`; JSON logs with request IDs | L4·04 |
| R12 | Tests: Vitest for schemas/DAL, Playwright for main flows + axe + 404 status | L3·07, L4·07 |
| R13 | `output: "standalone"`; Dockerfile; health check | L4·01, L4·09 |
| R14 | README with decisions: rendering per route, caching and invalidation, runtime, hosting | all |

---

Go deeper with the full [Next.js Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
