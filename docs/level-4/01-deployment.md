---
description: "Deploy Next.js vendor-neutrally: next start on Node, standalone output, a multi-stage Dockerfile, reverse proxies, multiple instances and platforms."
---

# 01 · Deployment: Managed & Self-Hosted

A Next.js app can run anywhere Node.js runs. Managed platforms make it easy; your own
server or container gives you control and predictable costs. Understanding the
self-hosted path also makes you a better user of managed platforms, because you know
what they're doing for you.

## Five deployment shapes

| Shape | What you deploy | Supports |
|---|---|---|
| **Node server** (`next build` + `next start`) | The project + `node_modules` | Everything |
| **Standalone output** | `.next/standalone` folder (minimal files) | Everything |
| **Docker container** | An image built from standalone output | Everything |
| **Static export** (`output: "export"`) | Plain HTML/CSS/JS files | No server features (lesson 08) |
| **Managed platform** | Your repo; the platform builds it | Everything, plus platform extras |

## `next start` on a server

The simplest production deployment:

```bash
npm ci
npm run build
NODE_ENV=production PORT=3000 npm start
```

Run it under a process manager (systemd, PM2, a container orchestrator) that restarts it
on crash and on boot. Put a reverse proxy (nginx, Caddy) in front for TLS, compression
and static caching if you like — but configure it not to **buffer** responses, or
streaming stops working (Level 3 · 02):

```nginx
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;   # let streamed chunks through immediately
}
```

## Standalone output

Deploying all of `node_modules` is wasteful. With standalone output, the build traces
which files the server actually imports and copies only those:

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = { output: "standalone" };

export default nextConfig;
```

On the Level 2 project (with Drizzle, SQLite, Zod and the dev tooling installed), we
observed:

```text
$ ls .next/standalone
node_modules  package.json  server.js
$ du -sh .next/standalone node_modules
 49M	.next/standalone
630M	node_modules
```

The standalone folder doesn't include `public/` or `.next/static/` (they're meant for
a CDN); copy them in if the Node server should serve them:

```bash
cp -R public .next/standalone/
cp -R .next/static .next/standalone/.next/
cd .next/standalone && PORT=3122 node server.js
```

We ran exactly that, and the server started (`▲ Next.js 16.3.6 … Local: http://localhost:3122`)
and served `/blog` with status 200.

## Docker

A multi-stage Dockerfile building standalone output (we could not run Docker on the
machine used to write this lesson, so no build output is shown; the structure follows
the official example's approach):

```dockerfile title="Dockerfile"
# 1. Install dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# 2. Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# 3. Run: only the standalone output
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production NEXT_TELEMETRY_DISABLED=1 PORT=3000 HOSTNAME=0.0.0.0
RUN addgroup -S nodejs && adduser -S nextjs -G nodejs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

Notes:

- Native modules (like `better-sqlite3`) must be compiled for the **runtime image's**
  platform — building on macOS and copying `node_modules` into a Linux image fails.
  Installing inside the image, as above, handles it; Alpine uses musl, so some native
  packages prefer a Debian-based `node:22-slim` image.
- `HOSTNAME=0.0.0.0` makes the server listen on all interfaces inside the container.
- Environment variables used by server code are read at **runtime**; `NEXT_PUBLIC_`
  variables are baked in at **build** time, so they must be present during `docker build`.

## Running several instances

Behind a load balancer with more than one instance, a few things must be shared or
consistent:

- **Cache**: by default each instance has its own in-memory/filesystem cache, so
  revalidating on one doesn't update others. Configure a shared cache handler (for
  example backed by Redis) via `cacheHandlers` / `cacheHandler` config.
- **Server Action encryption key**: set `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY` so all
  instances can decrypt each other's action payloads.
- **Build ID / deployment ID**: all instances must run the same build; set
  `deploymentId` to help clients detect version skew during rolling deploys.
- **Image optimisation cache**: per instance unless you put a CDN in front.

## Managed platforms

Vercel (the company behind Next.js), Netlify, Cloudflare, AWS Amplify and others build
from your repository and handle scaling, CDN distribution of static assets, preview
deployments per branch, and platform-specific features (edge caching of static shells,
image optimisation services, cron triggers). Next.js 16 introduced a **Build Adapters
API** (alpha at the time of writing) meant to let any platform integrate with the
build output in a supported way.

How to choose:

| If you value… | Consider |
|---|---|
| Zero ops, previews, global CDN out of the box | A managed platform |
| Predictable cost at steady high traffic, data residency, existing infra | Self-hosted Node / containers |
| No server at all | Static export |

Check any platform's documentation for which Next.js features it supports and how
(streaming, PPR, image optimisation, ISR) — support differs and changes.

## How It Actually Works

`next build` produces `.next/`: compiled server bundles per route, client chunks in
`.next/static/`, prerendered HTML/RSC files, and manifests. With `output: "standalone"`,
the build also runs **file tracing** (`@vercel/nft`): starting from each server entry
point, it follows every `require`/`import` and file read it can detect, and copies only
those files — including the specific parts of `node_modules` actually used — into
`.next/standalone`, along with a minimal `server.js` that boots the Next.js server
without the CLI. That's how 630 MB of dependencies became 49 MB. Anything loaded
dynamically in a way tracing can't see (a computed path) must be included with
`outputFileTracingIncludes`.

## Common mistakes

- **Reverse proxy buffering** that breaks streaming.
- **Forgetting to copy `public/` and `.next/static/`** into standalone deployments —
  pages load without CSS.
- **Expecting `NEXT_PUBLIC_` values to change at runtime.**
- **Multiple instances with separate caches** and no shared handler — users see
  different data depending on which instance they hit.
- **Running `next dev` in production.**

## Exercise

1. Build your Level 2 project with `output: "standalone"`, compare folder sizes, copy
   the static assets and run `node server.js`.
2. If you have Docker, build the image and run it with a volume for the SQLite file.
   Record the image size.
3. Put Caddy or nginx in front, and use `curl -N` against the Level 3 dashboard to
   confirm streaming still works through the proxy.
