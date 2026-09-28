---
description: "Handle forms with Next.js Server Actions: 'use server', useActionState for errors and pending UI, Zod validation, bind and progressive enhancement."
---

# 03 · Server Actions & Forms

Before the App Router, submitting a form in React meant: an `onSubmit` handler, a
`fetch` to an API route you also had to write, JSON parsing on both sides, and manual
loading and error state. **Server Actions** collapse that into one function that runs
on the server and can be passed straight to a form.

## The smallest Server Action

```tsx title="app/newsletter/page.tsx"
import { redirect } from "next/navigation";
import { addSubscriber } from "@/lib/subscribers";

export default function NewsletterPage() {
  async function subscribe(formData: FormData) {
    "use server";
    const email = String(formData.get("email") ?? "");
    await addSubscriber(email);
    redirect("/newsletter/thanks");
  }

  return (
    <form action={subscribe}>
      <label htmlFor="email">Email</label>
      <input id="email" name="email" type="email" required />
      <button>Subscribe</button>
    </form>
  );
}
```

The `"use server"` directive at the top of the function body marks it as a Server
Action. Passed to `<form action>`, it receives the form's `FormData`. This form works
**even before JavaScript loads** — it posts like an ordinary HTML form — and once React
has hydrated, submissions happen without a full page reload.

For anything beyond a demo, put actions in their own file with `"use server"` at the
top; every exported async function in that file becomes an action, and Client
Components can import them:

```ts title="app/notes/actions.ts"
"use server";

import { revalidatePath } from "next/cache";

export type State = { error?: string; ok?: boolean };

const notes: string[] = []; // stand-in for a database

export async function listNotes() {
  return [...notes];
}

export async function addNote(prev: State, formData: FormData): Promise<State> {
  const text = String(formData.get("text") ?? "").trim();
  if (text.length < 3) return { error: "Note must be at least 3 characters." };
  notes.push(text);
  revalidatePath("/notes");
  return { ok: true };
}
```

## Errors and pending state with `useActionState`

To show validation messages and a "Saving…" state, wrap the action with React's
`useActionState` in a Client Component. The action then takes the **previous state** as
its first argument and returns the next state:

```tsx title="app/notes/NoteForm.tsx"
"use client";

import { useActionState } from "react";
import { addNote, type State } from "./actions";

export function NoteForm() {
  const [state, formAction, pending] = useActionState<State, FormData>(addNote, {});
  return (
    <form action={formAction}>
      <label htmlFor="text">Note</label>
      <input id="text" name="text" />
      <button disabled={pending}>{pending ? "Saving…" : "Add"}</button>
      {state.error && <p role="alert">{state.error}</p>}
    </form>
  );
}
```

```tsx title="app/notes/page.tsx"
import { connection } from "next/server";
import { listNotes } from "./actions";
import { NoteForm } from "./NoteForm";

export default async function NotesPage() {
  await connection();
  const notes = await listNotes();
  return (
    <main>
      <NoteForm />
      <ul>{notes.map((n, i) => <li key={i}>{n}</li>)}</ul>
    </main>
  );
}
```

After a successful action, `revalidatePath("/notes")` tells Next.js the page's data is
stale; the response to the action carries the fresh page, so the list updates in the
same round trip. Also note React 19 **resets uncontrolled form fields** after a
successful action submission — when we exercised a form like this in a browser, the
input was empty again after adding an item. If you need to keep values after a
validation error, return them in the state and use them as `defaultValue`.

## Validate on the server — always

Client-side checks (`required`, `type="email"`) are for convenience. The action is
reachable with a hand-crafted POST, so validate there. A schema library such as Zod
keeps it tidy:

```ts title="app/signup/actions.ts"
"use server";

import { z } from "zod";

const Signup = z.object({
  name: z.string().trim().min(1, "Name is required").max(80),
  email: z.email("Enter a valid email"),
  plan: z.enum(["free", "pro"]),
});

export type SignupState = { errors?: Partial<Record<"name" | "email" | "plan", string[]>>; message?: string };

export async function signup(_prev: SignupState, formData: FormData): Promise<SignupState> {
  const parsed = Signup.safeParse(Object.fromEntries(formData));
  if (!parsed.success) return { errors: z.flattenError(parsed.error).fieldErrors };
  // ...create the account with parsed.data
  return { message: `Welcome, ${parsed.data.name}!` };
}
```

(`z.email()` and `z.flattenError()` are Zod 4 APIs; Zod 3 used `z.string().email()`
and `error.flatten()`.)

## Passing extra arguments with `bind`

A delete button needs the ID of the row. Don't put it in a hidden input you then trust
blindly — bind it:

```tsx
import { deleteTask } from "./actions";

<form action={deleteTask.bind(null, task.id)}>
  <button>Delete</button>
</form>
```

```ts title="app/tasks/actions.ts (excerpt)"
export async function deleteTask(id: number) {
  // authorise here: does the current user own task `id`?
  // ...delete
}
```

Bound arguments are serialised into the form, so they are **still client-controlled**
— binding is about convenience, not security. Authorisation checks inside the action
are what protect you.

## Calling actions outside forms

An action is an async function, so a Client Component can call it from an event
handler, typically inside `startTransition` so React tracks the pending state:

```tsx
"use client";
import { useTransition } from "react";
import { toggleFavourite } from "./actions";

export function Star({ id, on }: { id: number; on: boolean }) {
  const [pending, start] = useTransition();
  return (
    <button disabled={pending} aria-pressed={on} onClick={() => start(() => toggleFavourite(id))}>
      {on ? "★" : "☆"}
    </button>
  );
}
```

For instant feedback before the server responds, React's `useOptimistic` lets you show
the expected result and roll back automatically if the action fails.

## How It Actually Works

When the compiler sees `"use server"`, it replaces the function in the client bundle
with a **reference**: an ID (derived from the module and export) plus a small stub.
Calling the stub sends a `POST` to the current page's URL with a `Next-Action` header
carrying that ID and the arguments serialised in the body (multipart form data for
forms). On the server, Next.js looks the ID up in a manifest of actions, deserialises
the arguments, runs the function, and responds with the action's return value *and*,
if you revalidated, the updated RSC payload for the page — so UI and data update
together in one round trip.

For a `<form action={serverAction}>` rendered by a Server Component, React also emits a
real `action` attribute and hidden fields containing the action reference, which is why
the form works before hydration. Closures are handled by encrypting captured variables
into the form so they can't be read in transit — but captured values should still
never include secrets.

Two consequences follow directly: every exported action is a **public endpoint**
anyone can call with any arguments, and actions run **sequentially** per client by
design (they are for mutations, not parallel data fetching).

## Common mistakes

- **No authorisation inside the action.** Hiding the button isn't access control.
  Level 3 · 08 returns to this.
- **Trusting `formData` or bound arguments.** Parse and validate every field.
- **Using actions to fetch data for rendering.** Read data in Server Components; use
  actions for writes.
- **Forgetting to revalidate**, so the UI keeps showing old data after a successful
  write (lesson 04).
- **Calling `redirect()` inside `try/catch`.** It throws to work; redirect after the
  try block.
- **Exporting non-async functions from a `"use server"` file.** Only async functions
  are allowed there.

## Exercise

1. Build the notes example. Disable JavaScript in your browser and confirm the form
   still adds notes (the page reloads fully).
2. Replace the length check with a Zod schema and show per-field errors with
   `aria-describedby` linking each input to its message.
3. Open the Network panel, submit the form with JavaScript enabled, and find the
   `Next-Action` request header. Then use `curl` to call the same action ID with an
   empty body and observe that your server-side validation is what stops it.
