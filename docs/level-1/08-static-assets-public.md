---
description: "What Next.js serves from the public folder, how it differs from imported assets, the caching headers each gets, and where special files belong."
---

# 08 · Static Assets & the `public` Folder

Not everything on a site is a component. PDFs, verification files, a `robots.txt`
someone else maintains, a video, a legacy script — these just need to be served as
files. Next.js gives you two ways to ship a file, and choosing the wrong one costs
either caching or convenience.

## `public/`: served as-is from the site root

Anything in `public/` is available at the same path from the root of your site:

```text
public/
├── brochure.pdf            → https://example.com/brochure.pdf
├── images/
│   └── logo.svg            → https://example.com/images/logo.svg
└── .well-known/
    └── security.txt        → https://example.com/.well-known/security.txt
```

Reference these with absolute paths — no import needed:

```tsx title="app/downloads/page.tsx"
import Image from "next/image";

export default function Downloads() {
  return (
    <main>
      <Image src="/images/logo.svg" alt="Acme" width={120} height={32} />
      <p>
        <a href="/brochure.pdf" download>Download the brochure (PDF)</a>
      </p>
    </main>
  );
}
```

Rules worth knowing:

- File names are **not** changed or hashed. `/brochure.pdf` stays `/brochure.pdf`.
- Only files that exist **at build time** are served. Files added to `public/` while
  `next start` is running are not reliably picked up — for user uploads, use object
  storage (Level 2 · 08 discusses where uploads should go).
- A file in `public/` must not have the same path as a route in `app/`; that's a
  conflict.
- Don't name a folder `public/_next` — that prefix is reserved for Next.js's own assets.

## Importing assets: hashed and immutable

The alternative is to import a file from your code:

```tsx title="app/page.tsx"
import Image from "next/image";
import diagram from "./_assets/architecture.png";

export default function Home() {
  return <Image src={diagram} alt="Request flow from browser to server and database" />;
}
```

The bundler copies the file into `.next/static/media/` with a **content hash** in its
name. Change the image and the hash changes, which means the URL changes, which means
it can be cached forever safely.

## The caching difference

This is the practical reason to care:

| | `public/` file | Imported asset |
|---|---|---|
| URL | Fixed, human-readable | Hashed, changes with content |
| Default `Cache-Control` from `next start` (observed on 16.3.6) | `public, max-age=0` plus an `ETag` — revalidated every time | `public, max-age=31536000, immutable` |
| Good for | Files with meaningful, stable URLs (PDF links, `robots.txt`, verification files, `.well-known/`) | Images and files used by components |

If you replace `public/logo.png` with a new logo, visitors and CDNs may keep showing
the old one until their cached copy expires. With an import, the new file simply has a
new URL.

Should you need specific cache headers for `public/` files, set them in
`next.config.ts`:

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: "/fonts/:file*",
        headers: [{ key: "Cache-Control", value: "public, max-age=31536000, immutable" }],
      },
    ];
  },
};

export default nextConfig;
```

Only do this for files whose names include a version, or you'll have the stale-file
problem above in its worst form.

## Special files: prefer the conventions in `app/`

Some files you might instinctively drop into `public/` have smarter equivalents in
`app/`:

| Instead of `public/…` | Use `app/…` | Why |
|---|---|---|
| `favicon.ico` | `app/favicon.ico`, `app/icon.png` | Next.js adds the `<link>` tags for you |
| `robots.txt` | `app/robots.ts` | Can depend on environment (block crawlers on staging) |
| `sitemap.xml` | `app/sitemap.ts` | Generated from your real content |
| `og.png` | `app/opengraph-image.png` / `.tsx` | Wired into metadata automatically |

A static `public/robots.txt` still works; just don't have both a file and an
`app/robots.ts`.

## Worked example: a downloadable resource page

Say you publish a cheat sheet that is updated every term, and people bookmark and link
to it. You want a **stable URL** (`/cheatsheet.pdf`) and also want updates to show up.

```text
public/
└── cheatsheet.pdf
```

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  async headers() {
    return [
      {
        source: "/cheatsheet.pdf",
        headers: [
          { key: "Cache-Control", value: "public, max-age=3600, must-revalidate" },
          { key: "Content-Disposition", value: 'inline; filename="acme-cheatsheet.pdf"' },
        ],
      },
    ];
  },
};

export default nextConfig;
```

An hour of caching keeps load off the server while making sure an update reaches
readers the same day. The page that links to it can also show a version number so
readers know which edition they have.

## How It Actually Works

At build time Next.js records the list of files in `public/`. At request time the
server checks incoming paths against that list *after* checking the `_next/static`
prefix and before rendering app routes; a match is streamed from disk with an
`ETag`/`Last-Modified` so browsers can revalidate cheaply. Imported assets take a
different path entirely: the bundler treats the file as a module whose default export
is the final URL (and, for images, the dimensions and a tiny blur placeholder). Those
files live under `/_next/static/`, which the server marks as immutable because a
changed file always gets a new name. Platforms that deploy Next.js usually upload
`/_next/static/` and `public/` to a CDN, which is why both are served quickly even
though their caching semantics differ.

## Common mistakes

- **Writing uploads into `public/` at runtime.** It won't work reliably in production
  and is lost on every deploy. Use object storage or a database.
- **Replacing a `public/` image in place and wondering why users see the old one.** Use
  an import (hashed) or version the file name.
- **Importing large files (videos) into components.** They are copied into the build
  and can bloat it; host large media on a CDN or video platform.
- **Using relative paths like `src="images/logo.svg"`.** From `/blog/post` that
  resolves to `/blog/images/logo.svg`. Always use a leading slash.
- **Putting secrets in `public/`.** Everything there is downloadable by anyone.

## Exercise

1. Add a PDF to `public/` and link to it. Run `npm run build && npm start`, and inspect
   the response headers of the PDF in your browser's Network panel.
2. Import an image from `app/_assets/`, render it, and inspect its URL and response
   headers. Explain the difference in `Cache-Control`.
3. Replace a `public/robots.txt` with an `app/robots.ts` that disallows everything when
   `process.env.SITE_ENV === "staging"`. (You'll build a full version in Level 3 · 05.)
