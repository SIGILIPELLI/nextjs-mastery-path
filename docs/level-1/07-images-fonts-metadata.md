---
description: "Use next/image for responsive, lazy-loaded images, next/font for self-hosted fonts without layout shift, and the Metadata API for titles and social tags."
---

# 07 · Images, Fonts & Metadata

Three things that every site needs and that are surprisingly easy to get wrong by hand:
images that don't wreck page speed, fonts that don't make text jump, and `<head>` tags
that are right on every page. Next.js has a built-in answer for each.

## Images with `next/image`

```tsx title="app/page.tsx"
import Image from "next/image";
import hero from "./hero.jpg"; // static import: width, height and blur data are known at build time

export default function Home() {
  return (
    <main>
      <Image
        src={hero}
        alt="A developer's desk with a laptop showing a code editor"
        placeholder="blur"
        fetchPriority="high"      // the hero is the Largest Contentful Paint element
        sizes="100vw"
        style={{ width: "100%", height: "auto" }}
      />
      <Image src="/team/ana.png" alt="Ana, course author" width={160} height={160} />
    </main>
  );
}
```

What `<Image>` does for you:

- **Prevents layout shift** — it always reserves space, either from the static import's
  dimensions or from the `width`/`height` you give it.
- **Serves modern formats at the right size** — requests go through the image optimiser
  (`/_next/image?url=…&w=…&q=…`), which resizes and re-encodes on demand and caches the
  result.
- **Emits a responsive `srcset`** — combined with `sizes`, the browser downloads only
  the resolution it needs.
- **Lazy-loads by default** — images below the fold load as they approach the viewport.

For your above-the-fold hero, tell the browser it matters: `fetchPriority="high"` or
`loading="eager"`. (In Next.js 16 the old `priority` prop is deprecated in favour of
`preload`, and the docs recommend `loading="eager"` or `fetchPriority="high"` in most
cases.)

### `fill` for unknown dimensions

When the image should cover a container whose size comes from CSS:

```tsx
<div style={{ position: "relative", aspectRatio: "16 / 9" }}>
  <Image src={post.cover} alt="" fill sizes="(max-width: 768px) 100vw, 50vw" style={{ objectFit: "cover" }} />
</div>
```

The parent must be positioned, and `sizes` is essential — without it the browser
assumes the image could be full viewport width and downloads a larger file.

### Remote images

Images from other domains must be allow-listed so your optimiser can't be abused as an
open proxy:

```ts title="next.config.ts"
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  images: {
    remotePatterns: [new URL("https://images.example.com/uploads/**")],
  },
};

export default nextConfig;
```

Version 16 also changed some image defaults (for example, the default allowed
`qualities` became `[75]` and the default `minimumCacheTTL` became four hours). If a
`quality={90}` prop seems ignored, that's why: add the value to `images.qualities`.

## Fonts with `next/font`

```tsx title="app/layout.tsx"
import { Inter, JetBrains_Mono } from "next/font/google";
import "./globals.css";

const inter = Inter({ subsets: ["latin"], display: "swap", variable: "--font-sans" });
const mono = JetBrains_Mono({ subsets: ["latin"], variable: "--font-mono" });

export default function RootLayout({ children }: LayoutProps<"/">) {
  return (
    <html lang="en" className={`${inter.variable} ${mono.variable}`}>
      <body>{children}</body>
    </html>
  );
}
```

```css title="app/globals.css"
body { font-family: var(--font-sans), system-ui, sans-serif; }
code, pre { font-family: var(--font-mono), ui-monospace, monospace; }
```

`next/font/google` downloads the font files **at build time** and serves them from your
own domain — the browser never contacts Google, which is better for privacy and removes
a DNS lookup and connection. When we built a test app with `Inter`, the rendered HTML
referenced a file under `/_next/static/media/…woff2` on the app's own origin.

For fonts you own, use `next/font/local`:

```ts
import localFont from "next/font/local";
const brand = localFont({ src: "./fonts/Brand-Variable.woff2", variable: "--font-brand" });
```

## Metadata

Export a `metadata` object (static) or a `generateMetadata` function (dynamic) from a
layout or page:

```tsx title="app/layout.tsx (metadata)"
import type { Metadata } from "next";

export const metadata: Metadata = {
  metadataBase: new URL("https://example.com"),
  title: { default: "Acme", template: "%s | Acme" },
  description: "Guides and tools for building with Next.js.",
  openGraph: { type: "website", siteName: "Acme" },
};
```

```tsx title="app/about/page.tsx"
export const metadata = { title: "About" }; // renders <title>About | Acme</title>
```

For pages whose metadata depends on data:

```tsx title="app/blog/[slug]/page.tsx (excerpt)"
import type { Metadata } from "next";
import { getPost } from "@/lib/posts";

export async function generateMetadata({ params }: PageProps<"/blog/[slug]">): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  return {
    title: post?.title ?? "Not found",
    description: post?.excerpt,
    openGraph: { type: "article", publishedTime: post?.date },
  };
}
```

Observed on a build with the root layout above and a post titled "Routing with
folders":

```text
<title>Routing with folders | Acme</title>
```

Metadata **merges** down the tree: a page's `title` is combined with the layout's
`template`, and fields the page doesn't set are inherited. `metadataBase` lets you
write relative URLs for images and canonical links.

### File-based metadata

Some metadata is easier as a file: `app/favicon.ico`, `app/icon.png`,
`app/apple-icon.png`, `app/opengraph-image.png` (or `.tsx` to generate one),
`app/robots.ts` and `app/sitemap.ts`. Level 3's SEO lesson covers the generated ones.

## How It Actually Works

**Images.** `<Image>` renders a plain `<img>` with `width`, `height`, `loading`,
`decoding="async"`, `srcset` and `sizes`. Each `srcset` URL points at the
`/_next/image` endpoint with a width from a fixed list of device sizes. When requested,
the server fetches the original (from disk or an allowed remote), resizes and encodes
it, stores the result in a cache directory, and returns it with cache headers. The
browser picks one `srcset` candidate based on `sizes` and the device pixel ratio —
that choice is the reason `sizes` matters so much.

**Fonts.** At build time `next/font` fetches the font files, subsets them, and writes
them to static assets. It generates an `@font-face` rule with a unique family name,
and a **fallback font with adjusted metrics** (`size-adjust`, `ascent-override`…) so the
system font shown while loading occupies nearly the same space as the web font —
that's what removes the layout shift when the font swaps in.

**Metadata.** Next.js collects `metadata` exports and `generateMetadata` results from
every segment of the matched route, merges them from root to leaf, and renders the
resulting tags into `<head>`. Because this happens on the server, crawlers and link
unfurlers see correct tags without running JavaScript.

## Common mistakes

- **Empty or useless `alt` text on meaningful images.** `alt=""` is correct only for
  decorative images. Describe what the image conveys.
- **Omitting `sizes` with `fill` or responsive layouts** — the browser downloads
  oversized images.
- **Lazy-loading the hero image.** It delays Largest Contentful Paint. Mark it eager or
  high priority.
- **Importing the same Google font in many files.** Define it once (for example in
  `app/fonts.ts`) and import the object; each call creates a separate font instance.
- **Using `<head>` or `next/head` in the App Router.** `next/head` is Pages Router;
  use `metadata`.
- **Forgetting `metadataBase`,** then wondering why Open Graph image URLs are relative.

## Exercise

1. Add a hero image with a static import and `placeholder="blur"`. Build, start, and
   use the Network panel to see which `/_next/image` width the browser requested at two
   different window sizes.
2. Switch your site to a Google font through `next/font`, applied via a CSS variable.
   Confirm in the Network panel that no request goes to a Google domain.
3. Give every page a unique title using a layout `template`, and add
   `generateMetadata` to one dynamic page. View the page source to confirm.
