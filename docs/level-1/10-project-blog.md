---
description: "Level 1 project: a statically generated Markdown blog in Next.js with an index, per-post routes and metadata, verified with a real next build."
---

# 10 · Project — A Markdown Blog

Time to combine the level into one working site: a blog whose posts are Markdown files
in the repository, rendered to static HTML at build time. It uses file-based routing,
a layout, `<Link>`, Server Components reading the file system, metadata, and — as a
preview of Level 2 — a dynamic `[slug]` route with `generateStaticParams`.

The code below was built with `next build` on Next.js 16.3.6 and exercised with
`next start`; the outputs shown are from that run.

## What you'll build

- `/posts` — a list of posts, newest first, with title, date and summary.
- `/posts/<slug>` — each post rendered from Markdown, with its own `<title>` and
  description.
- Unknown slugs return a real **404**.
- Everything is **static**: no server work per request.

## 1. Content

Posts live outside `app/`, in `content/posts/`, one Markdown file each, with a small
front-matter block:

```markdown title="content/posts/hello-world.md"
---
title: Hello, world
date: 2026-03-01
summary: Why this blog exists and what it will cover.
---

This is the **first post**. It is written in Markdown and rendered at build time.

## What's next

- Routing
- Data fetching
```

```markdown title="content/posts/layouts-explained.md"
---
title: Layouts, explained
date: 2026-03-08
summary: How nested layouts persist across navigation.
---

Layouts wrap pages and **stay mounted** when you navigate between siblings.
```

## 2. Dependencies

We need a Markdown renderer. `marked` is small and has no runtime dependencies;
`server-only` guards our file-system code:

```bash
npm install marked server-only
```

Because both are only imported by Server Components, neither adds a byte to the
browser bundle.

## 3. The data layer

```ts title="lib/blog.ts"
import "server-only";
import { readdir, readFile } from "node:fs/promises";
import path from "node:path";
import { cache } from "react";
import { marked } from "marked";

const POSTS_DIR = path.join(process.cwd(), "content", "posts");

export type PostMeta = { slug: string; title: string; date: string; summary: string };
export type Post = PostMeta & { html: string };

function parseFrontMatter(source: string): { data: Record<string, string>; body: string } {
  const match = /^---\r?\n([\s\S]*?)\r?\n---\r?\n?/.exec(source);
  if (!match) return { data: {}, body: source };
  const data: Record<string, string> = {};
  for (const line of match[1].split(/\r?\n/)) {
    const i = line.indexOf(":");
    if (i > 0) data[line.slice(0, i).trim()] = line.slice(i + 1).trim();
  }
  return { data, body: source.slice(match[0].length) };
}

function toMeta(slug: string, data: Record<string, string>): PostMeta {
  if (!data.title || !data.date) throw new Error(`Post "${slug}" needs title and date`);
  return { slug, title: data.title, date: data.date, summary: data.summary ?? "" };
}

export const getAllPosts = cache(async (): Promise<PostMeta[]> => {
  const files = (await readdir(POSTS_DIR)).filter((f) => f.endsWith(".md"));
  const posts = await Promise.all(
    files.map(async (file) => {
      const slug = file.replace(/\.md$/, "");
      const { data } = parseFrontMatter(await readFile(path.join(POSTS_DIR, file), "utf8"));
      return toMeta(slug, data);
    }),
  );
  return posts.sort((a, b) => b.date.localeCompare(a.date));
});

export const getPost = cache(async (slug: string): Promise<Post | null> => {
  if (!/^[a-z0-9-]+$/.test(slug)) return null; // never let a URL segment walk the file system
  try {
    const source = await readFile(path.join(POSTS_DIR, `${slug}.md`), "utf8");
    const { data, body } = parseFrontMatter(source);
    return { ...toMeta(slug, data), html: await marked.parse(body) };
  } catch {
    return null;
  }
});
```

Design notes:

- The front-matter parser is deliberately tiny (key: value lines). For nested YAML
  you'd use a library such as `gray-matter`; the point here is that there's no magic.
- A post missing `title` or `date` **throws**, which fails the build. That's what you
  want: broken content should never deploy silently.
- The slug regex matters. Without it, a request for `/posts/..%2F..%2Fetc%2Fpasswd`
  would be joined into a file path. Validating input from the URL is a habit worth
  forming now.
- `cache()` lets the index, the page and `generateMetadata` share results within one
  render.

## 4. The index page

```tsx title="app/posts/page.tsx"
import Link from "next/link";
import { getAllPosts } from "@/lib/blog";

export const metadata = { title: "Posts" };

export default async function PostsIndex() {
  const posts = await getAllPosts();
  return (
    <main>
      <h1>Posts</h1>
      <ul>
        {posts.map((p) => (
          <li key={p.slug}>
            <Link href={`/posts/${p.slug}`}>{p.title}</Link>{" "}
            <time dateTime={p.date}>{p.date}</time>
            <p>{p.summary}</p>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

## 5. The post page

```tsx title="app/posts/[slug]/page.tsx"
import type { Metadata } from "next";
import Link from "next/link";
import { notFound } from "next/navigation";
import { getAllPosts, getPost } from "@/lib/blog";

export const dynamicParams = false; // only slugs from generateStaticParams exist

export async function generateStaticParams() {
  return (await getAllPosts()).map((p) => ({ slug: p.slug }));
}

export async function generateMetadata({ params }: PageProps<"/posts/[slug]">): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) return {};
  return { title: post.title, description: post.summary, openGraph: { type: "article", publishedTime: post.date } };
}

export default async function PostPage({ params }: PageProps<"/posts/[slug]">) {
  const { slug } = await params;
  const post = await getPost(slug);
  if (!post) notFound();
  return (
    <article>
      <p><Link href="/posts">← All posts</Link></p>
      <h1>{post.title}</h1>
      <time dateTime={post.date}>{post.date}</time>
      <div dangerouslySetInnerHTML={{ __html: post.html }} />
    </article>
  );
}
```

The folder name `[slug]` makes this a **dynamic segment**: one file handles every post.
`generateStaticParams` lists the slugs to prerender, and `dynamicParams = false` says
"any other slug is a 404" rather than "try to render it on demand". Level 2 lesson 01
covers these in depth.

!!! warning "`dangerouslySetInnerHTML` and trust"
    Rendering HTML from Markdown is safe here because *you* write the Markdown and it's
    in your repository. If posts ever come from users, sanitise the HTML (for example
    with DOMPurify on the server) — `marked` does not sanitise by default.

## 6. Build and verify

```bash
npm run build
```

The relevant part of the route table observed while building this project:

```text
├ ○ /posts
├   /posts/[slug]
│ ├ ● /posts/layouts-explained
│ └ ● /posts/hello-world
```

Both posts were prerendered (`●`) and the index is static (`○`). Then:

```bash
npm start
curl -s localhost:3000/posts/hello-world | grep -o "<title>[^<]*</title>"
curl -s -o /dev/null -w "%{http_code}\n" localhost:3000/posts/nope
```

Observed (the root layout used a `"%s | Acme"` title template):

```text
<title>Hello, world | Acme</title>
404
```

## How It Actually Works

During `next build`, Next.js first calls `generateStaticParams` for `app/posts/[slug]`.
For each returned object it renders the page and `generateMetadata` with those params,
writing an HTML file and an RSC payload file per post into `.next/`. Because the
functions read from disk *during the build*, the deployed server never touches
`content/posts/` — you could delete the folder from the production image and the site
would still work. That is also why adding a Markdown file requires a rebuild: the list
of pages was fixed when the build ran. With `dynamicParams = false`, the server has no
render function to fall back on for unknown slugs, so it returns the not-found page
with a 404 status directly.

## Common mistakes

- **Reading files with a relative path** like `readFile("content/posts/x.md")`. It
  depends on the working directory; `path.join(process.cwd(), …)` is explicit.
- **Forgetting that new posts need a rebuild.** In Level 2 you'll learn on-demand
  revalidation for content that changes without deploys.
- **Using the slug straight from the URL in a file path.** Validate it.
- **Sorting dates as `Date` objects created from ambiguous strings.** ISO `YYYY-MM-DD`
  strings sort correctly as text and avoid time-zone surprises.
- **Letting a post with bad front matter fail at runtime.** Fail the build instead.

## Exercise

Extend the blog:

1. Add a `tags: nextjs, routing` front-matter field, parse it into an array, and show
   tags on each post.
2. Add `app/tags/[tag]/page.tsx` listing posts with that tag, with
   `generateStaticParams` returning every tag used. Confirm in the route table that
   each tag page is `●`.
3. Add a reading-time estimate (words ÷ 200, rounded up) to the index.
4. Add a `draft: true` field and exclude drafts from both the index and
   `generateStaticParams`. Verify a draft's URL returns 404.
