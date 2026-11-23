---
title: "Next.js Interview Questions and Answers (179 Questions)"
description: "179 Next.js interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Next.js Mastery Path."
---

# Next.js Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Next.js Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Next.js Adds to React](level-1/01-what-nextjs-adds.md)

**Q1.** What does Next.js add to React?

**Answer:** Routing, server rendering, data fetching, bundling and deployment conventions

A full-stack React framework.

**Q2.** What is the App Router?

**Answer:** A file-system router in the app/ directory built on React Server Components

Layouts, streaming, server actions.

**Q3.** Why server-render pages?

**Answer:** Faster first paint, better SEO and less client JavaScript

HTML arrives ready.

**Q4.** What does `next dev` do?

**Answer:** Starts the development server with hot reloading

`next build` is for production.

### [02 · create-next-app &amp; Project Structure](level-1/02-create-next-app-structure.md)

**Q5.** What does `create-next-app` generate?

**Answer:** A project with app/, public/, config and scripts

Starter.

**Q6.** Where do static files like favicons live?

**Answer:** The public/ directory

Served from the root.

**Q7.** What is next.config.js for?

**Answer:** Configuring the build/runtime (images, redirects, headers)

Project config.

**Q8.** What do the TypeScript defaults give you?

**Answer:** Type checking for components, routes and config

Safer.

### [03 · File-Based Routing &amp; Layouts](level-1/03-file-routing-layouts.md)

**Q9.** How is a route created in the App Router?

**Answer:** A folder with a page.tsx file

File-system routing.

**Q10.** What does layout.tsx do?

**Answer:** Wraps child routes with shared UI that persists across navigation

Nested layouts.

**Q11.** What's a route segment?

**Answer:** A folder name in the path (e.g. app/blog/page.tsx is /blog)

Nesting.

**Q12.** What does a route group (marketing) do?

**Answer:** Organizes routes without affecting the URL

Parentheses.

### [04 · Pages &amp; Navigation with Link](level-1/04-pages-navigation-link.md)

**Q13.** Which component does client-side navigation between pages?

**Answer:** next/link's Link

Prefetching included.

**Q14.** What does Link prefetch?

**Answer:** Routes in the viewport so navigation feels instant

Production only.

**Q15.** How navigate programmatically in a client component?

**Answer:** useRouter().push('/path')

From next/navigation.

**Q16.** What does usePathname return?

**Answer:** The current URL path

Client hook.

### [05 · Server vs Client Components](level-1/05-server-vs-client-components.md)

**Q17.** What are Server Components by default in the App Router?

**Answer:** Components that render on the server with no client JS

Zero bundle cost.

**Q18.** When do you add 'use client'?

**Answer:** When you need state, effects, event handlers or browser APIs

Client boundary.

**Q19.** Can a Server Component import a Client Component?

**Answer:** Yes; the client component hydrates in the browser

Composition.

**Q20.** Can a Client Component import a Server Component directly?

**Answer:** No, but it can receive one as children/props

Pass as props.

### [06 · Styling Options](level-1/06-styling.md)

**Q21.** Which styling approaches does Next.js support?

**Answer:** CSS Modules, global CSS, Tailwind and CSS-in-JS (with client caveats)

Flexible.

**Q22.** What's a CSS Module?

**Answer:** Locally scoped class names per component file

styles.module.css.

**Q23.** Where is global CSS imported?

**Answer:** In the root layout

Once.

**Q24.** Why do CSS-in-JS libraries need care?

**Answer:** They often require client components and runtime support

Server components.

### [07 · Images, Fonts &amp; Metadata](level-1/07-images-fonts-metadata.md)

**Q25.** Why use next/image?

**Answer:** Automatic resizing, lazy loading and modern formats to improve performance

Core Web Vitals.

**Q26.** Why give images width and height?

**Answer:** Reserve space and avoid layout shift (CLS)

Or fill.

**Q27.** What does next/font do?

**Answer:** Self-hosts fonts and eliminates layout shift

No external requests.

**Q28.** How set page metadata?

**Answer:** Export a metadata object or generateMetadata

SEO.

### [08 · Static Assets &amp; the `public` Folder](level-1/08-static-assets-public.md)

**Q29.** Files in public/ are available at?

**Answer:** The site root (public/logo.png is /logo.png)

Direct URL.

**Q30.** When use public/ over importing?

**Answer:** For files referenced by fixed URL like robots.txt and favicons

No hashing.

**Q31.** Do imported assets get hashed names?

**Answer:** Yes, enabling long-term caching

Bundler.

**Q32.** What's a downside of public/?

**Answer:** No automatic optimization or cache busting

Use imports.

### [09 · Fetching Data in Server Components](level-1/09-data-fetching-basics.md)

**Q33.** How fetch data in a Server Component?

**Answer:** Make the component async and await fetch or a database call

Direct on the server.

**Q34.** What are the benefits?

**Answer:** No client waterfalls and secrets stay on the server

Security.

**Q35.** What does fetch cache control accept?

**Answer:** cache and next.revalidate options

Defaults changed across versions.

**Q36.** How run requests in parallel?

**Answer:** Start them together and await Promise.all

Avoid waterfalls.

### [10 · Project — A Markdown Blog](level-1/10-project-blog.md)

**Q37.** What does the blog project combine?

**Answer:** Routing, layouts, metadata, static content and data fetching

Level 1 project.

**Q38.** How render posts from files or a CMS?

**Answer:** Server components reading content and generateStaticParams for pages

Static generation.

**Q39.** How improve SEO for posts?

**Answer:** Per-post metadata and semantic HTML

generateMetadata.

**Q40.** How test navigation?

**Answer:** Link-based e2e/integration tests

Playwright.

## Level 2 · Intermediate

### [01 · Dynamic Routes &amp; Params](level-2/01-dynamic-routes-params.md)

**Q41.** How define a dynamic route?

**Answer:** A folder like app/blog/[slug]/page.tsx

params.slug.

**Q42.** What are params in a page component?

**Answer:** An object (promise in recent versions) with the dynamic segments

Await in newer Next.

**Q43.** What does [...slug] match?

**Answer:** Catch-all segments (one or more)

[[...slug]] is optional.

**Q44.** What does generateStaticParams do?

**Answer:** Tells Next which dynamic paths to prerender at build time

Static generation.

### [02 · Loading, Error &amp; Not-Found UI](level-2/02-loading-error-not-found.md)

**Q45.** What does loading.tsx provide?

**Answer:** An instant loading UI shown while a segment streams

Suspense boundary.

**Q46.** What does error.tsx do?

**Answer:** Catches errors in a segment and shows a fallback (client component)

Error boundary.

**Q47.** What does notFound() do?

**Answer:** Renders the nearest not-found.tsx with a 404 status

Missing resources.

**Q48.** Why colocate these files?

**Answer:** Each segment controls its own loading and error UX

Granular.

### [03 · Server Actions &amp; Forms](level-2/03-server-actions-forms.md)

**Q49.** What is a Server Action?

**Answer:** An async function that runs on the server, invoked from forms or client code

'use server'.

**Q50.** How call it from a form?

**Answer:** Pass it to the form's action prop

Progressive enhancement.

**Q51.** Why validate input in Server Actions?

**Answer:** They're public endpoints; never trust client data

Zod.

**Q52.** How update the UI after a mutation?

**Answer:** revalidatePath/revalidateTag or redirect

Cache invalidation.

### [04 · Caching &amp; Revalidation](level-2/04-caching-revalidation.md)

**Q53.** What does revalidate: 60 mean?

**Answer:** Cached data may be regenerated at most every 60 seconds (ISR)

Time-based.

**Q54.** What does revalidateTag do?

**Answer:** Invalidates cached data tagged with that label on demand

On-demand ISR.

**Q55.** What's the Full Route Cache?

**Answer:** Prerendered HTML/RSC payloads cached on the server

Static routes.

**Q56.** When is a route dynamic?

**Answer:** When it uses cookies, headers or uncached data per request

Dynamic APIs.

### [05 · Route Handlers (API Endpoints)](level-2/05-route-handlers.md)

**Q57.** What is a route handler?

**Answer:** A route.ts exporting HTTP method functions (GET, POST)

API endpoints.

**Q58.** What do route handlers receive?

**Answer:** A Request and return a Response (NextResponse)

Web standard.

**Q59.** When use route handlers vs server actions?

**Answer:** Handlers for external clients/webhooks; actions for app form/mutations

Different consumers.

**Q60.** How read JSON body?

**Answer:** await request.json()

Web API.

### [06 · Proxy (formerly Middleware)](level-2/06-proxy-middleware.md)

**Q61.** What is middleware (proxy) used for?

**Answer:** Running code before a request completes: redirects, auth checks, headers

Edge-friendly.

**Q62.** Where is it defined?

**Answer:** A middleware/proxy file at the project root (src/)

matcher config.

**Q63.** What should it avoid?

**Answer:** Heavy work and database calls (runs on every matched request)

Keep it light.

**Q64.** What's a matcher?

**Answer:** A config limiting which paths run the middleware

Performance.

### [07 · Authentication Patterns](level-2/07-authentication.md)

**Q65.** Where verify sessions securely?

**Answer:** On the server near the data access, not only in middleware

Defense in depth.

**Q66.** How store session tokens?

**Answer:** HttpOnly, Secure, SameSite cookies

XSS safety.

**Q67.** What does Auth.js (NextAuth) provide?

**Answer:** Authentication flows and providers

Integrations.

**Q68.** What must every protected action check?

**Answer:** Authorization for that specific resource

IDOR.

### [08 · Databases with an ORM](level-2/08-databases-orm.md)

**Q69.** Where run database queries?

**Answer:** In server components, server actions or route handlers (server only)

Secrets safe.

**Q70.** Why reuse the DB client in dev?

**Answer:** Hot reload creates many connections otherwise

Singleton pattern.

**Q71.** What do ORMs like Prisma/Drizzle give?

**Answer:** Typed queries and migrations

Safer data access.

**Q72.** How avoid SQL injection?

**Answer:** Use parameterized queries via the ORM

Never string-build.

### [09 · URL State &amp; Search Params](level-2/09-url-state-search-params.md)

**Q73.** Why keep UI state in the URL?

**Answer:** Shareable, bookmarkable and works with back/forward

Search params.

**Q74.** How read search params in a server page?

**Answer:** Via the searchParams prop

Dynamic.

**Q75.** How update them on the client?

**Answer:** useRouter/Link with new query strings or useSearchParams

Navigation.

**Q76.** What to debounce?

**Answer:** Search-as-you-type updates to avoid too many requests

UX.

### [10 · Project — Full-Stack Task Manager](level-2/10-project-crud-app.md)

**Q77.** What does the CRUD app project combine?

**Answer:** Dynamic routes, server actions, validation, DB and revalidation

Level 2 project.

**Q78.** How handle validation errors?

**Answer:** Return errors from the action and render them in the form

useActionState.

**Q79.** How ensure lists update after creating?

**Answer:** revalidatePath for the list route

Cache.

**Q80.** How protect mutations?

**Answer:** Authenticate and authorize inside each action

Server-side.

## Level 3 · Advanced

### [01 · Rendering Strategies](level-3/01-rendering-strategies.md)

**Q81.** Which strategy prerenders at build time?

**Answer:** Static rendering (SSG)

Fast and cacheable.

**Q82.** When use dynamic rendering?

**Answer:** When content depends on the request (user, cookies, fresh data)

Per request.

**Q83.** What is ISR?

**Answer:** Static pages regenerated in the background after a time or event

Best of both.

**Q84.** What's partial prerendering?

**Answer:** A static shell with dynamic holes streamed in

Hybrid.

### [02 · Suspense &amp; Streaming](level-3/02-suspense-streaming.md)

**Q85.** What does Suspense enable?

**Answer:** Showing fallbacks while parts of the UI stream in

Progressive rendering.

**Q86.** What does streaming improve?

**Answer:** Time to first byte/paint for slow data sections

Don't block the page.

**Q87.** Where place Suspense boundaries?

**Answer:** Around slow, independent sections

Granular.

**Q88.** What does loading.tsx wrap?

**Answer:** The segment in a Suspense boundary automatically

Convention.

### [03 · Parallel &amp; Intercepting Routes](level-3/03-parallel-intercepting-routes.md)

**Q89.** What are parallel routes?

**Answer:** Multiple pages rendered simultaneously in slots (@folder)

Dashboards.

**Q90.** What are intercepting routes?

**Answer:** Showing another route's content in the current context (e.g. a modal)

(.)photo.

**Q91.** Typical use of interception?

**Answer:** Modal previews with shareable URLs

UX.

**Q92.** What does default.tsx do for slots?

**Answer:** Provides a fallback when a slot has no match

Hard navigation.

### [04 · Internationalization](level-3/04-internationalization.md)

**Q93.** How localize routes?

**Answer:** Locale segments (app/[lang]) with a dictionary and middleware detection

i18n routing.

**Q94.** What helps SEO for locales?

**Answer:** hreflang alternates and localized metadata

Search engines.

**Q95.** Where load translations?

**Answer:** On the server so they ship no extra client JS

Server components.

**Q96.** How format dates and numbers?

**Answer:** Intl APIs with the locale

Correctness.

### [05 · SEO in Depth](level-3/05-seo.md)

**Q97.** How add sitemap and robots?

**Answer:** sitemap.ts and robots.ts route files

Generated.

**Q98.** What does canonical URL prevent?

**Answer:** Duplicate-content confusion

alternates.canonical.

**Q99.** What helps rich results?

**Answer:** Structured data (JSON-LD)

Schema.org.

**Q100.** What affects SEO most technically?

**Answer:** Crawlable HTML, speed and correct metadata

Fundamentals.

### [06 · Performance &amp; Core Web Vitals](level-3/06-performance-web-vitals.md)

**Q101.** Which are Core Web Vitals?

**Answer:** LCP, INP and CLS

User-centric.

**Q102.** What improves LCP?

**Answer:** Optimizing the hero image/font and server response

priority on images.

**Q103.** What causes CLS?

**Answer:** Content shifting due to missing dimensions or late-loading fonts/ads

Reserve space.

**Q104.** How reduce JavaScript?

**Answer:** Prefer server components, dynamic imports and smaller dependencies

Bundle analyzer.

### [07 · Testing with Vitest &amp; Playwright](level-3/07-testing.md)

**Q105.** What test types fit Next.js?

**Answer:** Unit/component (Jest/Vitest), e2e (Playwright) and integration

Pyramid.

**Q106.** How test async server components?

**Answer:** E2E or integration tests (unit tools struggle)

Current limitation.

**Q107.** What do you mock in unit tests?

**Answer:** next/navigation hooks and network calls

Isolation.

**Q108.** What does Playwright give?

**Answer:** Real browser tests across the full stack

Reliable flows.

### [08 · Security](level-3/08-security.md)

**Q109.** Which header reduces XSS risk?

**Answer:** Content-Security-Policy

Configure.

**Q110.** Why validate Server Action input?

**Answer:** Actions are callable by anyone; treat as public APIs

Zod.

**Q111.** How keep secrets out of the browser?

**Answer:** Don't prefix with NEXT_PUBLIC_; use server-only code

Env vars.

**Q112.** What's CSRF protection in Server Actions?

**Answer:** Origin checks built in plus SameSite cookies

Defense.

### [09 · Client-Side Data &amp; Interactivity](level-3/09-client-data-interactivity.md)

**Q113.** When use SWR/React Query?

**Answer:** Client-side data that needs caching, polling or optimistic updates

Interactivity.

**Q114.** What's optimistic UI?

**Answer:** Updating the UI immediately before the server confirms

useOptimistic.

**Q115.** How mix server and client data?

**Answer:** Fetch initial data on the server and hydrate client state

Initial data.

**Q116.** What should live in client state?

**Answer:** Interactive UI state, not server data you can fetch on the server

Minimize.

### [10 · Project — Streaming Analytics Dashboard](level-3/10-project-dashboard.md)

**Q117.** What does the dashboard project combine?

**Answer:** Streaming, parallel routes, charts, auth and caching

Level 3 project.

**Q118.** How keep slow widgets from blocking?

**Answer:** Suspense per widget

Streaming.

**Q119.** How ensure accurate numbers?

**Answer:** Server-side queries with caching rules and tests

Correctness.

**Q120.** How measure performance?

**Answer:** Lighthouse/Web Vitals and real-user monitoring

Evidence.

## Level 4 · Master

### [01 · Deployment: Managed &amp; Self-Hosted](level-4/01-deployment.md)

**Q121.** What's the simplest Next.js hosting?

**Answer:** Vercel (zero-config), or Node/Docker/self-host and static export

Many options.

**Q122.** What does `next build` output?

**Answer:** An optimized production build (.next) with static and server parts

Then `next start`.

**Q123.** How containerize?

**Answer:** Use the standalone output in a minimal Node image

output: 'standalone'.

**Q124.** What config differs per environment?

**Answer:** Environment variables (runtime and build-time)

12-factor.

### [02 · Node.js vs Edge Runtime](level-4/02-edge-vs-node-runtime.md)

**Q125.** Edge runtime vs Node runtime?

**Answer:** Edge is lightweight and globally distributed with limited APIs; Node has full APIs

Choose per route.

**Q126.** What can't Edge usually do?

**Answer:** Use Node-only APIs and most native drivers

Limitations.

**Q127.** When pick Edge?

**Answer:** Low-latency personalization, redirects and lightweight logic

Speed.

**Q128.** How set runtime?

**Answer:** export const runtime = 'edge' | 'nodejs'

Route-level.

### [03 · Monorepos](level-4/03-monorepos.md)

**Q129.** Why use a monorepo?

**Answer:** Share packages (UI, config, types) across apps with unified tooling

Turborepo/Nx.

**Q130.** What does Turborepo cache?

**Answer:** Task outputs keyed by inputs to skip rebuilds

Remote cache.

**Q131.** How share UI components?

**Answer:** Internal packages with transpilePackages

Config.

**Q132.** What risk to manage?

**Answer:** Dependency version drift and build times

Workspaces.

### [04 · Observability](level-4/04-observability.md)

**Q133.** What does OpenTelemetry in Next.js provide?

**Answer:** Traces for requests, rendering and fetches

instrumentation.ts.

**Q134.** Which signals matter?

**Answer:** Logs, metrics and traces plus Web Vitals from real users

Observability.

**Q135.** How track client errors?

**Answer:** An error monitoring service (Sentry) with source maps

Production.

**Q136.** Why add request IDs?

**Answer:** Correlate logs across services

Debugging.

### [05 · Large-App Architecture](level-4/05-large-app-architecture.md)

**Q137.** How organize a large app?

**Answer:** Feature folders, shared UI packages and clear server/client boundaries

Maintainability.

**Q138.** What does the server-only package do?

**Answer:** Prevents server modules from being imported by client code

Safety.

**Q139.** Where keep business logic?

**Answer:** In framework-agnostic modules called by actions/handlers

Testable.

**Q140.** What prevents tangled code?

**Answer:** Dependency rules and linting boundaries

Architecture.

### [06 · Upgrading &amp; Codemods](level-4/06-upgrading-codemods.md)

**Q141.** How upgrade Next.js safely?

**Answer:** Read the upgrade guide, use codemods, upgrade on a branch and run tests

@next/codemod.

**Q142.** What do codemods do?

**Answer:** Automate code changes for breaking API updates

Time saver.

**Q143.** What helps pin behavior?

**Answer:** Lockfiles and CI builds

Reproducible.

**Q144.** What should you read first?

**Answer:** Release notes and the migration guide

Breaking changes.

### [07 · Accessibility](level-4/07-accessibility.md)

**Q145.** What's key for accessible Next.js apps?

**Answer:** Semantic HTML, focus management and ARIA only when needed

WCAG.

**Q146.** How handle route change announcements?

**Answer:** Next's route announcer and focus management

Screen readers.

**Q147.** Which lint helps a11y?

**Answer:** eslint-plugin-jsx-a11y

Static checks.

**Q148.** How test accessibility?

**Answer:** axe checks in tests plus manual keyboard/screen reader testing

Automated is partial.

### [08 · Static Export &amp; CDN Caching](level-4/08-static-export-cdn.md)

**Q149.** What's static export?

**Answer:** output: 'export' producing plain HTML/CSS/JS for any static host/CDN

No server features.

**Q150.** What features don't work with static export?

**Answer:** Server actions, route handlers and dynamic rendering

Limitations.

**Q151.** When choose it?

**Answer:** Content sites without server needs and cheap global hosting

Simple.

**Q152.** How handle images?

**Answer:** Use unoptimized or a custom loader (no image server)

Config.

### [09 · Production Readiness Review](level-4/09-production-readiness.md)

**Q153.** What does production readiness include?

**Answer:** Security headers, monitoring, caching strategy, error pages and tests

Checklist.

**Q154.** What should fail the pipeline?

**Answer:** Type errors, lint errors, failing tests and build errors

CI gates.

**Q155.** How roll back?

**Answer:** Redeploy the previous immutable build

Fast recovery.

**Q156.** What caches need attention?

**Answer:** CDN, data cache and browser caching headers

Invalidation.

### [10 · Capstone — A Production Next.js App](level-4/10-capstone.md)

**Q157.** What does the capstone combine?

**Answer:** Routing, rendering strategy, auth, data, performance and deployment

Production app.

**Q158.** How prove quality?

**Answer:** Lighthouse, tests, accessibility and security review

Evidence.

**Q159.** What documentation helps?

**Answer:** README, architecture decisions and runbooks

Onboarding.

**Q160.** What sustains it?

**Answer:** CI, upgrades and monitoring

Maintenance.

## Mixed questions across all levels

**Q161.** What's the main benefit of Server Components by default?

**Answer:** Less client JavaScript and direct, secure access to server data

Components render on the server.

**Q162.** You need useState and a click handler. What do you add?

**Answer:** 'use client' at the top of the component file

Marks the client boundary.

**Q163.** Which file creates the /blog/[slug] page?

**Answer:** app/blog/[slug]/page.tsx

Folder names are segments.

**Q164.** What does a route group like (marketing) do to the URL?

**Answer:** Nothing: it only organizes files

Parentheses are omitted.

**Q165.** Where should a slow, independent widget be wrapped so the page can stream?

**Answer:** In a Suspense boundary with a fallback

Progressive rendering.

**Q166.** Which file shows an instant loading state while a segment fetches?

**Answer:** loading.tsx

Automatic Suspense.

**Q167.** A server action receives form data. What must you do first?

**Answer:** Validate and authorize it: actions are public endpoints

Treat input as untrusted.

**Q168.** After a mutation, how do you refresh a cached list page?

**Answer:** revalidatePath or revalidateTag in the action

Cache invalidation.

**Q169.** What does `revalidate: 60` mean for fetched data?

**Answer:** It may be regenerated at most once per 60 seconds (time-based ISR)

Stale-while-revalidate.

**Q170.** When is a route handler (route.ts) the right tool?

**Answer:** For external clients, webhooks or non-React consumers

Server actions are for your own UI.

**Q171.** Where should authorization be enforced?

**Answer:** Near data access on the server (not only in middleware)

Defense in depth.

**Q172.** Why keep filters and pagination in search params?

**Answer:** They're shareable, bookmarkable and work with back/forward

URL as state.

**Q173.** Which strategy gives a static shell with dynamic holes streamed in?

**Answer:** Partial prerendering

Hybrid.

**Q174.** A product image causes layout shift. What fixes CLS?

**Answer:** next/image with width/height (or fill) to reserve space

Core Web Vitals.

**Q175.** Which Core Web Vitals does Next.js help with?

**Answer:** LCP, INP and CLS

Measure with Lighthouse and RUM.

**Q176.** A secret key must never reach the browser. How are env vars named?

**Answer:** Without the NEXT_PUBLIC_ prefix and used only in server code

Prefixed vars are inlined.

**Q177.** A route needs Node-only database drivers. Which runtime?

**Answer:** Node.js runtime (Edge has limited APIs)

export const runtime.

**Q178.** You host on a plain CDN with no server. What mode?

**Answer:** Static export (output: 'export'), accepting no server features

No actions or handlers.

**Q179.** How do you upgrade major versions safely?

**Answer:** Read the guide, run codemods on a branch and rely on tests and CI

@next/codemod.

## Practical coding questions

### File-system routing and params

Implement toRoute (strip app/, route groups and the leaf file) and match: static segments, [x], [...x] and [[...x]] with parameters, preferring routes with more static segments, and 404 for unknown URLs. It models how the App Router maps folders to URLs.

Starter code:

```
// App Router style file-system routes. Files are paths like "app/blog/[slug]/page.tsx".
const files = [
  "app/page.tsx",
  "app/(marketing)/about/page.tsx",
  "app/blog/page.tsx",
  "app/blog/[slug]/page.tsx",
  "app/shop/[...parts]/page.tsx",
  "app/docs/[[...path]]/page.tsx",
  "app/api/health/route.ts",
];

function toRoute(file) {
  // strip the leading "app/", drop the leaf (page.* => kind "page", route.* => "handler") and route
  // groups "(name)"; return { pattern: "/" + remaining segments joined, kind, segs }
  return { pattern: "/", kind: "page", segs: [] };
}

const routes = files.map(toRoute);

function match(url) {
  // static segments match literally; [x] matches one segment; [...x] one or more; [[...x]] zero or more;
  // if several routes match prefer the one with more static segments. Return
  // "<kind> <pattern> <params as JSON>" or "404 not-found"
  return "404 not-found";
}

console.log(routes.map((r) => r.pattern).join("  "));
for (const u of ["/", "/about", "/blog", "/blog/hello-world", "/shop/a/b/c", "/docs", "/docs/x/y", "/api/health", "/nope"]) {
  console.log(u.padEnd(18), match(u));
}
```

Expected output:

```text
/  /about  /blog  /blog/[slug]  /shop/[...parts]  /docs/[[...path]]  /api/health
/                  page / {}
/about             page /about {}
/blog              page /blog {}
/blog/hello-world  page /blog/[slug] {"slug":"hello-world"}
/shop/a/b/c        page /shop/[...parts] {"parts":["a","b","c"]}
/docs              page /docs/[[...path]] {"path":[]}
/docs/x/y          page /docs/[[...path]] {"path":["x","y"]}
/api/health        handler /api/health {}
/nope              404 not-found
```

??? success "Show a solution"

    ```
    // App Router style file-system routes. Files are paths like "app/blog/[slug]/page.tsx".
    const files = [
      "app/page.tsx",
      "app/(marketing)/about/page.tsx",
      "app/blog/page.tsx",
      "app/blog/[slug]/page.tsx",
      "app/shop/[...parts]/page.tsx",
      "app/docs/[[...path]]/page.tsx",
      "app/api/health/route.ts",
    ];

    function toRoute(file) {
      const segs = file.replace(/^app\//, "").split("/");
      const leaf = segs.pop();                                  // page.tsx or route.ts
      const kept = segs.filter((s) => !(s.startsWith("(") && s.endsWith(")")));   // route groups vanish
      return { pattern: "/" + kept.join("/"), kind: leaf.startsWith("page") ? "page" : "handler", segs: kept };
    }

    const routes = files.map(toRoute);

    function match(url) {
      const parts = url.split("/").filter(Boolean);
      let best = null;
      for (const r of routes) {
        const params = {};
        let i = 0, ok = true;
        for (const [k, seg] of r.segs.entries()) {
          const optional = /^\[\[\.\.\.(.+)\]\]$/.exec(seg);
          const rest = /^\[\.\.\.(.+)\]$/.exec(seg);
          const dyn = /^\[(.+)\]$/.exec(seg);
          if (optional) { params[optional[1]] = parts.slice(i); i = parts.length; }
          else if (rest) { if (i >= parts.length) { ok = false; break; } params[rest[1]] = parts.slice(i); i = parts.length; }
          else if (dyn) { if (i >= parts.length) { ok = false; break; } params[dyn[1]] = parts[i++]; }
          else { if (parts[i] !== seg) { ok = false; break; } i++; }
        }
        if (ok && i === parts.length) {
          const specificity = r.segs.filter((s) => !s.startsWith("[")).length;   // static segments win
          if (!best || specificity > best.specificity) best = { pattern: r.pattern, kind: r.kind, params, specificity };
        }
      }
      return best ? `${best.kind} ${best.pattern} ${JSON.stringify(best.params)}` : "404 not-found";
    }

    console.log(routes.map((r) => r.pattern).join("  "));
    for (const u of ["/", "/about", "/blog", "/blog/hello-world", "/shop/a/b/c", "/docs", "/docs/x/y", "/api/health", "/nope"]) {
      console.log(u.padEnd(18), match(u));
    }
    ```

    Folders become URL segments, parenthesized groups disappear, brackets create params and the most specific route wins. This models the idea, not Next's actual router. Verified by running the solution with Node.

### Time-based and tag-based revalidation

Implement DataCache.get so entries are served from cache while fresh, refetched after `revalidate` seconds or after revalidateTag invalidates them, and counted as origin fetches. A fake clock drives the scenario.

Starter code:

```
// A tiny model of the Next.js data cache: time-based revalidation (ISR) plus on-demand tag invalidation.
class DataCache {
  constructor(clock) { this.clock = clock; this.entries = new Map(); this.fetches = 0; }

  async get(key, fetcher, { revalidate = Infinity, tags = [] } = {}) {
    // return { value, source } where source is "cache" while the entry is fresh (age < revalidate seconds and
    // not invalidated), "origin" for the first fetch of a key and "revalidated" when an existing entry is
    // refetched; store { value, at: now, tags, invalid: false } and count origin calls in this.fetches
    return { value: undefined, source: "origin" };
  }

  revalidateTag(tag) {
    for (const e of this.entries.values()) if (e.tags.includes(tag)) e.invalid = true;
  }
}

(async () => {
  const clock = { now: 0 };
  const cache = new DataCache(clock);
  let version = 1;
  const fetchPosts = async () => `posts v${version}`;
  const show = async (label, key, opts) => {
    const r = await cache.get(key, fetchPosts, opts);
    console.log(label.padEnd(30), r.source.padEnd(12), r.value);
  };

  await show("first request", "posts", { revalidate: 60, tags: ["posts"] });
  version = 2;
  clock.now = 30_000;
  await show("30s later (still fresh)", "posts", { revalidate: 60, tags: ["posts"] });
  clock.now = 61_000;
  await show("61s later (expired)", "posts", { revalidate: 60, tags: ["posts"] });
  version = 3;
  await show("immediately again", "posts", { revalidate: 60, tags: ["posts"] });
  cache.revalidateTag("posts");
  await show("after revalidateTag('posts')", "posts", { revalidate: 60, tags: ["posts"] });
  await show("force-cache forever", "config", { revalidate: Infinity });
  clock.now = 10_000_000;
  await show("much later, still cached", "config", { revalidate: Infinity });
  console.log("origin fetches:", cache.fetches);
})();
```

Expected output:

```text
first request                  origin       posts v1
30s later (still fresh)        cache        posts v1
61s later (expired)            revalidated  posts v2
immediately again              cache        posts v2
after revalidateTag('posts')   revalidated  posts v3
force-cache forever            origin       posts v3
much later, still cached       cache        posts v3
origin fetches: 4
```

??? success "Show a solution"

    ```
    // A tiny model of the Next.js data cache: time-based revalidation (ISR) plus on-demand tag invalidation.
    class DataCache {
      constructor(clock) { this.clock = clock; this.entries = new Map(); this.fetches = 0; }

      async get(key, fetcher, { revalidate = Infinity, tags = [] } = {}) {
        const e = this.entries.get(key);
        const now = this.clock.now;
        if (e && !e.invalid && now - e.at < revalidate * 1000) return { value: e.value, source: "cache" };
        const stale = e ? e.value : undefined;
        this.fetches++;
        const value = await fetcher();
        this.entries.set(key, { value, at: now, tags, invalid: false });
        return { value, source: e ? "revalidated" : "origin", stale };
      }

      revalidateTag(tag) {
        for (const e of this.entries.values()) if (e.tags.includes(tag)) e.invalid = true;
      }
    }

    (async () => {
      const clock = { now: 0 };
      const cache = new DataCache(clock);
      let version = 1;
      const fetchPosts = async () => `posts v${version}`;
      const show = async (label, key, opts) => {
        const r = await cache.get(key, fetchPosts, opts);
        console.log(label.padEnd(30), r.source.padEnd(12), r.value);
      };

      await show("first request", "posts", { revalidate: 60, tags: ["posts"] });
      version = 2;
      clock.now = 30_000;
      await show("30s later (still fresh)", "posts", { revalidate: 60, tags: ["posts"] });
      clock.now = 61_000;
      await show("61s later (expired)", "posts", { revalidate: 60, tags: ["posts"] });
      version = 3;
      await show("immediately again", "posts", { revalidate: 60, tags: ["posts"] });
      cache.revalidateTag("posts");
      await show("after revalidateTag('posts')", "posts", { revalidate: 60, tags: ["posts"] });
      await show("force-cache forever", "config", { revalidate: Infinity });
      clock.now = 10_000_000;
      await show("much later, still cached", "config", { revalidate: Infinity });
      console.log("origin fetches:", cache.fetches);
    })();
    ```

    Next's data cache serves fresh entries, revalidates after a time window (ISR) and can be invalidated on demand by tag, so unnecessary origin fetches are avoided. This is a simplified model. Verified by running the solution with Node.

---

More free practice: the [Next.js Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
