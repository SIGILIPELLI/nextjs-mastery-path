---
description: "Improve Next.js Core Web Vitals: diagnose LCP, INP and CLS, shrink client JavaScript, lazy-load widgets and report field data with useReportWebVitals."
---

# 06 · Performance & Core Web Vitals

"Is the site fast?" needs a definition. Google's **Core Web Vitals** provide one that
maps to what users feel, and they're used as a search ranking signal. This lesson
explains each metric, what in a Next.js app usually hurts it, and how to measure in
the lab and in the field.

## The three metrics

| Metric | Measures | "Good" threshold (at the 75th percentile of visits) |
|---|---|---|
| **LCP** — Largest Contentful Paint | When the main content (largest image or text block) appears | ≤ 2.5 s |
| **INP** — Interaction to Next Paint | How quickly the page responds visually to clicks, taps and keys, over the whole visit | ≤ 200 ms |
| **CLS** — Cumulative Layout Shift | How much visible content jumps around unexpectedly | ≤ 0.1 |

INP replaced First Input Delay as a Core Web Vital in 2024. Thresholds are published on
web.dev; they've been stable, but check there for the current definitions.

## LCP: get the main content on screen

Typical causes and Next.js fixes:

- **Slow server response.** Prefer static or cached rendering; stream slow parts
  (lesson 02) so the shell and main content aren't blocked by a side panel.
- **Hero image discovered late or lazy-loaded.** Use `next/image` with
  `fetchPriority="high"` (or `loading="eager"`) on the LCP image and a correct `sizes`.
- **Web font blocking text.** `next/font` self-hosts fonts and uses metric-matched
  fallbacks, so text paints immediately.
- **Main content rendered on the client** after a fetch. Render it in a Server
  Component.

## INP: keep the main thread free

INP is about JavaScript. Every Client Component adds code to download, parse and
hydrate, and heavy event handlers block the next paint.

- **Ship less client JS.** Keep `"use client"` at the leaves (Level 1 · 05). A static
  article page should need very little JavaScript beyond the framework runtime.
- **Lazy-load heavy client-only widgets** with `next/dynamic`:

```tsx title="app/report/ChartLoader.tsx"
"use client";

import dynamic from "next/dynamic";

const Chart = dynamic(() => import("./Chart"), {
  ssr: false, // the chart library touches `window`; render it only in the browser
  loading: () => <div style={{ height: 320 }} aria-busy="true" />,
});

export function ChartLoader(props: { data: number[] }) {
  return <Chart {...props} />;
}
```

(`ssr: false` is only allowed inside a Client Component, hence the small wrapper.)

- **Keep urgent updates urgent.** Wrap expensive, non-urgent state updates in
  `startTransition` or use `useDeferredValue` so typing stays responsive.
- **Defer third-party scripts** with `next/script` and `strategy="lazyOnload"` or
  `"afterInteractive"` — analytics and chat widgets are frequent INP culprits.
- **Consider the React Compiler** (`reactCompiler: true`, supported since Next.js 16)
  to reduce unnecessary re-renders automatically; measure before and after, since the
  benefit depends on the app.

## CLS: reserve space

- Images: always have dimensions — `next/image` enforces it.
- Fonts: `next/font`'s adjusted fallbacks prevent reflow when the font loads.
- Loading states: skeletons sized like the final content; `min-height` on containers.
- Late banners (cookie notices, promos): overlay them or reserve their space; don't push
  content down after first paint.

## Measuring

**Lab** (a controlled run): Lighthouse in Chrome DevTools, against a production build
(`next build && next start`), ideally with mobile throttling. Useful for debugging, but
one synthetic run is not what users experience.

**Field** (real users): collect metrics from real visits. Next.js exposes them via
`useReportWebVitals`:

```tsx title="app/_components/WebVitals.tsx"
"use client";

import { useReportWebVitals } from "next/web-vitals";

export function WebVitals() {
  useReportWebVitals((metric) => {
    const body = JSON.stringify({
      name: metric.name,       // "LCP", "INP", "CLS", "FCP", "TTFB"
      value: metric.value,
      rating: metric.rating,   // "good" | "needs-improvement" | "poor"
      id: metric.id,
      page: location.pathname,
    });
    navigator.sendBeacon?.("/api/vitals", body) || fetch("/api/vitals", { body, method: "POST", keepalive: true });
  });
  return null;
}
```

Render `<WebVitals />` once in the root layout, and add a Route Handler at
`/api/vitals` that stores the numbers. Look at the 75th percentile per page, not the
average. Public field data for popular sites is also available through Chrome's UX
Report (CrUX).

## Bundle analysis

To see what's in your client JavaScript, use a bundle analyser. The Next.js 16.3 CLI
includes an experimental analyser for Turbopack builds (`next experimental-analyze`,
which opens an interactive web UI); the older
`@next/bundle-analyzer` plugin works with webpack builds. Look for large libraries
imported by Client Components — date libraries, icon packs imported whole, markdown
renderers — and move that work to the server where possible. For packages with many
exports, `optimizePackageImports` in `next.config.ts` helps the bundler import only
what you use.

## Worked example: a slow marketing page

Symptoms from Lighthouse on a production build: LCP 4 s, CLS 0.25. Investigation:

1. The LCP element is the hero image — it was lazy-loaded and had no `sizes`, so a
   2400-px file downloaded on phones. Fix: `fetchPriority="high"`, `sizes="100vw"`.
2. The headline font came from a `<link>` to a font CDN. Fix: `next/font`.
3. A testimonials carousel mounted after a client fetch and pushed the footer down.
   Fix: render testimonials in a Server Component; reserve carousel height.
4. The whole page was a Client Component because the carousel needed state. Fix: page
   back to a Server Component; only the carousel is client.

Re-measure after each change so you know which fix mattered.

## How It Actually Works

The browser exposes these metrics through the `PerformanceObserver` API
(`largest-contentful-paint`, `layout-shift`, `event` entries). `useReportWebVitals`
wraps Google's `web-vitals` logic, which observes those entries, applies the official
rules (for example, CLS groups shifts into "session windows" and takes the worst;
INP takes a high percentile of interaction latencies over the visit), and reports a
final value when the page is hidden. On the Next.js side, what matters is the critical
path: HTML (server time) → CSS and the LCP resource → JavaScript for hydration. Server
Components shorten the last step by removing code from it entirely; streaming shortens
the first; `next/image` and `next/font` shorten the middle.

## Common mistakes

- **Measuring `next dev`.** Always a production build.
- **Optimising the Lighthouse score instead of field data.**
- **Lazy-loading the LCP image.**
- **Importing a heavy library into a Client Component** for something the server
  could do.
- **Averaging metrics.** Use the 75th percentile, per page type.

## Exercise

1. Run Lighthouse (mobile) on your Level 2 project's production build and record LCP,
   CLS and total blocking time.
2. Add the `WebVitals` reporter and a `/api/vitals` handler that logs to the console.
   Click around and watch INP values arrive when you switch tabs.
3. Find the largest client-side dependency in your app with a bundle analyser, and
   either lazy-load it or move its work to the server. Measure again.
