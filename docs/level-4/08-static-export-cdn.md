---
description: "Export Next.js as static files with output 'export', learn which features it drops, host it anywhere, and set CDN cache headers correctly."
---

# 08 · Static Export & CDN Caching

Not every Next.js site needs a server. Documentation, blogs, portfolios and marketing
sites can be exported as plain HTML, CSS and JavaScript and hosted on any static file
host or object storage bucket behind a CDN — cheap, fast and nearly impossible to take
down. This lesson covers static export, what you give up, and how caching at a CDN works
for both exported and server-rendered apps.

## Enabling static export

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "export",
  trailingSlash: true,              // /posts/hello/ → posts/hello/index.html (friendlier for many hosts)
  images: { unoptimized: true },    // no image optimisation server in an export
};

export default nextConfig;
```

`next build` now writes an `out/` folder. We exported the Level 1 Markdown blog
(`dynamicParams = false`, `generateStaticParams` for posts) on Next.js 16.3.6:

```text
Route (app)
┌ ○ /_not-found
├ ○ /posts
└   /posts/[slug]
  ├ ● /posts/layouts-explained
  └ ● /posts/hello-world
```

```text
$ find out -name "*.html" | sort
out/404.html
out/404/index.html
out/_not-found/index.html
out/posts/hello-world/index.html
out/posts/index.html
out/posts/layouts-explained/index.html
$ du -sh out
832K	out
```

Serve it with anything: `npx serve out`, nginx, GitHub Pages, an S3 bucket with a CDN.
Client-side navigation, prefetching and Client Components all still work — the RSC
payloads are exported as files too.

## What you give up

Anything that needs a server at request time. From the Next.js 16 static export guide,
unsupported features include:

- Dynamic routes with `dynamicParams: true`, or without `generateStaticParams`
- Route Handlers that depend on the request; `cookies()`
- `redirects`, `rewrites` and `headers` in `next.config.ts`
- Proxy
- ISR and on-demand revalidation
- Image optimisation with the default loader
- Draft Mode, Server Actions, intercepting routes

Server Components still work — they run **at build time**. So reading Markdown files or
calling a CMS during the build is fine; per-user content isn't.

Workarounds for common needs:

| Need | Static-export approach |
|---|---|
| Redirects | Configure them on the host/CDN, or meta-refresh pages |
| Forms | A third-party form endpoint or a separate API |
| Search | Build a search index at build time and search client-side |
| Images | `unoptimized`, pre-sized images, or a custom `loader` pointing to an image CDN |
| Fresh content | Rebuild and redeploy on content change (webhook-triggered CI) |

## CDN caching fundamentals

Whether you export or run a server, a CDN in front caches responses according to HTTP
headers:

- `Cache-Control: public, max-age=31536000, immutable` — cache for a year, never
  revalidate. Correct **only** for content-hashed URLs. Next.js sends this for
  `/_next/static/*`; we observed exactly this header on a CSS chunk from `next start`.
- `Cache-Control: public, max-age=0` (+ `ETag`) — the CDN/browser must revalidate; cheap
  304s. We observed this on a `public/` file from `next start`.
- `s-maxage=N` — lifetime for shared caches (CDNs) only; `stale-while-revalidate=N` lets
  them serve stale while refetching.
- `private` / `no-store` — never cache at the CDN (personalised pages).

For a static export, configure the host so HTML gets a short or revalidating policy and
`/_next/static/` gets the immutable one:

```nginx
location /_next/static/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
}
location / {
    add_header Cache-Control "public, max-age=0, must-revalidate";
    try_files $uri $uri/index.html =404;
}
```

## CDNs in front of `next start`

For a server-rendered app, Next.js sets `Cache-Control` on responses according to how
each route was rendered — prerendered and ISR pages get `s-maxage`/stale-while-revalidate
style headers matching their revalidation time, dynamic pages get `private, no-cache,
no-store`. Observed from `next start` on 16.3.6: the static `/posts` page returned
`Cache-Control: s-maxage=31536000`, and the dynamic `/tasks` page returned
`Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate`. A CDN that
respects these headers will cache static and ISR pages at the
edge automatically. Two cautions:

- **On-demand revalidation** (`revalidatePath`, tags) clears Next.js's cache, not the
  CDN's. With a CDN, either keep CDN lifetimes short, or purge the CDN in the same code
  path (most CDNs have a purge API), or use a platform that integrates both.
- **Never let a CDN cache a response that set a cookie** or depends on one. Check your
  CDN's default behaviour for `Set-Cookie`.

The Next.js docs include a "CDN caching" guide for your version with the exact headers
per rendering mode.

## How It Actually Works

With `output: "export"`, the build renders every route in the static prerender mode and,
instead of keeping results in `.next/` for a server, writes them out as files: one
`index.html` (with `trailingSlash`) plus RSC payload files per route, and copies
`public/` and `/_next/static/`. Any route that can't be fully prerendered — because it
uses a request-time API or an unlisted dynamic param — is a build error rather than a
silently dynamic route, since there will be no server to fall back to. On the client,
the Next.js router fetches RSC payloads as static files during navigation, which is why
Link and prefetching keep working without a server. A CDN then simply maps URL → file,
using `Cache-Control` to decide how long each copy is valid.

## Common mistakes

- **Using a Server Action or `cookies()`** and discovering it at build time — design for
  export from the start.
- **Forgetting `images.unoptimized`** (or a custom loader) — the build fails.
- **Immutable caching on HTML** — users are stuck on old pages.
- **Relying on `next.config.ts` redirects** in an export — they're ignored/unsupported.
- **Purging nothing after revalidating** when a CDN caches server-rendered pages.

## Exercise

1. Export your Level 1 blog with `output: "export"` and serve `out/` with `npx serve`.
   Check that client-side navigation between posts works.
2. Add a Route Handler that reads `request.headers` and try to export. Read the error,
   then make the route static (no request dependence) and export again.
3. Deploy the `out/` folder to a free static host and inspect the response headers for
   an HTML page and a `/_next/static/` file. Adjust the host's configuration to match
   the policy in this lesson.
