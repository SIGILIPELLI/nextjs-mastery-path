---
description: "Stream Next.js pages with React Suspense: independent loading boundaries, avoiding waterfalls, measured time-to-first-byte, and how chunks arrive."
---

# 02 · Suspense & Streaming

A page that needs three pieces of data — one fast, two slow — traditionally waits for
the slowest before sending anything. Streaming lets the server send the page in
pieces: the frame and fast parts first, each slow part as soon as it's ready. In the
App Router, the tool for this is React's `<Suspense>`.

## One page, independent sections

```tsx title="app/dashboard/page.tsx"
import { Suspense } from "react";
import { connection } from "next/server";

const wait = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function Revenue() {
  await connection();
  await wait(800); // stand-in for a slow query
  return <p>Revenue: 42</p>;
}

async function Signups() {
  await connection();
  await wait(1500);
  return <p>Signups: 7</p>;
}

export default function Dashboard() {
  return (
    <main>
      <h1>Dashboard</h1>
      <Suspense fallback={<p>Loading revenue…</p>}>
        <Revenue />
      </Suspense>
      <Suspense fallback={<p>Loading signups…</p>}>
        <Signups />
      </Suspense>
    </main>
  );
}
```

We built this and measured it against `next start` with `curl`:

```bash
curl -s -o /dev/null -w "ttfb=%{time_starttransfer}s total=%{time_total}s\n" localhost:3111/dashboard
```

```text
ttfb=0.008319s total=1.511174s
```

The first byte arrived in about **8 ms**; the response finished after about **1.5 s** —
the duration of the slowest section, not the sum (the two ran concurrently). The raw
response body contained both the "Loading revenue…" fallback and, later, "Revenue: 42"
and "Signups: 7": the fallbacks went out first and the content followed.

## `loading.tsx` vs `<Suspense>`

`loading.tsx` is a Suspense boundary around the **whole page** segment. Use it for
navigation feedback. Use explicit `<Suspense>` inside the page when sections load
independently, so a slow chart doesn't hide a fast summary. They combine well:
`loading.tsx` gives instant navigation, then inner boundaries reveal content piece by
piece.

## Designing boundaries

- **Group by what the user reads together.** Three boundaries for a card's title,
  body and footer create a jittery reveal; one boundary for the card is better.
- **Fallbacks should match the final size** (skeletons with fixed heights) to avoid
  layout shift.
- **Put boundaries around the slow thing, not the whole page.**
- **Nesting creates sequencing**: an outer boundary must resolve before inner fallbacks
  can show inside it.

## Avoiding waterfalls across components

Streaming doesn't fix waterfalls *inside* a component. If `Signups` awaited `getUser()`
and then `getSignups(user.team)`, that chain is sequential no matter what. Start
independent work early and pass promises down:

```tsx title="app/report/page.tsx"
import { Suspense, use } from "react";
import { getSummary, getDetails } from "@/lib/report";

export default function Report() {
  const detailsPromise = getDetails(); // started now, not awaited
  return (
    <main>
      <Summary />
      <Suspense fallback={<p>Loading details…</p>}>
        <Details promise={detailsPromise} />
      </Suspense>
    </main>
  );
}

async function Summary() {
  const s = await getSummary();
  return <p>{s.text}</p>;
}

function Details({ promise }: { promise: ReturnType<typeof getDetails> }) {
  const details = use(promise); // suspends until resolved
  return <pre>{JSON.stringify(details, null, 2)}</pre>;
}
```

The same pattern crosses the server/client boundary: pass a Promise from a Server
Component to a Client Component, and unwrap it there with `use()` inside a Suspense
boundary. The data starts loading on the server and streams to the client.

## Streaming and metadata, status codes and SEO

- Once streaming starts, the **status code is fixed** (Level 2 · 02 showed a
  `notFound()` inside a streamed segment returning 200 with `noindex`).
- `generateMetadata` that's slow can delay the head. Next.js may stream metadata
  separately for clients it knows handle it and wait for it for known bots (see the
  `htmlLimitedBots` config); keep metadata fetches fast and cached.
- Search engines execute streamed content, but fundamental content should not depend on
  a slow, failure-prone boundary.

## Errors inside streamed sections

An error in a streamed component is caught by the nearest error boundary. Put an
error boundary next to important Suspense boundaries so one failed widget shows
"couldn't load" instead of taking the page down — `error.tsx` at the segment level, or a
small client error-boundary component around individual sections.

## How It Actually Works

React's server renderer walks the tree. When a component suspends (its Promise is
pending) inside a `<Suspense>` boundary, React writes the boundary's fallback HTML
wrapped in a marker (a `<template>` with an ID) and carries on with the rest of the
page. The response is sent with chunked transfer encoding, so these bytes leave
immediately. When the suspended Promise resolves, React renders that subtree and writes
another chunk at the end of the stream: the finished HTML in a hidden `<div>` plus a
tiny inline `<script>` that moves it into the marker's place. Alongside the HTML, the
RSC payload for the new content is streamed too (that's the `self.__next_f.push(...)`
data you'll see in the raw response), so React can hydrate the section when its code is
ready. Everything happens over one HTTP response; no additional requests are made.

This also explains why `curl` shows both fallbacks *and* content in the body: curl
doesn't run the swap scripts, it just receives every chunk.

## Common mistakes

- **Awaiting slow data in the page component itself**, above the Suspense boundaries —
  nothing streams until it resolves.
- **Too many tiny boundaries** causing a popcorn effect.
- **Fallbacks with no dimensions**, causing layout shift.
- **Proxies or servers that buffer responses** (some reverse proxies do by default),
  destroying streaming. With nginx, for example, disable buffering for the app
  (`X-Accel-Buffering: no` or `proxy_buffering off`).
- **Assuming streaming fixes sequential awaits.**

## Exercise

1. Build the dashboard. Use `curl -N` to watch the chunks arrive, then view the page
   in a browser with network throttling.
2. Change it so both sections are inside one `<Suspense>`. Measure again and describe
   the difference in user experience, not just timing.
3. Refactor a page that has `await getA(); await getB();` at the top into two streamed
   sections, and measure time-to-first-byte before and after.
