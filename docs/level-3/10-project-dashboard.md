---
description: "Level 3 project: a Next.js analytics dashboard with streamed panels, date range in the URL, and page details in an intercepted dialog."
---

# 10 · Project — Streaming Analytics Dashboard

Dashboards are where rendering decisions matter most: several data sources of very
different speeds, a URL people share with colleagues, and drill-downs that should feel
instant but still be linkable. This project combines streaming (lesson 02),
URL state (Level 2 · 09), parallel and intercepting routes (lesson 03), SEO-neutral
private pages (lesson 05) and accessibility-minded markup.

It was built with `next build` on Next.js 16.3.6, measured with `curl`, and clicked
through in a browser; outputs are from those runs. The data layer is deterministic fake
data with artificial delays, so the project needs no analytics service — swap in real
queries later.

## What you'll build

- `/analytics?range=7d|30d` — a total-views panel with a sparkline (fast) and a
  top-pages table (slow), each streaming independently.
- Clicking a page in the table opens its details in a **dialog** over the dashboard,
  at the URL `/analytics/pages/<slug>`.
- Opening that URL directly (or refreshing) shows a full detail page.
- Unknown slugs return 404.

## File layout

```text
app/analytics/
├── layout.tsx                       ← renders children + @drawer slot
├── page.tsx                         ← the dashboard
├── _components/
│   ├── Sparkline.tsx
│   └── PageDetail.tsx               ← shared by the full page and the dialog
├── pages/[slug]/page.tsx            ← full detail page
└── @drawer/
    ├── default.tsx                  ← null when nothing is intercepted
    └── (.)pages/[slug]/
        ├── page.tsx                 ← intercepted: detail inside a dialog
        └── Drawer.tsx
lib/analytics.ts
```

## 1. Data layer

```ts title="lib/analytics.ts"
import "server-only";
import { cache } from "react";

export type Range = "7d" | "30d";
export type PageStat = { slug: string; views: number; avgSeconds: number };

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

// Deterministic fake data so the demo needs no external service.
function seeded(n: number) {
  const x = Math.sin(n) * 10000;
  return x - Math.floor(x);
}

export const getTotals = cache(async (range: Range) => {
  await sleep(300);
  const days = range === "7d" ? 7 : 30;
  const views = Array.from({ length: days }, (_, i) => Math.round(800 + seeded(i + 1) * 400));
  return { views: views.reduce((a, b) => a + b, 0), daily: views };
});

export const getTopPages = cache(async (range: Range): Promise<PageStat[]> => {
  await sleep(1200);
  const slugs = ["pricing", "docs-install", "blog-caching", "about", "changelog"];
  const factor = range === "7d" ? 1 : 4;
  return slugs
    .map((slug, i) => ({ slug, views: Math.round((5 - i) * 310 * factor * (0.8 + seeded(i) * 0.4)), avgSeconds: Math.round(30 + seeded(i + 9) * 90) }))
    .sort((a, b) => b.views - a.views);
});

export const getPageDetail = cache(async (slug: string) => {
  await sleep(400);
  const pages = await getTopPages("30d");
  const page = pages.find((p) => p.slug === slug);
  if (!page) return null;
  return { ...page, referrers: [{ source: "search", share: 0.52 }, { source: "direct", share: 0.31 }, { source: "social", share: 0.17 }] };
});
```

## 2. A server-rendered sparkline

No chart library, no client JavaScript — an SVG computed on the server:

```tsx title="app/analytics/_components/Sparkline.tsx"
export function Sparkline({ values, label }: { values: number[]; label: string }) {
  const w = 240, h = 48, max = Math.max(...values), min = Math.min(...values);
  const pts = values.map((v, i) => `${(i / (values.length - 1)) * w},${h - ((v - min) / (max - min || 1)) * h}`).join(" ");
  return (
    <svg width={w} height={h} viewBox={`0 0 ${w} ${h}`} role="img" aria-label={label}>
      <polyline points={pts} fill="none" stroke="currentColor" strokeWidth="2" />
    </svg>
  );
}
```

## 3. The dashboard page

```tsx title="app/analytics/page.tsx"
import Link from "next/link";
import { Suspense } from "react";
import { connection } from "next/server";
import { getTopPages, getTotals, type Range } from "@/lib/analytics";
import { Sparkline } from "./_components/Sparkline";

export const metadata = { title: "Analytics" };

function parseRange(v: string | string[] | undefined): Range {
  return v === "30d" ? "30d" : "7d";
}

async function Totals({ range }: { range: Range }) {
  await connection();
  const t = await getTotals(range);
  return (
    <section aria-labelledby="totals-h">
      <h2 id="totals-h">Page views</h2>
      <p style={{ fontSize: 32, margin: 0 }}>{t.views.toLocaleString("en-US")}</p>
      <Sparkline values={t.daily} label={`Daily views over the last ${range}`} />
    </section>
  );
}

async function TopPages({ range }: { range: Range }) {
  await connection();
  const pages = await getTopPages(range);
  return (
    <section aria-labelledby="top-h">
      <h2 id="top-h">Top pages</h2>
      <table>
        <thead><tr><th scope="col">Page</th><th scope="col">Views</th><th scope="col">Avg. time</th></tr></thead>
        <tbody>
          {pages.map((p) => (
            <tr key={p.slug}>
              <td><Link href={`/analytics/pages/${p.slug}`}>{p.slug}</Link></td>
              <td>{p.views.toLocaleString("en-US")}</td>
              <td>{p.avgSeconds}s</td>
            </tr>
          ))}
        </tbody>
      </table>
    </section>
  );
}

export default async function AnalyticsPage({ searchParams }: PageProps<"/analytics">) {
  const range = parseRange((await searchParams).range);
  return (
    <main>
      <h1>Analytics</h1>
      <nav aria-label="Date range">
        <Link href="/analytics?range=7d" aria-current={range === "7d" ? "true" : undefined}>7 days</Link> ·{" "}
        <Link href="/analytics?range=30d" aria-current={range === "30d" ? "true" : undefined}>30 days</Link>
      </nav>
      <Suspense key={`t-${range}`} fallback={<p style={{ height: 110 }}>Loading totals…</p>}>
        <Totals range={range} />
      </Suspense>
      <Suspense key={`p-${range}`} fallback={<p style={{ height: 220 }}>Loading top pages…</p>}>
        <TopPages range={range} />
      </Suspense>
    </main>
  );
}
```

Two details worth pausing on:

- **`key` on each `<Suspense>`**: when the range changes, the key changes, so React
  treats it as a new boundary and shows the fallback again instead of leaving stale
  numbers on screen while the new ones load.
- **Fallbacks with fixed heights** keep the table from pushing content around (CLS).

## 4. Shared detail component, two entry points

```tsx title="app/analytics/_components/PageDetail.tsx"
import { notFound } from "next/navigation";
import { getPageDetail } from "@/lib/analytics";

export async function PageDetail({ slug }: { slug: string }) {
  const d = await getPageDetail(slug);
  if (!d) notFound();
  return (
    <div>
      <h2>/{d.slug}</h2>
      <p>{d.views.toLocaleString("en-US")} views (30 days), average {d.avgSeconds}s on page</p>
      <h3>Referrers</h3>
      <ul>{d.referrers.map((r) => <li key={r.source}>{r.source}: {Math.round(r.share * 100)}%</li>)}</ul>
    </div>
  );
}
```

```tsx title="app/analytics/pages/[slug]/page.tsx"
import Link from "next/link";
import { PageDetail } from "../../_components/PageDetail";

export default async function PageDetailPage({ params }: PageProps<"/analytics/pages/[slug]">) {
  const { slug } = await params;
  return (
    <main>
      <p><Link href="/analytics">← Back to analytics</Link></p>
      <PageDetail slug={slug} />
    </main>
  );
}
```

## 5. The intercepted dialog

```tsx title="app/analytics/layout.tsx"
export default function AnalyticsLayout({ children, drawer }: LayoutProps<"/analytics">) {
  return (<>{children}{drawer}</>);
}
```

```tsx title="app/analytics/@drawer/default.tsx"
export default function Default() {
  return null;
}
```

```tsx title="app/analytics/@drawer/(.)pages/[slug]/Drawer.tsx"
"use client";

import { useRouter } from "next/navigation";
import { useEffect, useRef } from "react";

export function Drawer({ children }: { children: React.ReactNode }) {
  const router = useRouter();
  const ref = useRef<HTMLDialogElement>(null);
  useEffect(() => {
    const dialog = ref.current;
    if (dialog && !dialog.open) dialog.showModal();
  }, []);
  return (
    <dialog ref={ref} onClose={() => router.back()} aria-label="Page details" style={{ marginRight: 0, height: "100vh", maxHeight: "100vh" }}>
      {children}
      <form method="dialog"><button>Close</button></form>
    </dialog>
  );
}
```

```tsx title="app/analytics/@drawer/(.)pages/[slug]/page.tsx"
import { Suspense } from "react";
import { PageDetail } from "../../../_components/PageDetail";
import { Drawer } from "./Drawer";

export default async function DrawerPage({ params }: PageProps<"/analytics/pages/[slug]">) {
  const { slug } = await params;
  return (
    <Drawer>
      <Suspense fallback={<p>Loading details…</p>}>
        <PageDetail slug={slug} />
      </Suspense>
    </Drawer>
  );
}
```

The native `<dialog>` with `showModal()` gives focus trapping, Escape to close and an
inert background for free. Both Escape and the Close button (a `method="dialog"`
form) fire the `close` event, which navigates back.

## 6. Verify

Build output:

```text
├ ƒ /analytics
├ ƒ /analytics/(.)pages/[slug]
├ ƒ /analytics/pages/[slug]
```

Timing and streaming with `next start`:

```bash
curl -s -o /dev/null -w "ttfb=%{time_starttransfer}s total=%{time_total}s\n" "localhost:3111/analytics?range=30d"
curl -s -o /dev/null -w "%{http_code}\n" localhost:3111/analytics/pages/nope
```

```text
ttfb=0.081973s total=1.252658s
404
```

The first bytes (including both "Loading…" fallbacks) arrived in about 80 ms; the full
response completed after about 1.25 s — the slow top-pages query, not the sum of both
queries. The unknown slug returned a real 404 because nothing streamed before
`notFound()` (no `loading.tsx` above it).

In a browser: clicking "pricing" on `/analytics` changed the URL to
`/analytics/pages/pricing` while the "Top pages" table stayed on screen behind the
dialog; pressing Escape closed it and returned to `/analytics`. Loading
`/analytics/pages/pricing` directly rendered the full page with its referrer list.

## How It Actually Works

A request for `/analytics?range=30d` renders the page shell synchronously up to the two
Suspense boundaries, flushes it (the 80 ms first byte), and continues rendering both
panels concurrently. `getTotals` resolves after ~300 ms and its HTML is streamed and
swapped into place; `getTopPages` follows at ~1.2 s. On a click to a detail link, the
client router sends a navigation request carrying the current route state; the server
sees that `@drawer` has an intercepting route matching `pages/[slug]` *relative to*
`/analytics` and renders it into the slot while leaving `children` as the dashboard.
`PageDetail` is the same Server Component in both entry points, so there's one data path
and one rendering of the detail — only the frame differs.

## Common mistakes

- **One Suspense boundary around both panels** — the fast one waits for the slow one.
- **No `key` on boundaries that depend on search params** — stale values linger.
- **Forgetting `default.tsx` in the slot** — build error in Next.js 16.
- **Closing the dialog with `router.push`** — breaks back-button expectations.
- **Letting dashboards be indexed.** Add `robots: { index: false }` to the layout's
  metadata and require login in a real deployment.

## Exercise

1. Add a third panel, "Countries", with a 2-second delay and its own error boundary; make
   it throw on `?fail=countries` and confirm the other panels still render.
2. Replace the fake data with SQLite tables (Level 2 · 08) seeded with a week of events.
3. Protect `/analytics` with the session helper from Level 2 · 07 and add
   `robots: { index: false }`.
4. Write a Playwright test that opens the dialog, checks the URL, presses Escape and
   checks the URL again.
