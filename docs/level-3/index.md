---
description: "Level 3 of Next.js: rendering strategies and PPR, streaming, parallel and intercepting routes, i18n, SEO, Web Vitals, testing, security, client data."
---

# Level 3 · Advanced <span class="level-badge">Depth</span>

Goal: make deliberate, defensible choices about *how* each part of your app is
rendered and delivered — and prove the result is fast, findable, secure and correct.

By now you can build a full-stack Next.js app. Level 3 is about the difference between
an app that works and one that holds up: pages that show useful content in the first
few hundred milliseconds, URLs that search engines and link previews understand, UI
patterns such as modals that survive a refresh, a test suite you trust, and a clear
picture of which functions are exposed to the internet.

Three ideas run through this level:

1. **Rendering is a per-component decision.** Static, dynamic and streamed content can
   live on the same page. The skill is putting the boundaries in the right places.
2. **Measure, don't guess.** Core Web Vitals, the build's route table and real
   `curl` output tell you what's happening; intuition about "fast" is often wrong.
3. **Everything on the server is an attack surface.** Server Actions and Route
   Handlers are public endpoints; environment variables and headers are part of your
   security design.

## Modules

1. [Rendering Strategies](01-rendering-strategies.md) — static, dynamic, ISR, streaming and Partial Prerendering compared
2. [Suspense & Streaming](02-suspense-streaming.md) — independent loading sections and what streaming does on the wire
3. [Parallel & Intercepting Routes](03-parallel-intercepting-routes.md) — slots, `default.tsx`, and URL-addressable modals
4. [Internationalization](04-internationalization.md) — `[lang]` segments, dictionaries, locale detection and `hreflang`
5. [SEO in Depth](05-seo.md) — metadata, sitemaps, robots, Open Graph images and structured data
6. [Performance & Core Web Vitals](06-performance-web-vitals.md) — LCP, INP, CLS, bundle size and measuring in the field
7. [Testing with Vitest & Playwright](07-testing.md) — unit, component and end-to-end tests for the App Router
8. [Security](08-security.md) — Server Action exposure, data leaks to the client, env vars, headers and CSP
9. [Client-Side Data & Interactivity](09-client-data-interactivity.md) — when to fetch in the browser, SWR/TanStack Query, polling
10. [Project — Streaming Analytics Dashboard](10-project-dashboard.md) — a dashboard that streams independent panels with a modal drill-down

## Before you start

You should be comfortable with everything in Level 2, especially Server Actions,
caching (Level 2 · 04) and dynamic routes. Several lessons compare behaviour with and
without **Cache Components** (`cacheComponents: true`); if that setting is new to you,
re-read Level 2 lesson 04 first.

!!! note "Experimental and fast-moving features"
    Partial Prerendering shipped as experimental in Next.js 14–15 and became the
    default behaviour of the Cache Components model in 16. Details of prefetching and
    related flags continue to evolve in minor releases. Lessons mark such features
    clearly; verify against the docs bundled with your installed version.
