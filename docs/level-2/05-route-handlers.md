---
description: "Build HTTP endpoints with Route Handlers: route.ts, Web Request and Response, params, status codes, signed webhooks, and when to use Server Actions."
---

# 05 · Route Handlers (API Endpoints)

Server Actions cover mutations from your own UI. But some clients aren't your UI: a
mobile app, a partner's integration, a payment provider's webhook, a cron job, an RSS
reader. For those you need a plain HTTP endpoint. In the App Router, that's a
`route.ts` file.

## A first handler

```ts title="app/api/posts/route.ts"
import { getPosts } from "@/lib/posts";

export async function GET() {
  return Response.json(await getPosts());
}

export async function POST(request: Request) {
  const body = await request.json();
  if (typeof body?.title !== "string") {
    return Response.json({ error: "title required" }, { status: 400 });
  }
  return Response.json({ ok: true, title: body.title }, { status: 201 });
}
```

Export a function named after each HTTP method you support: `GET`, `POST`, `PUT`,
`PATCH`, `DELETE`, `HEAD`, `OPTIONS`. Unsupported methods get `405 Method Not Allowed`.
The handlers use the standard Web `Request` and `Response` objects, so the knowledge
transfers to any modern runtime.

Tested against `next start` while writing this lesson:

```bash
curl -s localhost:3111/api/posts
curl -s -X POST -H 'content-type: application/json' -d '{}' localhost:3111/api/posts
```

```text
[{"slug":"hello-next","title":"Hello, Next.js","body":"First post.","date":"2026-01-10"},{"slug":"routing","title":"Routing with folders","body":"Folders are routes.","date":"2026-02-02"}]
{"error":"title required"}
```

A folder can have `page.tsx` **or** `route.ts`, never both — a URL is either a page or an
endpoint. Using an `api/` prefix is a convention, not a requirement.

## Reading the request

```ts title="app/api/search/route.ts"
import type { NextRequest } from "next/server";
import { searchProducts } from "@/lib/products";

export async function GET(request: NextRequest) {
  const q = request.nextUrl.searchParams.get("q")?.trim() ?? "";
  if (q.length < 2) return Response.json({ error: "q must be at least 2 characters" }, { status: 400 });

  const limit = Math.min(Number(request.nextUrl.searchParams.get("limit") ?? 20) || 20, 100);
  const results = await searchProducts(q, limit);
  return Response.json({ q, results });
}
```

- `request.nextUrl` (from `NextRequest`) is a parsed URL with `searchParams`.
- `request.headers`, `request.cookies` (on `NextRequest`), `await request.json()`,
  `await request.formData()`, `await request.text()` for the body.
- Dynamic segments work as for pages; the second argument carries `params` (a
  Promise), typed with the generated `RouteContext` helper:

```ts title="app/api/posts/[slug]/route.ts"
import { getPost } from "@/lib/posts";

export async function GET(_req: Request, ctx: RouteContext<"/api/posts/[slug]">) {
  const { slug } = await ctx.params;
  const post = await getPost(slug);
  if (!post) return Response.json({ error: "not found" }, { status: 404 });
  return Response.json(post, { headers: { "Cache-Control": "public, max-age=60" } });
}
```

## Non-JSON responses

A handler can return anything a `Response` can carry — which makes it handy for feeds,
CSV exports and files:

```ts title="app/feed.xml/route.ts"
import { getAllPosts } from "@/lib/blog";

const SITE = "https://example.com";

function escapeXml(s: string) {
  return s.replace(/[<>&'"]/g, (c) => ({ "<": "&lt;", ">": "&gt;", "&": "&amp;", "'": "&apos;", '"': "&quot;" })[c]!);
}

export async function GET() {
  const posts = await getAllPosts();
  const items = posts
    .map((p) => `<item><title>${escapeXml(p.title)}</title><link>${SITE}/posts/${p.slug}</link><pubDate>${new Date(p.date).toUTCString()}</pubDate></item>`)
    .join("");
  const xml = `<?xml version="1.0" encoding="UTF-8"?><rss version="2.0"><channel><title>Acme blog</title><link>${SITE}</link><description>Posts</description>${items}</channel></rss>`;
  return new Response(xml, { headers: { "Content-Type": "application/rss+xml; charset=utf-8" } });
}
```

The folder is literally named `feed.xml`, so the URL is `/feed.xml`.

## Caching of Route Handlers

Since Next.js 15, `GET` handlers are **not cached by default** — they run per request.
In the default model you can opt a `GET` handler into static generation with
`export const dynamic = "force-static"` (useful for the feed above). With Cache
Components enabled, `GET` handlers follow the same prerendering rules as pages. Other
methods are never cached.

## Webhooks: verify before you trust

A webhook is an unauthenticated POST from the internet claiming to be from a provider.
Providers sign the body; verify the signature against the **raw** body before parsing:

```ts title="app/api/webhooks/billing/route.ts"
import { createHmac, timingSafeEqual } from "node:crypto";

export async function POST(request: Request) {
  const secret = process.env.BILLING_WEBHOOK_SECRET;
  if (!secret) return new Response("not configured", { status: 500 });

  const raw = await request.text(); // exact bytes that were signed
  const given = request.headers.get("x-signature") ?? "";
  const expected = createHmac("sha256", secret).update(raw).digest("hex");

  const ok = given.length === expected.length && timingSafeEqual(Buffer.from(given), Buffer.from(expected));
  if (!ok) return new Response("invalid signature", { status: 401 });

  const event = JSON.parse(raw) as { id: string; type: string };
  // idempotency: record event.id and ignore duplicates — providers retry
  // ...handle event.type
  return new Response(null, { status: 204 });
}
```

The header name and signing scheme here are generic; each provider documents its own
(often including a timestamp to prevent replays), and most ship an SDK helper that you
should prefer.

## Route Handler or Server Action?

| Need | Use |
|---|---|
| Your own form or button changes data | Server Action |
| External client, mobile app, third party | Route Handler |
| Webhook receiver | Route Handler |
| Non-HTML/JSON output: RSS, CSV, images, files | Route Handler |
| Data for your own Server Components | Neither — call the data function directly |

## How It Actually Works

Each `route.ts` is compiled into a separate server entry point. When a request matches
its path, Next.js constructs a Web `Request` from the incoming Node.js request
(`NextRequest` adds `nextUrl` and `cookies` helpers), calls the exported function for
the method, and converts the returned `Response` back into a Node.js response —
streaming the body if it is a `ReadableStream`. There's no React rendering involved at
all, which is why handlers are cheap and why they can't use components or hooks. At
build time, Next.js inspects each handler: a `GET` with `force-static` (or, under Cache
Components, one that doesn't touch request data) is executed once and its response
stored like a static page.

## Common mistakes

- **Calling your own Route Handler from a Server Component.** Import the function
  instead.
- **Parsing a webhook as JSON before verifying.** Re-serialised JSON won't match the
  signature; verify the raw text.
- **Returning 200 for errors.** Use accurate statuses: 400 bad input, 401/403 auth,
  404, 409 conflicts, 422 validation.
- **Forgetting CORS** when a browser on another origin calls the endpoint — you must
  add `Access-Control-Allow-*` headers and handle `OPTIONS` yourself.
- **No authentication on endpoints that change data.** Check a session or API key in
  every mutating handler.

## Exercise

1. Build `GET /api/products?category=` returning filtered JSON with a 400 for an
   unknown category. Test with `curl -i` and check status codes and headers.
2. Add `/feed.xml` for your Level 1 blog and validate it with an RSS validator.
3. Write the billing webhook handler, then write a small Node script that signs a body
   with the same secret and POSTs it. Confirm a tampered body is rejected with 401.
