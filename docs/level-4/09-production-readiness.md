---
description: "A production readiness review for Next.js apps covering build, security, data, performance, accessibility and operations, each with a verification."
---

# 09 · Production Readiness Review

"It works on my machine" and "it's ready for real users" are separated by a list of
unglamorous checks. This lesson turns the course into a **review you can run** on any
Next.js app before launch or on a schedule afterwards. Each item has a *how to verify*
so the review produces evidence, not opinions.

## How to run the review

- Do it on a **production build** (`next build && next start`, or a staging deploy),
  never `next dev`.
- Record each item as Pass / Fail / N/A with the evidence (command output, screenshot,
  link).
- Fix Fails or record an accepted risk with an owner and a date.

## 1. Build and configuration

| Check | Verify |
|---|---|
| `next build` passes with no type errors or warnings you haven't triaged | CI log |
| Next.js and React on a supported, patched version | `npm view next version` vs `package.json`; security advisories |
| Route table matches intent (static pages are `○`/`●`, not `ƒ`) | Save and review build output |
| Required env vars validated at startup | Start with one missing; it should fail loudly |
| No secrets in `NEXT_PUBLIC_*` | `grep -r NEXT_PUBLIC_ .env*` and review |
| `poweredByHeader: false`, source maps policy decided | `curl -I` |

## 2. Security

| Check | Verify |
|---|---|
| Every Server Action and Route Handler authenticates, authorises and validates | Inventory table (Level 3 · 08 exercise) |
| Authorisation lives in the data access layer, not only in Proxy/layouts | Code review of `queries.ts`/`actions.ts` |
| No raw DB rows passed to Client Components | Search for Client Components receiving model types |
| Security headers present; CSP decided (nonce vs hashes vs none) | `curl -I` on a page |
| Cookies `HttpOnly`, `Secure`, `SameSite` | Browser devtools → Application → Cookies |
| Redirect targets validated; remote image hosts restricted | Tests for `safeRedirect`; `images.remotePatterns` review |
| Dependencies audited | `npm audit` triaged |

## 3. Data and caching

| Check | Verify |
|---|---|
| Every mutation revalidates what it changes | For each action, note its `revalidatePath`/tag call; e2e test that the UI updates |
| No personalised data in shared caches | Review `"use cache"`/`force-cache` sites for cookie/user inputs |
| Multi-instance: shared cache handler, same action encryption key, same build | Deployment config |
| Migrations run as a release step, backups exist and restore has been tested | Runbook + last restore test date |

## 4. Errors and edge states

| Check | Verify |
|---|---|
| `error.tsx` at meaningful levels + `global-error.tsx` | Throw deliberately in staging |
| `not-found.tsx` and correct 404 status for missing resources | `curl -w "%{http_code}"` (watch streaming, Level 2 · 02) |
| Loading UI for every dynamic route users navigate to | Click through with network throttling |
| Forms handle validation errors, double submits and slow networks | Manual test + e2e |

## 5. Performance

| Check | Verify |
|---|---|
| Core Web Vitals within "good" at p75 on key pages | Field data (`useReportWebVitals`/CrUX) or Lighthouse on staging for new sites |
| LCP image prioritised; fonts via `next/font`; images sized | Lighthouse diagnostics |
| Client JS reasonable; no heavy libraries in Client Components without reason | Bundle analyser |
| Streaming works through the proxy/CDN | `curl -N`, time to first byte |

## 6. SEO and content (public sites)

| Check | Verify |
|---|---|
| Unique titles/descriptions; canonical URLs | View source on a sample of pages |
| `sitemap.xml`/`robots.txt` correct for production (and blocking staging) | Fetch both on each environment |
| Open Graph images render | Social preview debugger |
| Private areas `noindex` | Metadata review |

## 7. Accessibility

| Check | Verify |
|---|---|
| Keyboard-only walkthrough of main flows | Manual |
| Screen-reader walkthrough of signup/login/main task | Manual (VoiceOver/NVDA) |
| axe checks pass on main routes | Playwright + axe in CI |
| Titles unique (route announcements) | Automated test comparing titles |

## 8. Observability and operations

| Check | Verify |
|---|---|
| Server errors reported with digest and route context | Trigger an error, find it in logs |
| Request IDs correlate logs | Follow one request end to end |
| Alerts on error rate and latency, routed to someone | Alert config + on-call rota |
| Rollback procedure documented and tried | Last rollback drill date |
| Health check endpoint for the load balancer | `curl /api/health` |

A minimal health check that also verifies the database:

```ts title="app/api/health/route.ts"
import { sql } from "drizzle-orm";
import { db } from "@/db";

export async function GET() {
  try {
    db.run(sql`select 1`);
    return Response.json({ ok: true }, { headers: { "Cache-Control": "no-store" } });
  } catch {
    return Response.json({ ok: false }, { status: 503, headers: { "Cache-Control": "no-store" } });
  }
}
```

## Worked example: reviewing the Level 2 task manager

Running the review on the Level 2 project as built in that lesson produces findings
like these (your list may differ):

| Area | Finding | Severity | Fix |
|---|---|---|---|
| Security | Actions have no authentication; anyone can delete any task | Critical | Session + ownership checks in actions (Level 2 · 07) |
| Security | No security headers | Medium | `headers()` config (Level 3 · 08) |
| Data | SQLite file on local disk; no backup | High | Persistent volume + scheduled backup, or managed DB |
| Errors | No `error.tsx` under `/tasks` | Medium | Add boundary with retry |
| Observability | No `onRequestError`; logs are unstructured | Medium | `instrumentation.ts` (Level 4 · 04) |
| SEO | Task pages indexable | Low | `robots: { index: false }` |
| Tests | e2e covers create/complete/delete; no edit test | Low | Extend test |

The capstone asks you to take an app to "no Critical or High findings open".

## How It Actually Works

A readiness review works because each item maps to a failure mode seen in real
incidents: unauthenticated actions (data loss), unshared caches across instances
(inconsistent data), buffered streaming (slow pages), streamed 404s (SEO and monitoring
blind spots), missing digests (undiagnosable errors). Next.js's own production checklist
in the docs covers framework-level items; this review adds the application-level ones.
Automating the verifiable rows — build checks, header checks, axe, e2e, status-code
tests — in CI turns the review from an annual event into a continuous gate.

## Common mistakes

- **Reviewing `next dev`.**
- **Checklists without evidence** — "yes, we have auth" without looking at every action.
- **Treating the review as one-off.** Re-run after major changes and upgrades.
- **No owner for accepted risks.**

## Exercise

1. Run the full review on your Level 2 or Level 3 project and produce the findings table.
2. Automate at least five rows in CI (for example: build, header check with `curl`,
   axe, e2e, 404 status test).
3. Add the health check route and wire it to your process manager or container
   orchestrator's health probe.
