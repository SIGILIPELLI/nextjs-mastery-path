---
description: "Capstone: design, build, test, containerize and review a production-grade Next.js knowledge base with auth, caching, streaming and observability."
---

# 10 · Capstone — A Production Next.js App

The capstone is where you stop following along and make the decisions yourself. You'll
build **Handbook**, a small team knowledge base, to a standard you'd be comfortable
putting in front of real users — then prove it with tests, a build report and the
production readiness review from lesson 09.

Everything you need has appeared in an earlier lesson; each requirement below links
back to where. Expect this to take several days of focused work.

## The product

Handbook lets a team write and find internal articles.

- **Public marketing pages**: `/`, `/pricing` — static, fast, SEO-optimised.
- **Sign in** with email + password (or an Auth.js provider).
- **Articles**: list, search, read, create, edit, delete. Markdown body. Each article
  belongs to a team; users only see their team's articles.
- **Article view** opens as a dialog from the list (intercepted route) and as a full page
  on direct visit.
- **Dashboard**: article counts, recent edits, and "most viewed" — streamed panels.
- **Admin**: team owners can change member roles.

## Requirements

### Architecture (Level 4 · 05)

```text
src/
├── app/
│   ├── (marketing)/page.tsx, pricing/page.tsx
│   ├── (auth)/login/page.tsx
│   ├── (app)/layout.tsx                  ← requires session; nav; skip link
│   ├── (app)/articles/page.tsx           ← list + search (URL state)
│   ├── (app)/articles/[id]/page.tsx      ← full page
│   ├── (app)/articles/@dialog/(.)[id]/…  ← intercepted dialog + default.tsx
│   ├── (app)/articles/new/page.tsx, [id]/edit/page.tsx
│   ├── (app)/dashboard/page.tsx          ← streamed panels
│   ├── (app)/admin/members/page.tsx
│   ├── api/health/route.ts
│   ├── sitemap.ts, robots.ts
│   └── global-error.tsx
├── features/{articles,members,dashboard}/{queries,actions,schema}.ts + components/
├── lib/{session,dal,env,log}.ts
├── db/{schema,index}.ts + drizzle/ migrations
├── instrumentation.ts
└── proxy.ts                               ← request IDs; optimistic auth redirect
```

### Functional and technical requirements

| # | Requirement | Lessons |
|---|---|---|
| R1 | Marketing pages are `○` in the route table; unique metadata; OG image | L1·07, L3·05 |
| R2 | Session in an `HttpOnly` cookie; every query/action in the DAL checks session **and** team membership | L2·07, L3·08 |
| R3 | Article create/edit via Server Actions with Zod validation and accessible errors | L2·03, L4·07 |
| R4 | After edits, the editor sees changes immediately (`updateTag` or `revalidatePath`) | L2·04 |
| R5 | Article search, filters and pagination live in the URL | L2·09 |
| R6 | Dashboard panels stream independently; each has an error boundary | L3·02, L3·10 |
| R7 | Article dialog via intercepting route; full page on refresh; focus returns on close | L3·03, L4·07 |
| R8 | Markdown rendered on the server and **sanitised** (user content!) | L1·10, L3·08 |
| R9 | Unknown article → real 404 status | L2·02 |
| R10 | Security headers; decided CSP policy documented | L3·08 |
| R11 | `instrumentation.ts` with `onRequestError`; JSON logs with request IDs | L4·04 |
| R12 | Tests: Vitest for schemas/DAL, Playwright for main flows + axe + 404 status | L3·07, L4·07 |
| R13 | `output: "standalone"`; Dockerfile; health check | L4·01, L4·09 |
| R14 | README with decisions: rendering per route, caching and invalidation, runtime, hosting | all |

### Stretch goals

- Enable `cacheComponents` and convert the article list and dashboard to explicit
  `"use cache"` + tags, with a `◐` route table. Document what changed.
- Add a second locale for the marketing pages with `hreflang` (L3·04).
- Multi-instance readiness: shared cache handler and a fixed action encryption key (L4·01).

## Suggested plan

1. **Day 1 — skeleton.** Scaffold, route groups, layouts, marketing pages, DB schema and
   migrations, seed script. Commit the first route table.
2. **Day 2 — auth and DAL.** Session helpers, login/logout, `requireUser`, team scoping.
   Write DAL tests *before* building UI on top.
3. **Day 3 — articles.** CRUD actions, forms, Markdown rendering with sanitisation,
   search and pagination.
4. **Day 4 — UX depth.** Intercepted dialog, streaming dashboard, loading/error/not-found
   everywhere, accessibility pass.
5. **Day 5 — production.** Headers, instrumentation, standalone build, Docker, health
   check, e2e + axe in CI, readiness review.

## A few reference snippets

Sanitising user Markdown on the server (one approach — `marked` plus
`isomorphic-dompurify`; check both libraries' current docs):

```ts title="src/features/articles/render.ts"
import "server-only";
import { marked } from "marked";
import DOMPurify from "isomorphic-dompurify";

export async function renderMarkdown(source: string): Promise<string> {
  const html = await marked.parse(source, { gfm: true });
  return DOMPurify.sanitize(html, { USE_PROFILES: { html: true } });
}
```

Team-scoped article query with a DTO:

```ts title="src/features/articles/queries.ts"
import "server-only";
import { cache } from "react";
import { and, eq } from "drizzle-orm";
import { requireUser } from "@/lib/dal";
import { db } from "@/db";
import { articles } from "@/db/schema";
import { renderMarkdown } from "./render";

export const getArticle = cache(async (id: number) => {
  const user = await requireUser();
  const row = db.select().from(articles).where(and(eq(articles.id, id), eq(articles.teamId, user.teamId))).get();
  if (!row) return null; // not found *or* not yours: same answer, no information leak
  return { id: row.id, title: row.title, html: await renderMarkdown(row.body), updatedAt: row.updatedAt.toISOString() };
});
```

A 404-status e2e test:

```ts title="e2e/not-found.spec.ts"
import { test, expect } from "@playwright/test";

test("unknown article returns 404", async ({ page }) => {
  // sign in first with a helper that sets the session cookie (see your auth tests)
  const response = await page.goto("/articles/999999");
  expect(response?.status()).toBe(404);
});
```

If that test returns 200, look for a `loading.tsx` above the article route (Level 2 · 02).

## Deliverables

1. The repository, with a README covering R14.
2. `next build` output saved as `docs/route-table.txt`, annotated: why each route has
   its symbol.
3. Test results from CI: Vitest, Playwright, axe.
4. The completed production readiness review (lesson 09) with **no Critical or High
   findings open**, and owners/dates for accepted Medium/Low risks.
5. A one-page architecture decision record for the hardest decision you made (for
   example: nonce CSP vs static marketing pages; SQLite vs Postgres; where to put the
   article cache).

## Assessment rubric

| Area | Meets the bar when… |
|---|---|
| Correctness | All R1–R13 demonstrably work in a production build |
| Security | A reviewer can't find an action or handler without auth + authorisation + validation |
| Rendering | Route table matches the README's intent; no accidental `ƒ` |
| Resilience | Failing one dashboard panel or the DB for one request doesn't blank the app; 404s are real |
| Accessibility | Keyboard and screen-reader walkthroughs succeed; axe clean |
| Operability | An error in production can be traced from the user's digest to a log line with a request ID |
| Communication | README and ADR let a new teammate understand *why*, not just *what* |

## How It Actually Works

The capstone is deliberately an integration exercise: each requirement exercises a
mechanism you've studied in isolation, and the interesting bugs appear where they meet.
A session read in the `(app)` layout makes those routes dynamic — fine — but if it
leaks into the root layout, the marketing pages lose `○`. A `loading.tsx` added for
nicer navigation changes the status code of your 404s. A CSP nonce forces dynamic
rendering onto pages you wanted static. Sanitised Markdown rendered in a cached function
is safe; the same function reading the session is a cross-user cache leak. Finding and
resolving these interactions — and writing down why — is the skill this course has been
building toward.

## Common mistakes

- **Starting with UI** before the DAL and its tests; authorisation gets bolted on later.
- **Rendering user Markdown without sanitising.**
- **Accidentally dynamic marketing pages** from a session read in a shared layout.
- **Skipping the readiness review** because "the tests pass".
- **Documenting what, not why.**

## Exercise

Build Handbook to the requirements above and complete every deliverable. Then ask
someone else to review it using only your README, route-table annotations and readiness
review — and fix whatever they couldn't understand or verify.
