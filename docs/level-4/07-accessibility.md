---
description: "Make Next.js apps accessible: route announcements, semantic layouts, accessible Server Action forms, native dialogs, and axe checks in Playwright."
---

# 07 · Accessibility

Accessibility is not a feature bolted on at the end; it's whether people using screen
readers, keyboards, magnification, voice control or reduced motion can use your app.
Much of it is plain HTML done right. But client-side navigation, streaming and
optimistic UI create a few problems specific to apps like the ones built in this
course, and Next.js helps with some of them.

## What Next.js does for you

- **Route announcements.** After a client-side navigation, Next.js announces the new
  page to screen readers through a live region — the
  `<div role="alert" aria-live="assertive" id="__next-route-announcer__">` we met in the
  Level 2 project's Playwright test. It reads the `document.title`, falling back to the
  first `<h1>` or the URL. So **unique, descriptive page titles** (Level 1 · 07) are
  directly an accessibility feature.
- **Scroll and focus** move to the new content on navigation.
- **`<Link>` renders real anchors**, so links work with keyboards, context menus and
  assistive tech.
- **`next/image` requires `alt`** (it warns when missing).
- The ESLint config includes `jsx-a11y` rules for common mistakes.

## Semantic layout

```tsx title="app/layout.tsx (skeleton)"
export default function RootLayout({ children }: LayoutProps<"/">) {
  return (
    <html lang="en">
      <body>
        <a href="#main" className="skip-link">Skip to content</a>
        <header>
          <nav aria-label="Main">{/* NavLinks with aria-current */}</nav>
        </header>
        <main id="main" tabIndex={-1}>{children}</main>
        <footer>{/* ... */}</footer>
      </body>
    </html>
  );
}
```

```css title="app/globals.css (skip link)"
.skip-link { position: absolute; left: -9999px; }
.skip-link:focus { left: 1rem; top: 1rem; z-index: 100; background: var(--bg); padding: 0.5rem 1rem; }
```

Landmarks (`header`, `nav`, `main`, `footer`) let screen-reader users jump around; a skip
link does the same for keyboard users. One `<h1>` per page, headings in order.

## Accessible forms with Server Actions

Every pattern from Level 2 should carry these properties:

```tsx title="app/signup/SignupForm.tsx"
"use client";

import { useActionState } from "react";
import { signup, type SignupState } from "./actions";

export function SignupForm() {
  const [state, action, pending] = useActionState<SignupState, FormData>(signup, {});
  const emailError = state.errors?.email?.[0];
  return (
    <form action={action} noValidate aria-describedby={state.message ? "form-status" : undefined}>
      <label htmlFor="email">Email</label>
      <input
        id="email"
        name="email"
        type="email"
        autoComplete="email"
        aria-invalid={emailError ? true : undefined}
        aria-describedby={emailError ? "email-error" : undefined}
      />
      {emailError && <p id="email-error">{emailError}</p>}

      <button type="submit" aria-disabled={pending}>{pending ? "Creating account…" : "Create account"}</button>
      <p id="form-status" role="status">{state.message ?? ""}</p>
    </form>
  );
}
```

- Every input has a visible `<label>`.
- Errors are linked with `aria-describedby` and flagged with `aria-invalid`.
- `role="status"` (a polite live region) announces success; keep the element in the DOM
  so updates are announced.
- `aria-disabled` rather than `disabled` keeps the button focusable and announced while
  pending (you must still ignore extra clicks — `useActionState` queues them).
- After a failed submit, consider moving focus to the first invalid field.
- `autoComplete` values help password managers and users with motor impairments.

## Dynamic and streamed content

- **Loading states**: mark regions `aria-busy="true"` while loading; announce completion
  only when it matters (a search result count, not every panel).
- **Optimistic UI**: if an action fails and the UI rolls back, announce the failure;
  a silently reverted checkbox is invisible to a screen-reader user.
- **Don't steal focus** when content streams in.

## Dialogs and menus

Use the native `<dialog>` with `showModal()` (as in the Level 3 dashboard): it traps
focus, closes on Escape, and makes the background inert. Return focus to the trigger on
close. For menus, comboboxes and tabs, follow the WAI-ARIA Authoring Practices patterns
or use a well-tested headless component library — hand-rolled widgets are a leading
source of keyboard traps.

## Motion and colour

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; scroll-behavior: auto !important; }
}
```

Ensure text contrast of at least 4.5:1 (3:1 for large text), visible focus indicators
(`:focus-visible` outlines), and never use colour alone to convey state.

## Automated checks

Automated tools catch perhaps a third to a half of issues (estimates vary), so they're
a floor, not a certificate. Add axe to your Playwright suite:

```bash
npm install --save-dev @axe-core/playwright
```

```ts title="e2e/a11y.spec.ts"
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

for (const path of ["/", "/tasks", "/analytics"]) {
  test(`no detectable a11y violations on ${path}`, async ({ page }) => {
    await page.goto(path);
    const results = await new AxeBuilder({ page }).withTags(["wcag2a", "wcag2aa"]).analyze();
    expect(results.violations).toEqual([]);
  });
}
```

Then test manually: navigate with only the keyboard; use a screen reader (VoiceOver on
macOS, NVDA on Windows) for your main flows; zoom to 200%.

## How It Actually Works

Screen readers consume the browser's **accessibility tree**, derived from the DOM:
element roles (from semantic tags or `role`), names (from labels, `alt`, `aria-label`),
states (`aria-invalid`, `aria-current`) and relationships (`aria-describedby`). Live
regions are elements whose content changes the screen reader announces without moving
focus — `role="status"` politely, `role="alert"` immediately. A full page load
naturally resets the reading position and announces the new title; client-side
navigation doesn't, which is why Next.js's route announcer exists: after each navigation
it writes the new title into its live region. Your job is to make sure everything the
tree needs is in the DOM with the right semantics.

## Common mistakes

- **`<div onClick>` instead of `<button>`** — not focusable, not announced.
- **Placeholder text as the only label.**
- **Identical titles on every page**, so route announcements say nothing useful.
- **Removing focus outlines** without a replacement.
- **Custom modals without focus management.**
- **Relying on automated scans alone.**

## Exercise

1. Add a skip link, landmarks and unique titles to your capstone-in-progress. Navigate
   the whole app with only Tab, Shift+Tab, Enter, Space and Escape.
2. Make the Level 2 task form fully accessible as above, then test it with a screen
   reader and write down what's announced on error and on success.
3. Add the axe Playwright test for your main routes and fix every violation it finds.
