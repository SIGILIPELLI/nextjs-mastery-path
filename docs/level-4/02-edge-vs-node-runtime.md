---
description: "Node.js vs Edge runtime in Next.js: what each provides, why runtime 'edge' is deprecated in v16, Proxy on Node, and edge as a location."
---

# 02 · Node.js vs Edge Runtime

For several years Next.js offered two server runtimes, and a lot of articles
recommend moving code "to the edge" for speed. In Next.js 16 the picture changed: the
Node.js runtime is the default everywhere and the Edge runtime option for routes is
deprecated. This lesson explains what the two runtimes are, why the recommendation
shifted, and how to think about "edge" as a *location* rather than a runtime.

## The two runtimes

| | Node.js runtime | Edge runtime |
|---|---|---|
| APIs | Full Node.js (`fs`, `net`, `crypto`, native modules, all npm packages) | Web-standard subset (`fetch`, `Request`, `Response`, Web Crypto, streams) |
| Startup | Heavier process | Very light isolates (V8 isolates, not full processes) |
| Limits | Whatever your server allows | Small code-size limits, no file system, restricted packages |
| Where it ran (typically) | Your server or a regional serverless function | Platform's globally distributed network |

The Edge runtime was attractive for small, latency-sensitive code — middleware, simple
API routes — because it could run close to users with fast cold starts. The costs:
many npm packages didn't work, database drivers needed special HTTP-based variants, and
running compute near the user doesn't help if every request then travels to a database
in one region anyway.

## What changed in Next.js 16

From the documentation shipped with Next.js 16:

- The route segment option `export const runtime = "edge"` is **deprecated**; the docs
  advise removing the `runtime` export from route files (the default is `"nodejs"`).
- **Proxy** (formerly Middleware) defaults to the Node.js runtime, and setting `runtime`
  in `proxy.ts` throws an error. Node.js support for Middleware had become stable in
  15.5 before this.
- Regions for functions on hosts that support it are chosen with `preferredRegion`,
  which previously required the Edge runtime on some platforms.

So for new code: **write for Node.js**. If you maintain code that exports
`runtime = "edge"`, plan to remove it and test that the route still works on Node
(usually it does, because Web APIs are available in Node too).

## "Edge" as a location

Separating two ideas clears up most confusion:

- **Runtime**: which JavaScript environment executes your code.
- **Location**: where in the world it runs.

You can get most of the latency benefit people wanted from "the edge" without an edge
*runtime*:

1. **Cache at the CDN.** Static pages, prerendered shells (PPR), `/_next/static/` assets
   and images served from CDN locations near users need no compute at all. This is the
   biggest win and works with any runtime.
2. **Put compute near the data.** A server in the same region as your database, doing
   three queries at 1 ms each, beats an edge function doing three queries at 80 ms each
   across the world.
3. **Stream.** Send the shell immediately (Level 3 · 02) so the user sees something while
   the regional server works.

## Worked example: auditing a mixed app

Suppose an app written for Next.js 14 has:

```ts title="app/api/geo/route.ts (old)"
export const runtime = "edge";

export async function GET(request: Request) {
  const country = request.headers.get("x-vercel-ip-country") ?? "unknown";
  return Response.json({ country });
}
```

```ts title="middleware.ts (old)"
export const config = { runtime: "experimental-edge", matcher: ["/account/:path*"] };
export function middleware() { /* cookie check */ }
```

Migration on 16:

1. Rename `middleware.ts` to `proxy.ts` (`npx @next/codemod@canary middleware-to-proxy .`),
   rename the function to `proxy`, and remove the `runtime` key — it's not allowed.
2. Remove `export const runtime = "edge"` from the route handler. Its code only uses Web
   APIs, which Node provides, so it works unchanged.
3. Run `next build` and the Playwright suite. Check the deploy platform's notes on
   regions if latency matters, and whether the platform exposes the same geo headers
   to Node functions (header names are platform-specific).

## When the runtime still matters

- **Native modules and Node APIs** (`better-sqlite3`, `sharp`, `fs`) require Node —
  another reason Node is the sensible default.
- **Platform-specific edge functions** outside Next.js (a CDN's own worker product) are
  still useful for things like header rewriting or bot filtering in front of your app.
  Those are infrastructure choices, not Next.js runtime choices.
- **Libraries that detect runtime**: some packages ship different builds for "edge-light"
  and Node conditions. On Node you get the full build.

## How It Actually Works

A Node.js runtime route is compiled into a server bundle executed inside a regular
Node process (a long-running `next start` server or a serverless function that boots
Node). An Edge runtime route was compiled with a restricted set of globals and bundled
into a single file that platforms loaded into V8 isolates — lightweight sandboxes in an
already-running process, which is why cold starts were fast and why Node built-ins were
missing. Next.js enforced the restriction at build time by resolving imports with
"edge" export conditions and erroring on disallowed modules. Removing that second
target simplifies the framework (one module graph for server code) and your app (one
set of compatible dependencies).

## Common mistakes

- **Equating "edge" with "fast".** Latency is dominated by data location and caching.
- **Keeping `runtime = "edge"` after upgrading** and ignoring the deprecation.
- **Setting `runtime` in `proxy.ts`** — an error in 16.
- **Assuming platform geo headers exist everywhere** — they're provider-specific.
- **Using HTTP-only database drivers you adopted for the edge** when a native driver
  would now be simpler — re-evaluate after migrating.

## Exercise

1. Search your codebase (or a public Next.js 13/14 example repo) for `runtime`. List each
   occurrence and what you'd change for 16.
2. Measure: deploy (or run locally) a page that does three sequential 50 ms "queries" and
   compare total time when the "database" is simulated as near (1 ms added latency) vs
   far (80 ms). Explain what this means for edge compute.
3. Write a short decision record for your team: "Default runtime and where we cache",
   with the reasoning from this lesson.
