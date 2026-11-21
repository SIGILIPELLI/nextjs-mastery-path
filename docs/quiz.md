---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Next.js Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What's the main benefit of Server Components by default?",
    "options": [
      "More hooks",
      "Offline mode",
      "Less client JavaScript and direct, secure access to server data",
      "Faster typing"
    ],
    "correctIndex": 2,
    "explanation": "Components render on the server."
  },
  {
    "question": "You need useState and a click handler. What do you add?",
    "options": [
      "'use client' at the top of the component file",
      "A config flag",
      "getServerSideProps",
      "'use server'"
    ],
    "correctIndex": 0,
    "explanation": "Marks the client boundary."
  },
  {
    "question": "Which file creates the /blog/[slug] page?",
    "options": [
      "app/blog/[slug]/page.tsx",
      "pages/blog.js",
      "app/blog/slug.tsx",
      "routes/blog.ts"
    ],
    "correctIndex": 0,
    "explanation": "Folder names are segments."
  },
  {
    "question": "What does a route group like (marketing) do to the URL?",
    "options": [
      "Adds /marketing",
      "Requires auth",
      "Creates a layout",
      "Nothing: it only organizes files"
    ],
    "correctIndex": 3,
    "explanation": "Parentheses are omitted."
  },
  {
    "question": "Where should a slow, independent widget be wrapped so the page can stream?",
    "options": [
      "In a Suspense boundary with a fallback",
      "try/catch",
      "useEffect",
      "Middleware"
    ],
    "correctIndex": 0,
    "explanation": "Progressive rendering."
  },
  {
    "question": "Which file shows an instant loading state while a segment fetches?",
    "options": [
      "error.tsx",
      "not-found.tsx",
      "template.tsx",
      "loading.tsx"
    ],
    "correctIndex": 3,
    "explanation": "Automatic Suspense."
  },
  {
    "question": "A server action receives form data. What must you do first?",
    "options": [
      "Validate and authorize it: actions are public endpoints",
      "Nothing",
      "Trust it",
      "Hide the button"
    ],
    "correctIndex": 0,
    "explanation": "Treat input as untrusted."
  },
  {
    "question": "After a mutation, how do you refresh a cached list page?",
    "options": [
      "location.reload()",
      "Wait",
      "revalidatePath or revalidateTag in the action",
      "setState"
    ],
    "correctIndex": 2,
    "explanation": "Cache invalidation."
  },
  {
    "question": "What does `revalidate: 60` mean for fetched data?",
    "options": [
      "Cached forever",
      "Refetched every request",
      "Never cached",
      "It may be regenerated at most once per 60 seconds (time-based ISR)"
    ],
    "correctIndex": 3,
    "explanation": "Stale-while-revalidate."
  },
  {
    "question": "When is a route handler (route.ts) the right tool?",
    "options": [
      "Form posts only",
      "For external clients, webhooks or non-React consumers",
      "Layouts",
      "Styling"
    ],
    "correctIndex": 1,
    "explanation": "Server actions are for your own UI."
  },
  {
    "question": "Where should authorization be enforced?",
    "options": [
      "CSS",
      "Image config",
      "Near data access on the server (not only in middleware)",
      "Only client"
    ],
    "correctIndex": 2,
    "explanation": "Defense in depth."
  },
  {
    "question": "Why keep filters and pagination in search params?",
    "options": [
      "They're shareable, bookmarkable and work with back/forward",
      "Smaller",
      "Law",
      "Faster"
    ],
    "correctIndex": 0,
    "explanation": "URL as state."
  },
  {
    "question": "Which strategy gives a static shell with dynamic holes streamed in?",
    "options": [
      "CSR",
      "SSR only",
      "Pure SSG",
      "Partial prerendering"
    ],
    "correctIndex": 3,
    "explanation": "Hybrid."
  },
  {
    "question": "A product image causes layout shift. What fixes CLS?",
    "options": [
      "Use CSS",
      "Add JS",
      "next/image with width/height (or fill) to reserve space",
      "Remove image"
    ],
    "correctIndex": 2,
    "explanation": "Core Web Vitals."
  },
  {
    "question": "Which Core Web Vitals does Next.js help with?",
    "options": [
      "TTFB only",
      "LCP, INP and CLS",
      "FCP only",
      "None"
    ],
    "correctIndex": 1,
    "explanation": "Measure with Lighthouse and RUM."
  },
  {
    "question": "A secret key must never reach the browser. How are env vars named?",
    "options": [
      "Without the NEXT_PUBLIC_ prefix and used only in server code",
      "In the layout",
      "With the prefix",
      "In public/"
    ],
    "correctIndex": 0,
    "explanation": "Prefixed vars are inlined."
  },
  {
    "question": "A route needs Node-only database drivers. Which runtime?",
    "options": [
      "None",
      "Node.js runtime (Edge has limited APIs)",
      "Edge",
      "Static"
    ],
    "correctIndex": 1,
    "explanation": "export const runtime."
  },
  {
    "question": "You host on a plain CDN with no server. What mode?",
    "options": [
      "SSR",
      "Edge",
      "Static export (output: 'export'), accepting no server features",
      "Standalone"
    ],
    "correctIndex": 2,
    "explanation": "No actions or handlers."
  },
  {
    "question": "How do you upgrade major versions safely?",
    "options": [
      "Skip",
      "Rewrite",
      "Latest blind",
      "Read the guide, run codemods on a branch and rely on tests and CI"
    ],
    "correctIndex": 3,
    "explanation": "@next/codemod."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### File-system routing and params

Implement toRoute (strip app/, route groups and the leaf file) and match: static segments, [x], [...x] and [[...x]] with parameters, preferring routes with more static segments, and 404 for unknown URLs. It models how the App Router maps folders to URLs.

**Starter code**

```javascript
// App Router style file-system routes. Files are paths like "app/blog/[slug]/page.tsx".
const files = [
  "app/page.tsx",
  "app/(marketing)/about/page.tsx",
  "app/blog/page.tsx",
  "app/blog/[slug]/page.tsx",
  "app/shop/[...parts]/page.tsx",
  "app/docs/[[...path]]/page.tsx",
  "app/api/health/route.ts",
];

function toRoute(file) {
  // strip the leading "app/", drop the leaf (page.* => kind "page", route.* => "handler") and route
  // groups "(name)"; return { pattern: "/" + remaining segments joined, kind, segs }
  return { pattern: "/", kind: "page", segs: [] };
}

const routes = files.map(toRoute);

function match(url) {
  // static segments match literally; [x] matches one segment; [...x] one or more; [[...x]] zero or more;
  // if several routes match prefer the one with more static segments. Return
  // "<kind> <pattern> <params as JSON>" or "404 not-found"
  return "404 not-found";
}

console.log(routes.map((r) => r.pattern).join("  "));
for (const u of ["/", "/about", "/blog", "/blog/hello-world", "/shop/a/b/c", "/docs", "/docs/x/y", "/api/health", "/nope"]) {
  console.log(u.padEnd(18), match(u));
}
```

**Expected output**

```text
/  /about  /blog  /blog/[slug]  /shop/[...parts]  /docs/[[...path]]  /api/health
/                  page / {}
/about             page /about {}
/blog              page /blog {}
/blog/hello-world  page /blog/[slug] {"slug":"hello-world"}
/shop/a/b/c        page /shop/[...parts] {"parts":["a","b","c"]}
/docs              page /docs/[[...path]] {"path":[]}
/docs/x/y          page /docs/[[...path]] {"path":["x","y"]}
/api/health        handler /api/health {}
/nope              404 not-found
```

<details><summary>Show solution and explanation</summary>

<pre><code>// App Router style file-system routes. Files are paths like &quot;app/blog/[slug]/page.tsx&quot;.
const files = [
  &quot;app/page.tsx&quot;,
  &quot;app/(marketing)/about/page.tsx&quot;,
  &quot;app/blog/page.tsx&quot;,
  &quot;app/blog/[slug]/page.tsx&quot;,
  &quot;app/shop/[...parts]/page.tsx&quot;,
  &quot;app/docs/[[...path]]/page.tsx&quot;,
  &quot;app/api/health/route.ts&quot;,
];

function toRoute(file) {
  const segs = file.replace(/^app\//, &quot;&quot;).split(&quot;/&quot;);
  const leaf = segs.pop();                                  // page.tsx or route.ts
  const kept = segs.filter((s) =&gt; !(s.startsWith(&quot;(&quot;) &amp;&amp; s.endsWith(&quot;)&quot;)));   // route groups vanish
  return { pattern: &quot;/&quot; + kept.join(&quot;/&quot;), kind: leaf.startsWith(&quot;page&quot;) ? &quot;page&quot; : &quot;handler&quot;, segs: kept };
}

const routes = files.map(toRoute);

function match(url) {
  const parts = url.split(&quot;/&quot;).filter(Boolean);
  let best = null;
  for (const r of routes) {
    const params = {};
    let i = 0, ok = true;
    for (const [k, seg] of r.segs.entries()) {
      const optional = /^\[\[\.\.\.(.+)\]\]$/.exec(seg);
      const rest = /^\[\.\.\.(.+)\]$/.exec(seg);
      const dyn = /^\[(.+)\]$/.exec(seg);
      if (optional) { params[optional[1]] = parts.slice(i); i = parts.length; }
      else if (rest) { if (i &gt;= parts.length) { ok = false; break; } params[rest[1]] = parts.slice(i); i = parts.length; }
      else if (dyn) { if (i &gt;= parts.length) { ok = false; break; } params[dyn[1]] = parts[i++]; }
      else { if (parts[i] !== seg) { ok = false; break; } i++; }
    }
    if (ok &amp;&amp; i === parts.length) {
      const specificity = r.segs.filter((s) =&gt; !s.startsWith(&quot;[&quot;)).length;   // static segments win
      if (!best || specificity &gt; best.specificity) best = { pattern: r.pattern, kind: r.kind, params, specificity };
    }
  }
  return best ? `${best.kind} ${best.pattern} ${JSON.stringify(best.params)}` : &quot;404 not-found&quot;;
}

console.log(routes.map((r) =&gt; r.pattern).join(&quot;  &quot;));
for (const u of [&quot;/&quot;, &quot;/about&quot;, &quot;/blog&quot;, &quot;/blog/hello-world&quot;, &quot;/shop/a/b/c&quot;, &quot;/docs&quot;, &quot;/docs/x/y&quot;, &quot;/api/health&quot;, &quot;/nope&quot;]) {
  console.log(u.padEnd(18), match(u));
}
</code></pre>

<p>Folders become URL segments, parenthesized groups disappear, brackets create params and the most specific route wins. This models the idea, not Next&#x27;s actual router. Verified by running the solution with Node.</p>
</details>

### Time-based and tag-based revalidation

Implement DataCache.get so entries are served from cache while fresh, refetched after `revalidate` seconds or after revalidateTag invalidates them, and counted as origin fetches. A fake clock drives the scenario.

**Starter code**

```javascript
// A tiny model of the Next.js data cache: time-based revalidation (ISR) plus on-demand tag invalidation.
class DataCache {
  constructor(clock) { this.clock = clock; this.entries = new Map(); this.fetches = 0; }

  async get(key, fetcher, { revalidate = Infinity, tags = [] } = {}) {
    // return { value, source } where source is "cache" while the entry is fresh (age < revalidate seconds and
    // not invalidated), "origin" for the first fetch of a key and "revalidated" when an existing entry is
    // refetched; store { value, at: now, tags, invalid: false } and count origin calls in this.fetches
    return { value: undefined, source: "origin" };
  }

  revalidateTag(tag) {
    for (const e of this.entries.values()) if (e.tags.includes(tag)) e.invalid = true;
  }
}

(async () => {
  const clock = { now: 0 };
  const cache = new DataCache(clock);
  let version = 1;
  const fetchPosts = async () => `posts v${version}`;
  const show = async (label, key, opts) => {
    const r = await cache.get(key, fetchPosts, opts);
    console.log(label.padEnd(30), r.source.padEnd(12), r.value);
  };

  await show("first request", "posts", { revalidate: 60, tags: ["posts"] });
  version = 2;
  clock.now = 30_000;
  await show("30s later (still fresh)", "posts", { revalidate: 60, tags: ["posts"] });
  clock.now = 61_000;
  await show("61s later (expired)", "posts", { revalidate: 60, tags: ["posts"] });
  version = 3;
  await show("immediately again", "posts", { revalidate: 60, tags: ["posts"] });
  cache.revalidateTag("posts");
  await show("after revalidateTag('posts')", "posts", { revalidate: 60, tags: ["posts"] });
  await show("force-cache forever", "config", { revalidate: Infinity });
  clock.now = 10_000_000;
  await show("much later, still cached", "config", { revalidate: Infinity });
  console.log("origin fetches:", cache.fetches);
})();
```

**Expected output**

```text
first request                  origin       posts v1
30s later (still fresh)        cache        posts v1
61s later (expired)            revalidated  posts v2
immediately again              cache        posts v2
after revalidateTag('posts')   revalidated  posts v3
force-cache forever            origin       posts v3
much later, still cached       cache        posts v3
origin fetches: 4
```

<details><summary>Show solution and explanation</summary>

<pre><code>// A tiny model of the Next.js data cache: time-based revalidation (ISR) plus on-demand tag invalidation.
class DataCache {
  constructor(clock) { this.clock = clock; this.entries = new Map(); this.fetches = 0; }

  async get(key, fetcher, { revalidate = Infinity, tags = [] } = {}) {
    const e = this.entries.get(key);
    const now = this.clock.now;
    if (e &amp;&amp; !e.invalid &amp;&amp; now - e.at &lt; revalidate * 1000) return { value: e.value, source: &quot;cache&quot; };
    const stale = e ? e.value : undefined;
    this.fetches++;
    const value = await fetcher();
    this.entries.set(key, { value, at: now, tags, invalid: false });
    return { value, source: e ? &quot;revalidated&quot; : &quot;origin&quot;, stale };
  }

  revalidateTag(tag) {
    for (const e of this.entries.values()) if (e.tags.includes(tag)) e.invalid = true;
  }
}

(async () =&gt; {
  const clock = { now: 0 };
  const cache = new DataCache(clock);
  let version = 1;
  const fetchPosts = async () =&gt; `posts v${version}`;
  const show = async (label, key, opts) =&gt; {
    const r = await cache.get(key, fetchPosts, opts);
    console.log(label.padEnd(30), r.source.padEnd(12), r.value);
  };

  await show(&quot;first request&quot;, &quot;posts&quot;, { revalidate: 60, tags: [&quot;posts&quot;] });
  version = 2;
  clock.now = 30_000;
  await show(&quot;30s later (still fresh)&quot;, &quot;posts&quot;, { revalidate: 60, tags: [&quot;posts&quot;] });
  clock.now = 61_000;
  await show(&quot;61s later (expired)&quot;, &quot;posts&quot;, { revalidate: 60, tags: [&quot;posts&quot;] });
  version = 3;
  await show(&quot;immediately again&quot;, &quot;posts&quot;, { revalidate: 60, tags: [&quot;posts&quot;] });
  cache.revalidateTag(&quot;posts&quot;);
  await show(&quot;after revalidateTag(&#x27;posts&#x27;)&quot;, &quot;posts&quot;, { revalidate: 60, tags: [&quot;posts&quot;] });
  await show(&quot;force-cache forever&quot;, &quot;config&quot;, { revalidate: Infinity });
  clock.now = 10_000_000;
  await show(&quot;much later, still cached&quot;, &quot;config&quot;, { revalidate: Infinity });
  console.log(&quot;origin fetches:&quot;, cache.fetches);
})();
</code></pre>

<p>Next&#x27;s data cache serves fresh entries, revalidates after a time window (ISR) and can be invalidated on demand by tag, so unnecessary origin fetches are avoided. This is a simplified model. Verified by running the solution with Node.</p>
</details>
