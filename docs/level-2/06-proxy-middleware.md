---
description: "Next.js Proxy (renamed from Middleware in v16): run code before routing to redirect, rewrite, set headers and gate routes, with matchers and its limits."
---

# 06 · Proxy (formerly Middleware)

Sometimes you want to act on a request **before** Next.js decides which page to
render: send visitors from an old URL to a new one, route `/` to a different page by
country, add a security header everywhere, or bounce signed-out users away from
`/admin` without rendering anything. That's what Proxy is for.

!!! note "Renamed in Next.js 16"
    Until version 15 this feature was called **Middleware** and lived in
    `middleware.ts` with an exported `middleware` function. Next.js 16 deprecated that
    name and renamed it **Proxy**: file `proxy.ts`, function `proxy`. Behaviour is the
    same; a codemod does the rename
    (`npx @next/codemod@canary middleware-to-proxy .`). Also new in 16: Proxy runs on
    the **Node.js runtime** by default, and the `runtime` option is not allowed in the
    file. Tutorials that say "middleware runs on the Edge runtime" describe older
    versions.

## A first proxy

Create `proxy.ts` in the project root (next to `app/`, or inside `src/` if you use it):

```ts title="proxy.ts"
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  if (request.nextUrl.pathname.startsWith("/admin") && !request.cookies.has("session")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
  const res = NextResponse.next();
  res.headers.set("x-request-path", request.nextUrl.pathname);
  return res;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico).*)"],
};
```

Observed against `next start` with this file:

```bash
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" localhost:3111/admin
curl -s -i localhost:3111/api/posts | grep -i x-request-path
```

```text
307 http://localhost:3111/login
x-request-path: /api/posts
```

The build output also listed `ƒ Proxy (Middleware)` under the route table — the
build's own confirmation that a proxy exists.

## What a proxy can return

| Return | Effect |
|---|---|
| `NextResponse.next()` | Continue to the matched route (optionally with modified headers) |
| `NextResponse.redirect(url)` | Send a 307 (or given status) to the browser; URL changes |
| `NextResponse.rewrite(url)` | Serve a different route **without** changing the URL |
| `new Response(...)` / `NextResponse.json(...)` | Answer directly; no route renders |

Rewrites are the powerful one. A/B testing, for example:

```ts title="proxy.ts (A/B excerpt)"
export function proxy(request: NextRequest) {
  if (request.nextUrl.pathname === "/") {
    let bucket = request.cookies.get("ab-home")?.value;
    if (bucket !== "a" && bucket !== "b") bucket = Math.random() < 0.5 ? "a" : "b";
    const res = NextResponse.rewrite(new URL(`/home-${bucket}`, request.url));
    res.cookies.set("ab-home", bucket, { maxAge: 60 * 60 * 24 * 30, path: "/" });
    return res;
  }
  return NextResponse.next();
}
```

Visitors always see `/` in the address bar, and each keeps their bucket.

## Passing information to the app

Proxy runs separately from rendering, so you can't share variables with pages. Use
request headers instead — set them on the **request** being forwarded:

```ts
const requestHeaders = new Headers(request.headers);
requestHeaders.set("x-country", request.headers.get("x-vercel-ip-country") ?? "unknown");
return NextResponse.next({ request: { headers: requestHeaders } });
```

A page can then read it with `(await headers()).get("x-country")`. (Which geo header
exists depends on your host or CDN; the one above is platform-specific.)

## Matchers

Without a `matcher`, Proxy runs on **every** request, including static files and image
optimisation. That wastes time and can break things — an auth redirect that also
catches `/_next/static/...` will stop CSS from loading on your login page. Always
restrict it:

```ts
export const config = {
  matcher: [
    "/admin/:path*",
    "/account/:path*",
  ],
};
```

Matchers accept path patterns (`:param`, `:path*`), regular-expression groups, and
objects with `has`/`missing` conditions on headers or cookies. They must be static
values the build can read.

## What Proxy is *not* for

- **Not the place for full authorisation.** Proxy is good for an optimistic check
  ("is there a session cookie?"). It shouldn't query your database on every request, and
  it must not be your only check: pages, Server Actions and Route Handlers must verify
  the session themselves (lesson 07). This isn't theoretical — in 2025 a vulnerability
  (CVE-2025-29927) let crafted requests skip Middleware entirely in affected versions;
  apps that relied on it alone for auth were exposed.
- **Not for slow work.** It's in the path of every matched request.
- **Not for simple static redirects.** Use `redirects()` in `next.config.ts` — it's
  declarative and runs without your code.

## Worked example: locale redirect

Send `/` to `/en` or `/de` based on the browser's `Accept-Language`, but only for
paths without a locale:

```ts title="proxy.ts"
import { NextResponse, type NextRequest } from "next/server";

const locales = ["en", "de"] as const;
const defaultLocale = "en";

function pickLocale(header: string | null): string {
  if (!header) return defaultLocale;
  for (const part of header.split(",")) {
    const tag = part.split(";")[0].trim().toLowerCase().slice(0, 2);
    if ((locales as readonly string[]).includes(tag)) return tag;
  }
  return defaultLocale;
}

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const hasLocale = locales.some((l) => pathname === `/${l}` || pathname.startsWith(`/${l}/`));
  if (hasLocale) return NextResponse.next();

  const locale = request.cookies.get("locale")?.value ?? pickLocale(request.headers.get("accept-language"));
  const url = request.nextUrl.clone();
  url.pathname = `/${locale}${pathname}`;
  return NextResponse.redirect(url);
}

export const config = { matcher: ["/((?!api|_next/static|_next/image|favicon.ico|robots.txt|sitemap.xml).*)"] };
```

Level 3 · 04 builds the `[lang]` pages this points to.

## How It Actually Works

The build compiles `proxy.ts` into its own bundle, separate from your routes, along
with the matcher list. At request time the server tests the URL against the matchers
first; on a match it builds a `NextRequest`, invokes your function, and interprets the
returned response: `next()` continues the normal routing pipeline (with any header
changes applied), `rewrite()` restarts routing with a new internal URL, and a redirect
or direct response short-circuits everything. Because it's a separate bundle — and on
some hosting platforms is deployed to a separate location closer to users — module
state isn't shared with your pages, which is why information must travel through
headers, cookies or the URL.

## Common mistakes

- **No matcher**, so Proxy runs for every asset.
- **Keeping `middleware.ts` after upgrading** — it still works for now but is
  deprecated; run the codemod.
- **Setting `export const runtime` in `proxy.ts`** — an error in 16.
- **Relying on Proxy alone for access control.**
- **Redirect loops**: redirecting `/login` to `/login` because the matcher includes it.
- **Setting headers on the response when the page needs them** — use
  `NextResponse.next({ request: { headers } })` for the page to read them.

## Exercise

1. Add a proxy that sets `x-request-id` (use `crypto.randomUUID()`) on every forwarded
   request, and log it from a Server Component via `headers()`.
2. Protect `/admin/:path*` with a cookie check and a redirect to `/login?next=<path>`.
   Make sure `/login` itself isn't matched.
3. Move a simple `/old → /new` redirect from Proxy into `next.config.ts` `redirects()`
   and compare the two with `curl -i`.
