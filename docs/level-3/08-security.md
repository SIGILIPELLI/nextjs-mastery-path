---
description: "Secure a Next.js app: Server Actions as public endpoints, data leaks to Client Components, env vars, security headers and a nonce-based CSP."
---

# 08 · Security

The App Router blurs the line between frontend and backend — which is convenient, and
also means backend mistakes can hide in files that look like UI. This lesson covers the
risks specific to Next.js, in the order they most often bite.

## 1. Every Server Action is a public endpoint

A `"use server"` function becomes an HTTP endpoint that anyone can call with any
arguments — not just from the form you wrote. Hiding the button for non-admins does
nothing to stop a crafted request.

```ts title="app/admin/actions.ts"
"use server";

import { z } from "zod";
import { requireUser } from "@/lib/dal";
import { db } from "@/db";
import { users } from "@/db/schema";
import { eq } from "drizzle-orm";

const Input = z.object({ userId: z.string().uuid(), role: z.enum(["user", "admin"]) });

export async function setRole(raw: unknown) {
  const session = await requireUser();                  // 1. authenticated?
  if (session.role !== "admin") throw new Error("Forbidden"); // 2. authorised?
  const { userId, role } = Input.parse(raw);            // 3. valid input?
  if (userId === session.userId) throw new Error("Cannot change your own role"); // 4. business rules
  await db.update(users).set({ role }).where(eq(users.id, userId));
}
```

The checklist for every action: **authenticate, authorise, validate, then act.**
Next.js adds some protections — actions only accept `POST`, compare the `Origin`
header to the host to block cross-site calls (configurable with
`serverActions.allowedOrigins`), and action IDs are unguessable and change between
builds, with unused actions removed from the bundle — but none of those replace your
checks.

## 2. Don't leak server data to the client

Anything passed as a prop to a Client Component is serialised into the page. It's easy
to send a whole database row:

```tsx
// ❌ sends passwordHash, email, internal notes… to every visitor
<ProfileCard user={user} />

// ✅ send only what the component displays
<ProfileCard user={{ name: user.name, avatarUrl: user.avatarUrl }} />
```

Good habits:

- Return **DTOs** (plain objects with only safe fields) from your data access layer, so
  pages never hold raw rows.
- Mark modules with secrets `import "server-only"`.
- React's experimental **taint** APIs (`experimental_taintObjectReference`,
  `experimental_taintUniqueValue`, enabled with `experimental.taint` in Next.js) can
  make it an error to pass a specific object or value to the client — a useful extra
  net, still experimental.

## 3. Environment variables

- Only variables prefixed **`NEXT_PUBLIC_`** are exposed to the browser, and they are
  **inlined at build time** — the value in the client bundle is whatever it was during
  `next build`, not at runtime.
- Never put secrets in `NEXT_PUBLIC_` variables. If you see `NEXT_PUBLIC_API_SECRET`,
  it's already public.
- Load secrets from `.env.local` in development (git-ignored) and from your host's
  secret store in production. `.env` files committed to the repo should hold only
  non-sensitive defaults.
- Validate required variables at startup so a missing secret fails loudly:

```ts title="lib/env.ts"
import "server-only";
import { z } from "zod";

export const env = z
  .object({
    DATABASE_URL: z.string().min(1),
    SESSION_SECRET: z.string().min(32),
  })
  .parse(process.env);
```

## 4. Security headers

Set baseline headers for every route in `next.config.ts`:

```ts title="next.config.ts"
import type { NextConfig } from "next";

const securityHeaders = [
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "X-Frame-Options", value: "DENY" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
];

const nextConfig: NextConfig = {
  poweredByHeader: false, // drop "X-Powered-By: Next.js"
  async headers() {
    return [{ source: "/:path*", headers: securityHeaders }];
  },
};

export default nextConfig;
```

Only send HSTS with `preload` once you're sure every subdomain serves HTTPS.

## 5. Content Security Policy with a nonce

A CSP tells the browser which scripts may run, limiting the damage of an XSS bug.
Next.js injects inline scripts (for hydration and streaming), so a strict CSP needs a
**nonce**: a random value per request, allowed in the policy and attached to each
script. Generate it in Proxy:

```ts title="proxy.ts"
import { NextResponse, type NextRequest } from "next/server";

export function proxy(request: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");
  const isDev = process.env.NODE_ENV === "development";
  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic'${isDev ? " 'unsafe-eval'" : ""}`,
    `style-src 'self' 'unsafe-inline'`,
    `img-src 'self' blob: data:`,
    `font-src 'self'`,
    `object-src 'none'`,
    `base-uri 'self'`,
    `form-action 'self'`,
    `frame-ancestors 'none'`,
  ].join("; ");

  const requestHeaders = new Headers(request.headers);
  requestHeaders.set("x-nonce", nonce);
  requestHeaders.set("Content-Security-Policy", csp);

  const response = NextResponse.next({ request: { headers: requestHeaders } });
  response.headers.set("Content-Security-Policy", csp);
  return response;
}

export const config = {
  matcher: [{ source: "/((?!api|_next/static|_next/image|favicon.ico).*)", missing: [{ type: "header", key: "next-router-prefetch" }] }],
};
```

Next.js reads the nonce from the request's CSP header and applies it to the scripts it
generates. The important trade-off: a per-request nonce means pages **must be rendered
per request** — static pages can't contain a nonce that changes every time.

We tested exactly this proxy on Next.js 16.3.6. On the dynamic `/tasks` page, all 16
`<script>` tags in the HTML carried the nonce. On the statically prerendered `/posts`
page, the response had the same CSP header but **none** of its 15 script tags had a
nonce — the HTML was generated at build time, before any nonce existed — so a browser
enforcing that policy would block the page's scripts. Adding the proxy did not make
`/posts` dynamic; you have to do that yourself (for example with `await connection()`
in the root layout) for every page covered by a nonce policy. Many sites
therefore choose a CSP without nonces for static content (allowing `'self'` scripts plus
hashes), or accept dynamic rendering for the pages that need the strict policy. The
Next.js CSP guide for your version details the current options, including an
experimental hash-based approach.

## 6. The usual web risks, Next.js edition

- **XSS**: React escapes text by default. The danger zones are
  `dangerouslySetInnerHTML` (sanitise), `href={userInput}` (block `javascript:` URLs), and
  JSON-LD script tags (escape `<`, Level 3 · 05).
- **Open redirects**: `redirect(searchParams.next)` can send users to a phishing site.
  Only allow relative paths starting with a single `/`.
- **SSRF**: fetching a user-supplied URL on the server (link previews, image proxies)
  can reach internal services. Allow-list hosts. The same reason `next/image` requires
  `remotePatterns`.
- **Dependency vulnerabilities**: keep Next.js patched. CVE-2025-29927 (a way to bypass
  Middleware in affected versions, fixed in 2025) is the canonical example of why you
  upgrade promptly and never rely on one layer for auth.

## How It Actually Works

Server Actions are dispatched by an ID sent in a request header; the server looks it
up in a manifest built at compile time, so only functions actually referenced from
client code are callable, and IDs are generated from a build-specific secret. Closure
variables captured by inline actions are encrypted with a key generated per build
(you can pin it with the `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` environment variable
when running several instances). Headers from `next.config.ts` are applied by the
server's routing layer after matching `source` patterns. For CSP, Next.js parses the
`Content-Security-Policy` request header during rendering, extracts the nonce, and adds
`nonce="…"` to every `<script>` it emits, including streamed chunks; `'strict-dynamic'`
then lets those trusted scripts load the framework's chunk files.

## Common mistakes

- **Authorisation only in the UI or Proxy.**
- **Passing full records to Client Components.**
- **Secrets in `NEXT_PUBLIC_` variables** or committed `.env` files.
- **A nonce CSP on pages you expect to be static.**
- **`redirect()` to unvalidated user input.**
- **Running an unpatched Next.js version** on a public site.

## Exercise

1. Audit your Level 2 project: list every Server Action and Route Handler and write,
   for each, who may call it and where that's enforced. Fix the gaps.
2. Add the security headers and verify them with `curl -I`. Then add the nonce CSP
   proxy; view the source of a static page and a dynamic page and check which scripts
   carry the nonce. Make the covered pages dynamic and confirm the browser console
   shows no CSP violations.
3. Write a `safeRedirect(target)` helper that accepts only same-site relative paths,
   with unit tests for `/ok`, `//evil.com`, `https://evil.com` and `/\evil.com`.
