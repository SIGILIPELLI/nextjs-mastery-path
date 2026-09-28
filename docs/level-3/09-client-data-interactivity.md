---
description: "When Next.js apps should fetch in the browser: polling, live updates, infinite scroll and per-user widgets with SWR or TanStack Query."
---

# 09 · Client-Side Data & Interactivity

Server Components should load most of your data. But some data is inherently a
browser concern: a notification count that changes while the page is open, an
infinite-scrolling feed, search-as-you-type suggestions, data from a browser-only API.
This lesson is about recognising those cases and handling them without undoing the
server-first architecture.

## When client fetching is the right call

| Situation | Why the server alone isn't enough |
|---|---|
| Data changes while the user watches (prices, job status, chat) | Server render is a snapshot |
| User-triggered loading of more items (infinite scroll) | Unknown in advance how much to render |
| Highly interactive widgets with frequent small queries (autocomplete) | Full route re-renders per keystroke are wasteful |
| Browser-only sources (geolocation, IndexedDB, device APIs) | Not available on the server |
| Third-party APIs the browser calls directly with a user token | The server shouldn't proxy them |

Everything else — the page's main content — stays in Server Components.

## Pattern 1: server first, client refresh

Render initial data on the server (fast first paint, SEO), then let the client keep it
fresh. With **SWR**:

```bash
npm install swr
```

```tsx title="app/jobs/[id]/page.tsx"
import { getJob } from "@/lib/jobs";
import { JobStatus } from "./JobStatus";

export default async function JobPage({ params }: PageProps<"/jobs/[id]">) {
  const { id } = await params;
  const job = await getJob(id); // server render has real data
  return (
    <main>
      <h1>Export #{id}</h1>
      <JobStatus id={id} initial={job} />
    </main>
  );
}
```

```tsx title="app/jobs/[id]/JobStatus.tsx"
"use client";

import useSWR from "swr";

type Job = { status: "queued" | "running" | "done" | "failed"; progress: number };

const fetcher = (url: string) =>
  fetch(url).then((r) => {
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return r.json() as Promise<Job>;
  });

export function JobStatus({ id, initial }: { id: string; initial: Job }) {
  const { data, error } = useSWR(`/api/jobs/${id}`, fetcher, {
    fallbackData: initial,
    refreshInterval: (latest) => (latest && (latest.status === "done" || latest.status === "failed") ? 0 : 2000),
  });
  if (error) return <p role="alert">Couldn’t refresh status.</p>;
  return (
    <p aria-live="polite">
      {data.status} — {data.progress}%
    </p>
  );
}
```

The page is useful immediately; the client polls a Route Handler every two seconds
until the job finishes, then stops. **TanStack Query** offers the same capabilities
(`initialData`, `refetchInterval`) with a different API and more tooling for complex
cache management; either is a fine choice.

## Pattern 2: server-side reload with `router.refresh()`

If the data is already rendered by Server Components and you only need it refreshed
occasionally, you don't need a client data library at all:

```tsx title="app/_components/AutoRefresh.tsx"
"use client";

import { useRouter } from "next/navigation";
import { useEffect } from "react";

export function AutoRefresh({ seconds }: { seconds: number }) {
  const router = useRouter();
  useEffect(() => {
    const id = setInterval(() => {
      if (document.visibilityState === "visible") router.refresh();
    }, seconds * 1000);
    return () => clearInterval(id);
  }, [router, seconds]);
  return null;
}
```

`router.refresh()` re-requests the current route's Server Components and merges the
result, preserving client state. Coarse, but simple — good for dashboards refreshed
every minute. Skip refreshes while the tab is hidden.

## Pattern 3: infinite scroll with a Server Action

A Server Action can return data (and even JSX) to a Client Component. For "load more":

```ts title="app/feed/actions.ts"
"use server";

import { listPosts } from "@/lib/posts";

export async function loadMore(cursor: string | null) {
  const page = await listPosts({ after: cursor, limit: 20 });
  return { items: page.items.map((p) => ({ id: p.id, title: p.title })), next: page.nextCursor };
}
```

```tsx title="app/feed/Feed.tsx"
"use client";

import { useState, useTransition } from "react";
import { loadMore } from "./actions";

type Item = { id: string; title: string };

export function Feed({ initial, cursor }: { initial: Item[]; cursor: string | null }) {
  const [items, setItems] = useState(initial);
  const [next, setNext] = useState(cursor);
  const [pending, start] = useTransition();
  return (
    <>
      <ul>{items.map((i) => <li key={i.id}>{i.title}</li>)}</ul>
      {next && (
        <button disabled={pending} onClick={() => start(async () => {
          const res = await loadMore(next);
          setItems((prev) => [...prev, ...res.items]);
          setNext(res.next);
        })}>
          {pending ? "Loading…" : "Load more"}
        </button>
      )}
    </>
  );
}
```

Caveat: Server Actions run **one at a time** per client and use POST (not cacheable).
For read-heavy, parallel or cacheable client fetching, a `GET` Route Handler with
SWR/TanStack Query is the better tool; the docs recommend actions for mutations.

## Real-time updates

For truly live data (chat, collaborative editing), polling becomes wasteful. Options:

- **Server-Sent Events** from a Route Handler returning a `ReadableStream` with
  `Content-Type: text/event-stream` — works on a long-running Node server; many
  serverless platforms limit request duration.
- **WebSockets** — `next start` doesn't provide a WebSocket server; run one alongside
  (a custom server or separate service) or use a hosted real-time provider.

Either way, the Client Component subscribes and updates local state, or calls
`router.refresh()` when told something changed.

## How It Actually Works

SWR and TanStack Query keep a client-side cache keyed by request (the URL). A hook
returns the cached value immediately (or `fallbackData` from the server), then
revalidates in the background — on mount, on window focus, on an interval — and
re-renders subscribers when data changes. This is the stale-while-revalidate idea from
server caching, applied in the browser. `router.refresh()` is a different mechanism:
it asks the Next.js server to re-render the current route's Server Components and
reconciles the new RSC payload, so data logic stays on the server. Choosing between them
is choosing where the data-fetching code lives: in a Route Handler plus client hooks, or
in Server Components re-rendered on demand.

## Common mistakes

- **Fetching the main content client-side by habit** — slower first paint, worse SEO.
- **Polling with no stop condition** or while the tab is hidden.
- **No `fallbackData`/`initialData`**, so the client shows a spinner for data the server
  already had.
- **Using Server Actions for high-frequency reads.**
- **Duplicating server and client caches without an invalidation plan** — after a
  mutation, update or revalidate both (`mutate()` in SWR, `invalidateQueries` in
  TanStack Query, plus `revalidatePath` on the server).

## Exercise

1. Build the job-status page with a Route Handler that advances a fake job's progress
   on each call. Confirm polling stops at 100%.
2. Replace SWR with `AutoRefresh` + a Server Component. Compare code size and network
   traffic.
3. Implement "load more" with cursor-based pagination and make sure the button
   disappears on the last page.
