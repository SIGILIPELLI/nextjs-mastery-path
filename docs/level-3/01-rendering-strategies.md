---
description: "Compare Next.js rendering strategies (static, ISR, dynamic, streaming, Partial Prerendering) and choose per route or component via the build output."
---

# 01 · Rendering Strategies

"Server-side rendering" is too coarse a term for the App Router. The same page can be
partly generated at build time, partly regenerated every hour, and partly rendered for
each visitor — and the choice has big consequences for speed, cost and correctness.
This lesson lays out the strategies side by side and gives you a way to decide.

## The strategies

| Strategy | When HTML is produced | Freshness | Cost per request | Route table |
|---|---|---|---|---|
| **Static (SSG)** | At build | Until next build | Serve a file | `○` / `●` |
| **ISR** (time or on-demand revalidation) | At build, then in background after it goes stale | Seconds to hours | Mostly a file | `○` with Revalidate column |
| **Dynamic (SSR)** | Every request | Always fresh | A full render | `ƒ` |
| **Streaming** | Every request, sent in chunks | Always fresh | Full render, faster first byte | `ƒ` |
| **Partial Prerendering (PPR)** | Shell at build; holes per request | Mixed | Serve file + render holes | `◐` |

Plus one outside the server: **client-side rendering**, where a Client Component fetches
its own data after load (lesson 09).

## What decides the strategy

In the default (previous) model, Next.js *infers* the strategy per route:

- No request-time APIs → **static**.
- `export const revalidate = N`, or `fetch(..., { next: { revalidate: N } })` → **ISR**.
- `cookies()`, `headers()`, `searchParams`, `connection()`, `cache: "no-store"` →
  **dynamic**.
- A `loading.tsx` or `<Suspense>` around slow parts of a dynamic route → **streaming**.

With Cache Components (`cacheComponents: true` in Next.js 16), the unit becomes the
**component**:

- Pure and `"use cache"` work → part of the prerendered shell.
- Request-time APIs and uncached I/O → must sit inside `<Suspense>`; rendered per
  request and streamed into the shell. This combination *is* PPR.
- A route with no dynamic holes is fully static; one with holes is `◐`.

## Partial Prerendering, concretely

The shop page from Level 2 · 04, built with Cache Components, produced:

```text
Route (app)      Revalidate  Expire
┌ ◐ /                    1h      1d
```

On request, the server immediately sends the prerendered shell (header, cached product
list, and the `<Suspense>` fallback where the greeting goes), then renders the
cookie-dependent greeting and streams it into place in the same response. One request,
no client-side fetch, and the static part costs no rendering time.

!!! note "Status of PPR"
    PPR was an experimental flag (`experimental.ppr`) in Next.js 14 and 15. In 16 that
    flag was removed; PPR is how the App Router behaves when `cacheComponents` is on.
    Hosting support matters: a plain `next start` server supports it, and hosting
    platforms need to understand the static-shell-plus-stream response to serve the
    shell from a CDN. Check your platform's documentation.

## Choosing: a decision guide

Ask of each piece of content:

1. **Is it the same for every visitor?** If no (personalised, auth-dependent) → dynamic
   (or a streamed hole).
2. **How stale may it be?** Never (stock count at checkout) → dynamic. Minutes/hours
   (product catalogue, blog) → cached with revalidation. Only changes on deploy (docs,
   marketing) → static.
3. **Do you know when it changes?** Yes (an admin edits it) → on-demand revalidation
   with tags/paths. No (third-party data) → time-based.
4. **Is it slow?** Put it behind its own Suspense boundary so it doesn't hold up the rest.

Worked through for an e-commerce product page:

| Part | Same for all? | Staleness OK? | Strategy |
|---|---|---|---|
| Header, footer, layout | Yes | Until deploy | Static shell |
| Product name, description, images | Yes | Until edited | Cached, tag `product-<id>`, `updateTag` on edit |
| Price | Yes | Minutes | Cached, short `cacheLife` |
| Stock level | Yes | Seconds | Dynamic hole or client polling |
| "Recommended for you", cart count | No | — | Dynamic holes (read cookies) |
| Reviews | Yes | Hours | Cached, streamed below the fold |

A decade ago that page would have been fully server-rendered on every request. Here,
most bytes come from cache and only the personal bits cost a render.

## Reading the route table as a review tool

Make "check the route table" a code-review habit:

```text
├ ○ /about          ← fine
├ ƒ /blog           ← why? the blog index shouldn't be dynamic
├ ● /blog/[slug]
```

A dynamic blog index usually means someone read `searchParams` or `cookies()` in a
layout the page shares. Tracking it down and moving that read into a small Suspense-
wrapped component (or a Client Component) restores static rendering.

## How It Actually Works

All strategies share one render pipeline; they differ in *when* it runs and *what is
stored*. During `next build`, Next.js renders every route in a special prerender mode
where request-time APIs are tracked. In the previous model, touching one aborts
prerendering for the whole route (it becomes `ƒ`). Under Cache Components, the renderer
keeps going: it lets everything that completes synchronously or from cache finish,
records the Suspense fallbacks for anything still pending when prerendering ends, and
saves that output as the shell, along with enough state to resume rendering the holes
later. At request time, the server writes the shell bytes immediately and resumes the
render for the holes, streaming each as it completes. ISR is the same prerender stored
with a lifetime: when a stale entry is requested, the old bytes are served and a
background render replaces them.

## Common mistakes

- **Making a whole route dynamic for one personalised widget.** Isolate it.
- **Using ISR for data that must be exact** (balances, inventory at checkout).
- **Reading cookies in the root layout** (e.g. for a theme), which makes *every* route
  dynamic. Read it in a small component, or handle theme on the client.
- **Assuming PPR works identically on every host.**
- **Optimising without looking at the route table.**

## Exercise

1. Take the product page table above and implement it with Cache Components: static
   shell, a cached product fetch with a tag, and two Suspense-wrapped dynamic parts.
   Build and confirm `◐`.
2. Deliberately read `cookies()` in the root layout of a default-model app. Build, and
   count how many routes changed from `○` to `ƒ`. Undo it.
3. For three pages of a site you know, fill in the decision-guide table and justify
   each choice in a sentence.
