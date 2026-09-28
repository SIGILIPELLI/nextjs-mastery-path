---
description: "Structure a large Next.js codebase: feature folders, route groups, a server-only data access layer returning DTOs, and lint-enforced boundaries."
---

# 05 · Large-App Architecture

A ten-page app can live with any structure. At two hundred routes and twenty
developers, structure decides whether a change takes an hour or a week. Next.js is
deliberately unopinionated beyond `app/`'s special files, so the conventions are yours to
choose — this lesson proposes a set that works well with the App Router's server/client
split, and explains the reasoning so you can adapt it.

## Principle 1: the `app/` folder is for routing

Keep route folders thin: pages and layouts compose features; they don't contain
business logic.

```text
src/
├── app/                         ← routes only: pages, layouts, special files
│   ├── (marketing)/
│   ├── (app)/
│   │   ├── layout.tsx
│   │   ├── invoices/
│   │   │   ├── page.tsx         ← ~20 lines: fetch via feature, render feature components
│   │   │   └── [id]/page.tsx
│   │   └── settings/
│   └── api/webhooks/
├── features/                    ← one folder per business capability
│   ├── invoices/
│   │   ├── components/          ← InvoiceTable.tsx, InvoiceForm.tsx (client)
│   │   ├── actions.ts           ← "use server" mutations
│   │   ├── queries.ts           ← server-only reads (the DAL for this feature)
│   │   ├── schema.ts            ← Zod schemas shared by form and action
│   │   └── index.ts             ← public surface of the feature
│   └── billing/
├── components/ui/               ← generic design-system components
├── lib/                         ← cross-cutting: session, log, env, format
└── db/                          ← schema and connection
```

Colocating by feature means one pull request usually touches one folder, and deleting a
feature means deleting a folder.

## Principle 2: a data access layer that returns DTOs

All reads and writes go through server-only functions that (a) check authorisation and
(b) return plain objects shaped for the UI:

```ts title="src/features/invoices/queries.ts"
import "server-only";
import { cache } from "react";
import { requireUser } from "@/lib/dal";
import { db } from "@/db";

export type InvoiceRow = { id: string; number: string; customer: string; totalCents: number; status: "draft" | "sent" | "paid" };

export const listInvoices = cache(async (): Promise<InvoiceRow[]> => {
  const user = await requireUser();
  const rows = await db.query.invoices.findMany({
    where: (i, { eq }) => eq(i.orgId, user.orgId),
    with: { customer: true },
    orderBy: (i, { desc }) => desc(i.createdAt),
  });
  return rows.map((r) => ({
    id: r.id,
    number: r.number,
    customer: r.customer.name,
    totalCents: r.totalCents,
    status: r.status,
  }));
});
```

Pages never touch `db`; they call `listInvoices()`. Benefits: authorisation can't be
forgotten, private columns can't leak to Client Components, and swapping the database or
adding caching happens in one place.

## Principle 3: explicit client boundaries

- Client Components live in `components/` folders of features and are named for what
  they do (`InvoiceFilters.tsx`), each file starting with `"use client"`.
- They receive DTOs and Server Actions as props or import actions directly — never
  `queries.ts`.
- Providers (theme, query client, feature flags) sit in one `app/providers.tsx`
  Client Component rendered by the root layout, wrapping `children` so pages remain
  Server Components.

## Principle 4: enforce the rules with tooling

Conventions decay unless machines check them. Use ESLint's `no-restricted-imports` (or a
dedicated boundaries plugin) to forbid cross-feature deep imports:

```js title="eslint.config.mjs (excerpt)"
export default [
  // ...next's config
  {
    files: ["src/**/*.{ts,tsx}"],
    rules: {
      "no-restricted-imports": ["error", {
        patterns: [
          { group: ["@/features/*/*"], message: "Import from the feature's index (e.g. '@/features/invoices'), not its internals." },
          { group: ["@/db", "@/db/*"], message: "Only queries.ts/actions.ts may use the database." },
        ],
      }],
    },
  },
  {
    files: ["src/features/*/queries.ts", "src/features/*/actions.ts", "src/lib/**", "src/db/**"],
    rules: { "no-restricted-imports": "off" },
  },
];
```

Combine with `server-only` (runtime/build guard) and TypeScript's strict mode.

## Principle 5: shared layouts carry little data

Layouts persist across navigations and don't re-render per page, so they're a poor
place for page-specific data. Keep them to the frame: navigation, session-dependent
bits streamed in small Suspense boundaries, and providers. Heavy layout data fetching
also makes every child route wait.

## Scaling the team

- **Route groups per area** (`(app)/(billing)/...`) with a CODEOWNERS entry each.
- **Typed routes**: `typedRoutes: true` in `next.config.ts` makes `<Link href>` check
  paths against your route tree at compile time — renaming a route becomes a type error
  wherever it's linked.
- **Feature flags** evaluated on the server (in the DAL or a layout) so disabled code
  paths don't ship to the client.
- **Multi-zones** when separate teams need independent deployments under one domain:
  several Next.js apps each own a path prefix, stitched together by rewrites. It's a
  significant operational choice; prefer a monorepo with one app until deploy coupling
  genuinely hurts.

## How It Actually Works

None of this structure is visible to Next.js except `app/`: the router only scans that
folder, and everything else is ordinary modules resolved through the `@/` alias. That's
why you're free to organise the rest. The server/client split, however, *is* enforced by
the bundler: the module graph is walked from each page, and wherever a `"use client"`
file is reached, its whole import subtree goes to the browser. A clean architecture is
therefore one where the import graph matches your intent — server-only modules
reachable only from server code (guarded by `server-only`), client modules small and at
the leaves. Lint rules and folder conventions exist to keep that graph honest as the team
grows.

## Common mistakes

- **Business logic in `page.tsx`** — untestable and duplicated across routes.
- **Pages querying the database directly**, bypassing authorisation.
- **A `utils/` junk drawer** everyone imports from, creating hidden coupling.
- **Barrel files with `"use client"`** that drag whole folders into client bundles.
- **Premature multi-zones or micro-frontends.**

## Exercise

1. Restructure your Level 2 task manager into `features/tasks/` with `queries.ts`,
   `actions.ts`, `schema.ts` and `components/`, leaving pages under 25 lines each.
2. Add the `no-restricted-imports` rules and make a page import `@/db` to see the lint
   error.
3. Turn on `typedRoutes`, misspell a `<Link href>`, and run `next build` to see the
   type error.
