---
description: "Level 4 of Next.js: managed or self-hosted deployment, runtimes, monorepos, observability, architecture, upgrades, accessibility and production review."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: run Next.js applications in production responsibly — on infrastructure you
chose deliberately, observable when something breaks, structured so a team can keep
changing them, accessible to everyone, and upgradeable when the next major version
arrives.

The first three levels were about building. This level is about everything around the
code: where it runs, how you'd know it's failing, how a codebase with thirty
contributors stays coherent, and what to check before calling something
production-ready.

Three ideas run through this level:

1. **Next.js is not tied to one host.** `next start` on a Node server, a Docker
   container, a static export on any file host, or a managed platform — each is a valid
   choice with different trade-offs. The course stays vendor-neutral and says which
   features depend on the platform.
2. **Production problems are found through telemetry, not reproduction.** Logs with
   request IDs, traces through rendering and data access, and error reports with
   digests let you answer "what happened?" after the fact.
3. **Upgrades are a routine, not an event.** Framework majors arrive roughly yearly;
   teams that upgrade with codemods, tests and small steps spend days, not months.

## Modules

1. [Deployment: Managed & Self-Hosted](01-deployment.md) — `next start`, standalone output, Docker, reverse proxies and managed platforms
2. [Node.js vs Edge Runtime](02-edge-vs-node-runtime.md) — what each runtime is, why Edge is deprecated for routes in 16, and where "edge" still matters
3. [Monorepos](03-monorepos.md) — workspaces, shared packages, `transpilePackages` and build caching
4. [Observability](04-observability.md) — `instrumentation.ts`, OpenTelemetry, `onRequestError`, structured logs
5. [Large-App Architecture](05-large-app-architecture.md) — feature folders, a data access layer, boundaries and conventions
6. [Upgrading & Codemods](06-upgrading-codemods.md) — reading upgrade guides, running codemods, staged migrations from Pages Router
7. [Accessibility](07-accessibility.md) — route announcements, focus, forms, semantics, and automated checks
8. [Static Export & CDN Caching](08-static-export-cdn.md) — `output: "export"`, its limits, and cache headers for CDNs
9. [Production Readiness Review](09-production-readiness.md) — a checklist-driven review of a real app, with how to verify each item
10. [Capstone — A Production Next.js App](10-capstone.md) — design, build, test, containerise and review a complete application

## Before you start

- Complete Levels 1–3, or be comfortable with Server Actions, caching, streaming and
  testing in the App Router.
- Some lessons involve Docker or a cloud account. Those parts are optional: every
  lesson explains what to do and what to expect, and nothing requires paid services.
  Where we could not run something while writing (for example, a Docker build on the
  machine used), the lesson says so rather than showing invented output.
- Related courses: [Docker Mastery Path](https://sigilipelli.github.io/docker-mastery-path/)
  for containers, [Kubernetes Mastery Path](https://sigilipelli.github.io/kubernetes-mastery-path/)
  for orchestration, and [Server Ops Mastery Path](https://sigilipelli.github.io/server-ops-mastery-path/)
  for running Linux servers and reverse proxies.
