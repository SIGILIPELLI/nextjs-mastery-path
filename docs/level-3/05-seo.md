---
description: "SEO for Next.js App Router sites: metadata, canonicals, generated sitemap.xml and robots.txt, dynamic Open Graph images and escaped JSON-LD."
---

# 05 · SEO in Depth

Next.js gives you good SEO defaults — server-rendered HTML, real links, fast pages —
but search visibility depends on details: correct titles and canonicals, a sitemap that
reflects reality, preview images people want to click, and structured data. This
lesson covers each with code verified on Next.js 16.3.6.

## Start with what crawlers see

Search engines can render JavaScript, but server-rendered HTML is indexed faster and
more reliably, and social preview bots (for chat apps and social networks) generally
don't run JavaScript at all. App Router pages are server-rendered by default, so the
main risks are content that only appears after client-side fetching, and links that
aren't real `<a href>` elements (buttons with `onClick` navigation). Check any page
with `curl`: if the content isn't in the HTML, a preview bot won't see it.

## Metadata: titles, descriptions, canonicals

```tsx title="app/layout.tsx (metadata)"
import type { Metadata } from "next";

export const metadata: Metadata = {
  metadataBase: new URL("https://example.com"),
  title: { default: "Acme", template: "%s | Acme" },
  description: "Guides and tools for building with Next.js.",
  openGraph: { siteName: "Acme", type: "website" },
  twitter: { card: "summary_large_image" },
};
```

Per page, set a unique `title`, a specific `description`, and a **canonical URL** when
the same content is reachable at several URLs (tracking parameters, sort orders):

```tsx
export const metadata = {
  title: "Pricing",
  description: "Plans for individuals and teams, with a free tier.",
  alternates: { canonical: "/pricing" },
};
```

For pages that must not be indexed (search results, account pages, staging):

```tsx
export const metadata = { robots: { index: false, follow: true } };
```

## Sitemap and robots, generated from real data

```ts title="app/sitemap.ts"
import type { MetadataRoute } from "next";
import { getPosts } from "@/lib/posts";

const base = "https://example.com";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await getPosts();
  return [
    { url: base, lastModified: new Date("2026-01-01") },
    ...posts.map((p) => ({ url: `${base}/blog/${p.slug}`, lastModified: new Date(p.date) })),
  ];
}
```

```ts title="app/robots.ts"
import type { MetadataRoute } from "next";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: { userAgent: "*", allow: "/", disallow: "/tasks" },
    sitemap: "https://example.com/sitemap.xml",
  };
}
```

Observed output from `next start`:

```text
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
<url>
<loc>https://example.com</loc>
<lastmod>2026-01-01T00:00:00.000Z</lastmod>
</url>
<url>
<loc>https://example.com/blog/hello-next</loc>
<lastmod>2026-01-10T00:00:00.000Z</lastmod>
</url>
...
</urlset>
```

```text
User-Agent: *
Allow: /
Disallow: /tasks

Sitemap: https://example.com/sitemap.xml
```

Both were listed as `○` (static) in the build. Use real modification dates — a
`lastModified: new Date()` on every URL tells crawlers everything changed on every
build, and they learn to ignore it. For very large sites, `generateSitemaps` splits
the sitemap into several files. Remember `robots.txt` `Disallow` stops crawling, not
indexing; use `noindex` metadata to keep a page out of results.

## Open Graph images, generated

A file named `opengraph-image.tsx` in a route folder generates that route's preview
image using `ImageResponse` from `next/og`:

```tsx title="app/blog/[slug]/opengraph-image.tsx"
import { ImageResponse } from "next/og";
import { getPost } from "@/lib/posts";

export const size = { width: 1200, height: 630 };
export const contentType = "image/png";

export default async function OgImage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params;
  const post = await getPost(slug);
  return new ImageResponse(
    (
      <div style={{ fontSize: 64, width: "100%", height: "100%", display: "flex", alignItems: "center", justifyContent: "center", background: "#0b1220", color: "white" }}>
        {post?.title ?? "Blog"}
      </div>
    ),
    size,
  );
}
```

Next.js adds the `og:image` tag automatically. Observed on a post page:

```text
og:image" content="https://example.com/blog/routing/opengraph-image?f00c0c02b4c3b6c9
```

and the image URL returned `200 image/png`. `ImageResponse` supports a subset of CSS
(flexbox layout, no grid), so keep designs simple.

## Structured data (JSON-LD)

Structured data describes what a page *is* (an article, a product, a course) in a
vocabulary search engines understand. Render it as a script tag in the page:

```tsx title="app/blog/[slug]/page.tsx (excerpt)"
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  headline: post.title,
  datePublished: post.date,
  author: { "@type": "Person", name: "Ana Diaz" },
};

return (
  <article>
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd).replace(/</g, "\\u003c") }}
    />
    {/* ... */}
  </article>
);
```

The `replace(/</g, "\\u003c")` prevents a title containing `</script>` from breaking
out of the tag — a real XSS vector if titles come from users. Validate the output with
a structured-data testing tool; only markup that matches visible content is eligible
for rich results.

## Technical checklist

- One `<h1>` per page that matches the topic; logical heading order.
- Descriptive link text ("Read the caching guide", not "click here").
- Real 404 status for missing content (watch the streaming caveat from Level 2 · 02).
- `redirects()` with `permanent: true` (308) when URLs change.
- Fast pages — Core Web Vitals are a ranking signal (next lesson).
- `lang` on `<html>` and `hreflang` for translations (lesson 04).

## How It Actually Works

`sitemap.ts`, `robots.ts` and `opengraph-image.tsx` are **metadata route files**. The
build turns each into a Route Handler at a fixed URL (`/sitemap.xml`, `/robots.txt`,
`/blog/[slug]/opengraph-image`) that calls your function and serialises the result to
XML, text or PNG. Because they are handlers, the same static/dynamic rules apply: no
request data → generated once at build. The hash query string on the `og:image` URL is
derived from the image's content so link-preview caches refetch when it changes.
`ImageResponse` converts your JSX to SVG with a layout engine (Satori) and then
rasterises it to PNG, which is why it supports only a subset of CSS.

## Common mistakes

- **Same title and description on every page.**
- **Canonical pointing to the wrong place** (for example every page canonicalised to the
  homepage) — search engines may drop those pages.
- **Blocking CSS/JS in `robots.txt`** — crawlers need them to render.
- **Sitemaps listing URLs that redirect or 404.**
- **Unescaped JSON-LD** built from user content.
- **Client-only content for key text**, invisible to preview bots.

## Exercise

1. Add `sitemap.ts` and `robots.ts` to your Level 1 blog, using real post dates. Make
   robots disallow everything when `process.env.SITE_ENV === "staging"`.
2. Add a generated Open Graph image per post, and check it with `curl -I` and in a
   social preview debugger.
3. Add `BlogPosting` JSON-LD and validate it. Then set a post title containing
   `</script>` and confirm your escaping holds.
