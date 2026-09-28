---
description: "Run Next.js in a monorepo: workspaces, shared UI and config packages, transpilePackages, the Turbopack root, and cached builds with Turborepo."
---

# 03 · Monorepos

Once an organisation has a marketing site, a product app, an admin tool and a shared
design system, keeping them in separate repositories means publishing packages to
share a button. A **monorepo** puts them in one repository with workspace tooling so
apps import shared packages directly from source.

## A typical layout

```text
acme/
├── package.json            ← workspaces declared here
├── package-lock.json
├── apps/
│   ├── web/                ← Next.js marketing site
│   └── dashboard/          ← Next.js product app
└── packages/
    ├── ui/                 ← shared React components
    ├── db/                 ← Drizzle schema + queries (server-only)
    ├── config-eslint/
    └── config-typescript/
```

```json title="package.json (root)"
{
  "name": "acme",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "build": "npm run build --workspaces --if-present",
    "dev": "npm run dev --workspace=apps/dashboard"
  }
}
```

npm, pnpm, Yarn and Bun all support workspaces. pnpm is popular for monorepos because
its strict `node_modules` layout surfaces undeclared dependencies.

## A shared UI package

```json title="packages/ui/package.json"
{
  "name": "@acme/ui",
  "version": "0.0.0",
  "private": true,
  "exports": {
    "./button": "./src/button.tsx",
    "./card": "./src/card.tsx"
  },
  "peerDependencies": { "react": "^19" }
}
```

```tsx title="packages/ui/src/button.tsx"
export function Button({ children, ...props }: React.ButtonHTMLAttributes<HTMLButtonElement>) {
  return (
    <button {...props} style={{ padding: "0.5rem 1rem", borderRadius: 8 }}>
      {children}
    </button>
  );
}
```

Using it from an app:

```json title="apps/dashboard/package.json (excerpt)"
{ "dependencies": { "@acme/ui": "*", "next": "16.3.6", "react": "19.2.8", "react-dom": "19.2.8" } }
```

```ts title="apps/dashboard/next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@acme/ui"], // compile the package's TypeScript/JSX source
};

export default nextConfig;
```

```tsx title="apps/dashboard/app/page.tsx"
import { Button } from "@acme/ui/button";

export default function Home() {
  return <Button type="button">Create report</Button>;
}
```

The package exports **source** (`.tsx`), not compiled JavaScript, so there's no build
step for it; `transpilePackages` tells Next.js to compile it as part of the app. The
alternative — building each package to `dist/` — is better when packages are also
published to npm or consumed by non-Next tools.

### Server and client in shared packages

Directives work across package boundaries. A package component that uses state needs
`"use client"` at the top of its file, exactly as inside an app. Keep server-only
packages (like `@acme/db`) guarded with `import "server-only"` so no app accidentally
bundles database code for the browser. Separate entry points per component (the
`exports` map above) avoid pulling every component — and every `"use client"` boundary —
into a page that needs one button.

## Turbopack and the workspace root

Turbopack needs to know the monorepo root to resolve files outside the app directory.
It usually infers it from lock files; if it guesses wrong (for example with several lock
files on the machine), set it explicitly:

```ts title="apps/dashboard/next.config.ts"
import path from "node:path";
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@acme/ui"],
  turbopack: { root: path.join(__dirname, "..", "..") },
};

export default nextConfig;
```

For standalone output in a monorepo, `outputFileTracingRoot` plays the same role for
file tracing, so files from `packages/` are included.

## Task runners and caching

Building five apps and ten packages on every commit is slow. **Turborepo** and **Nx**
understand the dependency graph between workspaces and:

- run tasks in dependency order and in parallel;
- **cache** task outputs keyed by input file hashes, so an unchanged app isn't rebuilt;
- optionally share that cache across CI machines and developers.

```json title="turbo.json"
{
  "$schema": "https://turborepo.com/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "!.next/cache/**", "dist/**"] },
    "lint": {},
    "test": { "dependsOn": ["^build"] }
  }
}
```

`"^build"` means "build my dependencies first". Excluding `.next/cache` keeps the cached
artifact small. Check the Turborepo docs for the current schema URL and options.

Next.js's own build cache (`.next/cache`, including Turbopack's file-system cache) should
also be persisted between CI runs; the Next.js docs have a CI build caching guide per
provider.

## How It Actually Works

Workspaces are a package-manager feature: during install, each workspace package is
symlinked into `node_modules` (`node_modules/@acme/ui → packages/ui`), so imports
resolve like any dependency. Next.js, by default, doesn't compile code from
`node_modules` — it expects published packages to be plain JavaScript.
`transpilePackages` adds the listed packages to the set of modules run through the
TypeScript/JSX transform and the server/client directive handling. Turbopack resolves
the symlinks to real paths and must be allowed to read them, which is why it needs a
root that contains both the app and the packages. Task runners hash each package's
inputs (source files, dependencies' outputs, env vars you declare) to decide if a cached
output can be replayed instead of re-running the build.

## Common mistakes

- **Forgetting `transpilePackages`** for source-only packages — syntax errors on JSX or TS.
- **Several React copies** (each app and package with its own `react`) — hooks break.
  Make React a peer dependency of packages.
- **One giant barrel export** from `@acme/ui` with `"use client"` at the top — every page
  imports all client components.
- **Turbopack root inferred incorrectly** — "module not found" for files outside the app.
- **Not declaring env vars in the task runner's config**, so cached builds reuse outputs
  built with different environment values.

## Exercise

1. Create a workspace with `apps/web` (create-next-app) and `packages/ui` exporting a
   Button and a Card from source. Use both in the app with `transpilePackages`.
2. Add `packages/db` with a `server-only` guard and try importing it into a Client
   Component in the app. Read the error.
3. Add Turborepo, run `turbo build` twice, and note which tasks were cached the second
   time.
