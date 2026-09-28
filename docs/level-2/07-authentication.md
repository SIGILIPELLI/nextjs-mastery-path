---
description: "Authentication in the App Router: session cookies, a signed-session helper with jose, a data access layer that checks auth, and where Auth.js fits."
---

# 07 · Authentication Patterns

Authentication ("who are you?") and authorisation ("are you allowed to do this?")
touch every layer of a Next.js app: Proxy, layouts, pages, Server Actions and Route
Handlers. This lesson builds a small but sound session system so you can see every
moving part, then explains where libraries and hosted services fit.

!!! warning "Use a maintained library or provider in production"
    Rolling your own login is educational and fine for internal tools, but password
    storage, account recovery, OAuth, MFA and brute-force protection are easy to get
    subtly wrong. For production, strongly consider Auth.js, Better Auth, or a hosted
    provider (Clerk, Auth0, WorkOS, Supabase Auth…). The *architecture* in this lesson
    — sessions in cookies, checks in a data access layer — applies to all of them.

## The pieces

1. **Credentials check** — verify a password (hashed with a slow algorithm such as
   scrypt, bcrypt or Argon2) or complete an OAuth flow.
2. **Session** — once verified, issue something the browser sends back on every
   request: an `HttpOnly` cookie holding either a signed token (stateless) or a random
   session ID that points to a database row (stateful).
3. **Verification** — on each request, read the cookie and establish the user.
4. **Authorisation** — before reading or writing data, check the user may do so.

## A stateless signed session with `jose`

`jose` is a small, standards-based JWT library that works in Node.js and other
runtimes.

```bash
npm install jose server-only
```

```ts title="lib/session.ts"
import "server-only";
import { SignJWT, jwtVerify } from "jose";
import { cookies } from "next/headers";
import { cache } from "react";

const key = new TextEncoder().encode(process.env.SESSION_SECRET); // at least 32 random bytes
const COOKIE = "session";
const MAX_AGE = 60 * 60 * 24 * 7; // 7 days

export type Session = { userId: string; role: "user" | "admin" };

export async function createSession(session: Session) {
  const token = await new SignJWT(session)
    .setProtectedHeader({ alg: "HS256" })
    .setIssuedAt()
    .setExpirationTime(`${MAX_AGE}s`)
    .sign(key);
  (await cookies()).set(COOKIE, token, {
    httpOnly: true,                                 // not readable by JavaScript
    secure: process.env.NODE_ENV === "production",  // HTTPS only in production
    sameSite: "lax",                                // not sent on cross-site POSTs
    path: "/",
    maxAge: MAX_AGE,
  });
}

export async function deleteSession() {
  (await cookies()).delete(COOKIE);
}

// cache(): verify once per request even if many components ask
export const getSession = cache(async (): Promise<Session | null> => {
  const token = (await cookies()).get(COOKIE)?.value;
  if (!token) return null;
  try {
    const { payload } = await jwtVerify(token, key, { algorithms: ["HS256"] });
    return { userId: String(payload.userId), role: payload.role === "admin" ? "admin" : "user" };
  } catch {
    return null; // expired or tampered
  }
});
```

Generate a secret with `openssl rand -base64 32` and put it in `.env.local` as
`SESSION_SECRET=...`. Never commit it.

Trade-off: a stateless token can't be revoked before it expires (logging out deletes
the cookie in *that* browser only). If you need "log out everywhere" or instant
revocation, store sessions in a database and put only a random ID in the cookie.

## Login and logout actions

```ts title="app/login/actions.ts"
"use server";

import { redirect } from "next/navigation";
import { z } from "zod";
import { createSession, deleteSession } from "@/lib/session";
import { verifyPassword, findUserByEmail } from "@/lib/users"; // your own: scrypt/bcrypt compare

const Login = z.object({ email: z.email(), password: z.string().min(1) });

export async function login(_prev: { error?: string }, formData: FormData) {
  const parsed = Login.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { error: "Invalid email or password." };

  const user = await findUserByEmail(parsed.data.email);
  const ok = user && (await verifyPassword(parsed.data.password, user.passwordHash));
  if (!ok) return { error: "Invalid email or password." }; // same message: don't reveal which was wrong

  await createSession({ userId: user.id, role: user.role });
  redirect("/dashboard");
}

export async function logout() {
  await deleteSession();
  redirect("/login");
}
```

## Where to check: a data access layer

The most important decision is **where** authorisation lives. Put it in the functions
that touch data, not only in pages:

```ts title="lib/dal.ts"
import "server-only";
import { redirect } from "next/navigation";
import { getSession } from "./session";
import { db } from "@/db";

export async function requireUser() {
  const session = await getSession();
  if (!session) redirect("/login");
  return session;
}

export async function getMyInvoices() {
  const { userId } = await requireUser();
  return db.query.invoices.findMany({ where: (i, { eq }) => eq(i.ownerId, userId) });
}

export async function deleteInvoice(id: number) {
  const { userId, role } = await requireUser();
  const invoice = await db.query.invoices.findFirst({ where: (i, { eq }) => eq(i.id, id) });
  if (!invoice) throw new Error("Not found");
  if (invoice.ownerId !== userId && role !== "admin") throw new Error("Forbidden");
  // ...delete
}
```

Now a page, a Server Action and a Route Handler all get the same protection simply by
calling `getMyInvoices()` — nobody can forget a check, because there's no unchecked way
to get the data.

Layered checks, from cheapest to most important:

| Layer | Check | Purpose |
|---|---|---|
| Proxy | Cookie present? | Fast redirect of obviously signed-out users |
| Layout / page | `requireUser()` | Correct UI; redirect to login |
| **Data access layer** | Session valid *and* user may access this record | **Actual security** |

!!! danger "Layouts are not a security boundary"
    Because layouts don't re-render on client navigation (Level 1 · 03), and because
    Server Actions and Route Handlers don't go through your layouts at all, an auth
    check in a layout alone is not enough. Check in the data layer.

## Auth.js and other libraries

**Auth.js** (formerly NextAuth.js) handles OAuth providers ("Sign in with GitHub"),
email links and credentials, and exposes helpers such as `auth()`, `signIn()` and
`signOut()` that plug into Server Components, Server Actions and Proxy. Its API has
changed between major versions (v4 → v5 was a significant rewrite), and v5 spent a
long time in beta, so follow the documentation for the exact version you install
rather than blog posts. Whichever library you choose, keep the data-access-layer
pattern above: call the library's "get session" function inside your DAL.

## How It Actually Works

A cookie is just a header. On login, the server's response includes
`Set-Cookie: session=<token>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800`.
The browser stores it and attaches `Cookie: session=<token>` to every later request to
your origin — including Server Action POSTs and RSC navigation requests. The signed
token contains the claims in plain base64 plus an HMAC signature; `jwtVerify`
recomputes the HMAC with your secret, so any edit to the claims invalidates it (that's
also why the claims must not contain secrets — anyone can read them). `HttpOnly` keeps
page scripts (and therefore most XSS payloads) from reading the token, and `SameSite=Lax`
stops browsers sending it on cross-site form posts, which together with Next.js's
origin check on Server Actions defends against CSRF.

Reading `cookies()` in a component makes that route dynamic — a personalised page
can't be prerendered once for everyone — which is why authenticated pages show as `ƒ`
in the route table (or stream inside a Suspense boundary under Cache Components).

## Common mistakes

- **Checking auth only in Proxy or only in a layout.**
- **Storing the session in `localStorage`.** Readable by any script on the page; use an
  `HttpOnly` cookie.
- **Putting sensitive data in JWT claims** — they're signed, not encrypted.
- **Different error messages for "no such user" and "wrong password"** — lets attackers
  enumerate accounts.
- **Hashing passwords with SHA-256.** Use a slow, salted algorithm (scrypt is in Node's
  `crypto`).
- **Caching personalised data in a shared cache.**

## Exercise

1. Implement `lib/session.ts`, a login page with a hard-coded demo user (password hash
   created with `crypto.scryptSync`), and a `/dashboard` page that calls
   `requireUser()`.
2. Decode your session cookie at jwt.io (or with `atob` on the middle segment) and
   confirm the claims are readable. Change one character and confirm `getSession()`
   returns `null`.
3. Write a Server Action that deletes a record, and prove with a `curl` request carrying
   a different user's cookie that the DAL refuses it.
