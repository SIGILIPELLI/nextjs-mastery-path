---
description: "Upgrade Next.js safely: read version guides, run @next/codemod transforms, handle 15-to-16 breaking changes, and migrate from the Pages Router."
---

# 06 · Upgrading & Codemods

Next.js ships a major version roughly once a year, and each has brought breaking
changes: async `params` in 15, Proxy, Turbopack-by-default and removed APIs in 16.
Teams that treat upgrades as routine maintenance spend a few days; teams that skip two
majors face a migration project. This lesson gives you a repeatable process.

## The upgrade process

1. **Read the version's upgrade guide** end to end — it ships in your installed package
   at `node_modules/next/dist/docs/01-app/02-guides/upgrading/`. Make a checklist of
   items that apply to you.
2. **Start from green**: tests passing, `next build` clean, a branch with nothing else in it.
3. **Upgrade one major at a time** (14 → 15 → 16), even if you're behind.
4. **Run the codemods**, then review the diff like any other change.
5. **Fix what codemods can't**: behavioural changes (caching defaults), removed options.
6. **Build, test, and compare the route table** with the pre-upgrade one — rendering-mode
   changes show up there.
7. **Deploy to a staging environment** and watch error rates and Web Vitals after release.

## Codemods

`@next/codemod` rewrites source code with AST transforms:

```bash
# upgrade packages and run the main migrations for the target version
npx @next/codemod@canary upgrade latest

# individual transforms
npx @next/codemod@canary middleware-to-proxy .
npx @next/codemod@canary next-async-request-api .
npx @next/codemod@canary next-lint-to-eslint-cli .
```

Pass `--dry` to preview, and commit before running (the tool checks for a clean git tree
unless you use `--force`). We ran the Proxy rename on a small project containing this
`middleware.ts`:

```ts title="middleware.ts (before)"
import { NextResponse, type NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  if (!request.cookies.has("session")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }
}

export const config = { matcher: ["/account/:path*"] };
```

```bash
npx @next/codemod@16.3.6 middleware-to-proxy .
```

Afterwards `middleware.ts` was gone and `proxy.ts` existed with the same body and the
function renamed to `export function proxy(...)`. (The tool's own summary line reported
"1 skipped / 0 ok", which is misleading — always check the actual files and `git diff`
rather than trusting a summary.)

## What changed in 15 → 16 (summary)

From the version 16 upgrade guide (check it for the complete list):

| Change | Action |
|---|---|
| Node.js 20.9+ and TypeScript 5.1+ required | Upgrade runtime/CI images |
| Turbopack default for `dev` and `build` | Move custom `webpack` config to `turbopack` or opt out with `--webpack` |
| Synchronous `params`, `searchParams`, `cookies()`, `headers()` removed | `next-async-request-api` codemod, then fix leftovers |
| `middleware` → `proxy` (deprecated name) | `middleware-to-proxy` codemod |
| `revalidateTag(tag)` single-argument form deprecated | Add a profile (`"max"`) or use `updateTag` in actions |
| `experimental.ppr`, `dynamicIO`, `useCache` → `cacheComponents` | Opt in deliberately; see the Cache Components migration guide |
| Parallel route slots need `default.js` | Add them; the build fails otherwise |
| `next lint` removed; `next build` no longer lints | Run ESLint/Biome directly in CI |
| `serverRuntimeConfig` / `publicRuntimeConfig` removed | Environment variables |
| `next/image` defaults changed (qualities, cache TTL, local IP restriction…) | Review image config |
| AMP support removed | Remove AMP pages |

## Behavioural changes need tests, not codemods

Codemods fix syntax. They can't tell you that a page which used to be cached is now
dynamic, or that `fetch` no longer caches by default (the big 14 → 15 change). Protect
yourself with:

- **Route-table diffs**: save `next build` output before and after; review every
  symbol change.
- **E2E tests** on critical paths (Level 3 · 07).
- **Performance budgets**: compare Web Vitals and server timings on staging.

## Migrating from the Pages Router

The App Router and Pages Router can coexist, so migrate **route by route**:

1. Keep `pages/` working. Add `app/` with a root layout that reproduces `_app.tsx` and
   `_document.tsx` (global CSS, fonts, providers as a Client Component).
2. Move one simple, low-traffic route. A given path must exist in only one router.
3. Translate data fetching:

| Pages Router | App Router |
|---|---|
| `getStaticProps` | `async` Server Component (static by default) |
| `getStaticProps` + `revalidate` | `export const revalidate` or `"use cache"` + `cacheLife` |
| `getServerSideProps` | Server Component reading `cookies()`/`headers()`/`searchParams`, or `connection()` |
| `getStaticPaths` | `generateStaticParams` (+ `dynamicParams`) |
| `pages/api/*` | `app/**/route.ts` or Server Actions |
| `next/head` | `metadata` / `generateMetadata` |
| `next/router` | `next/navigation` (`useRouter`, `usePathname`, `useSearchParams`) |

4. Repeat, moving shared components to Server Components where they don't need
   interactivity.
5. Remove `pages/` when empty.

Navigating between a Pages Router route and an App Router route is a full page load,
so migrate related routes (a section) together for the best experience.

## How It Actually Works

A codemod parses each file into an abstract syntax tree (jscodeshift over Babel/TS
parsers), finds patterns — an exported function named `middleware`, a property access on
`params` without `await` — and rewrites the tree, then prints it back preserving
formatting where it can. Because it works on syntax, it can't follow data flow across
files or know runtime behaviour; transforms that can't be applied safely are left with a
comment or skipped, which is why you review the diff. The `upgrade` command combines a
package-version bump with the transforms listed for the target version.

## Common mistakes

- **Jumping several majors at once.**
- **Running codemods on a dirty tree** and losing the ability to review their diff.
- **Trusting summaries** instead of `git diff`.
- **Not diffing the route table**, missing silent rendering changes.
- **Migrating the Pages Router all at once** instead of route by route.

## Exercise

1. Take a Next.js 14 or 15 example project (the Next.js repo's examples are tagged by
   version), upgrade it to 16 using the process above, and write down every manual fix.
2. Save route tables before and after and explain each difference.
3. Migrate one `getServerSideProps` page to an App Router Server Component, keeping the
   URL and behaviour identical, verified by an e2e test written *before* the migration.
