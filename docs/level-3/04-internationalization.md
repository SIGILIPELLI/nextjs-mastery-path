---
description: "Internationalize a Next.js App Router site: [lang] segments, server-side dictionaries, locale detection in Proxy, Intl formatting and hreflang."
---

# 04 · Internationalization

Next.js's App Router doesn't ship a built-in i18n system (the Pages Router had an
`i18n` config key; the App Router does not). Instead it gives you the primitives —
dynamic segments, Server Components and Proxy — to build one, or to plug in a library.
Building the simple version yourself first makes any library easy to understand.

## The architecture

1. **Locale in the URL**: `/en/about`, `/de/about`. Each language has its own
   crawlable, shareable URL.
2. **A `[lang]` segment** at the top of `app/` holds every localised route.
3. **Dictionaries loaded on the server**, so translations for other languages never
   ship to the browser.
4. **Proxy redirects** locale-less URLs (`/about`) to the best locale
   (Level 2 · 06 has the full proxy).
5. **`hreflang` alternates** tell search engines the pages are translations of each other.

## Dictionaries

```json title="app/[lang]/dictionaries/en.json"
{ "about": { "title": "About us", "body": "We build fast websites." } }
```

```json title="app/[lang]/dictionaries/de.json"
{ "about": { "title": "Über uns", "body": "Wir bauen schnelle Websites." } }
```

```ts title="app/[lang]/dictionaries.ts"
import "server-only";

const dictionaries = {
  en: () => import("./dictionaries/en.json").then((m) => m.default),
  de: () => import("./dictionaries/de.json").then((m) => m.default),
};

export type Locale = keyof typeof dictionaries;
export const locales = Object.keys(dictionaries) as Locale[];
export const hasLocale = (l: string): l is Locale => l in dictionaries;
export const getDictionary = (l: Locale) => dictionaries[l]();
```

Dynamic `import()` means each locale's JSON is loaded only when used. `hasLocale` is a
type guard: after checking, TypeScript narrows the string to `"en" | "de"`.

## A localised page

```tsx title="app/[lang]/about/page.tsx"
import { notFound } from "next/navigation";
import { getDictionary, hasLocale, locales } from "../dictionaries";

export function generateStaticParams() {
  return locales.map((lang) => ({ lang }));
}

export default async function About({ params }: PageProps<"/[lang]/about">) {
  const { lang } = await params;
  if (!hasLocale(lang)) notFound();
  const t = await getDictionary(lang);
  return (
    <main lang={lang}>
      <h1>{t.about.title}</h1>
      <p>{t.about.body}</p>
    </main>
  );
}
```

Built and requested on Next.js 16.3.6:

```text
├   /[lang]/about
│ ├ ● /en/about
│ └ ● /de/about
```

```bash
curl -s localhost:3111/de/about | grep -o "<h1>[^<]*</h1>"
```

```text
<h1>Über uns</h1>
```

Both languages are prerendered static pages — i18n costs nothing at request time.

In a full app, the root layout moves to `app/[lang]/layout.tsx` so it can set
`<html lang={lang}>` properly for every page (screen readers and browsers use it to
choose pronunciation and hyphenation).

## Formatting: use `Intl`, not string concatenation

Dates, numbers, currencies, lists and plurals differ by locale. The built-in `Intl` APIs
handle them on the server with no library:

```ts title="lib/format.ts"
export const formatPrice = (amount: number, locale: string, currency = "EUR") =>
  new Intl.NumberFormat(locale, { style: "currency", currency }).format(amount);

export const formatDate = (d: Date, locale: string) =>
  new Intl.DateTimeFormat(locale, { dateStyle: "long", timeZone: "UTC" }).format(d);

export function itemsLabel(count: number, locale: string, forms: Record<Intl.LDMLPluralRule, string>) {
  const rule = new Intl.PluralRules(locale).select(count);
  return forms[rule].replace("{n}", String(count));
}
```

`formatPrice(1234.5, "de")` gives `1.234,50 €`; `formatPrice(1234.5, "en")` gives
`€1,234.50`. Always pass an explicit `timeZone` when formatting on the server, or the
server's time zone leaks into the output and can cause hydration mismatches when the
same value is formatted in the browser.

## Telling search engines about translations

```tsx title="app/[lang]/about/page.tsx (metadata)"
import type { Metadata } from "next";

export async function generateMetadata({ params }: PageProps<"/[lang]/about">): Promise<Metadata> {
  const { lang } = await params;
  return {
    alternates: {
      canonical: `/${lang}/about`,
      languages: { en: "/en/about", de: "/de/about", "x-default": "/en/about" },
    },
  };
}
```

With `metadataBase` set in the root layout, these become absolute
`<link rel="alternate" hreflang="…">` tags.

## Libraries

For larger apps, libraries such as **next-intl** add message formatting (ICU syntax
for plurals and genders), typed keys, locale-aware `Link` wrappers and middleware/proxy
helpers. They're built on the same ideas as above. Check that a library's current
version documents support for your Next.js major version — i18n libraries often need
updates when routing or proxy APIs change.

## How It Actually Works

`[lang]` is an ordinary dynamic segment; `generateStaticParams` returning every locale
means each localised route is prerendered once per language, so translation lookup
happens at build time. Because dictionaries are imported in Server Components (and
guarded with `server-only`), they're bundled only into server code; the client receives
the already-translated HTML and RSC payload. Proxy handles the one thing the static
pages can't: choosing a locale for a URL without one, based on a cookie or the
`Accept-Language` request header, then redirecting so every page view has an explicit
locale in its URL.

## Common mistakes

- **Detecting language on every page render** from headers — it makes all pages
  dynamic. Detect once in Proxy and put the locale in the URL.
- **Shipping all dictionaries to the client** by importing them in a Client Component.
  Pass only the strings a client component needs as props.
- **Concatenating translated fragments** ("You have " + n + " items") — word order and
  plurals vary. Use whole messages with placeholders.
- **Forgetting `<html lang>`** per locale.
- **Translating slugs inconsistently** so `hreflang` alternates point at 404s.

## Exercise

1. Build `/[lang]/about` for three locales and the locale-detecting proxy from Level 2
   · 06. Test with `curl -H "Accept-Language: de-DE,de;q=0.9" -i localhost:3000/about`.
2. Add a language switcher that links to the same path in each other locale (read the
   current path with `usePathname` in a Client Component and replace the first segment).
3. Show a price and a date on the page formatted with `Intl` for each locale, and add
   `hreflang` alternates. Verify them in the page source.
