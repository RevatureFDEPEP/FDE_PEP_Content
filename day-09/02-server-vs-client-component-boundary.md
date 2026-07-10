# Server vs Client Component Boundary

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
This is the single most important Next.js 16 concept, and the one most likely to bite you on Day 9. Our question authoring form is interactive (state, handlers, controlled inputs, drag-drop), so it must be a **client component**, while the page shell that renders it stays a **server component**. Drawing this boundary deliberately is what separates a fast App Router app from a "why does my whole page ship 800KB of JavaScript" app.

## The Two Worlds

| | Server Component | Client Component |
|---|---|---|
| Runs on | Node.js server only | Server (for initial HTML) + browser |
| Default? | Yes (in `app/`) | No — must opt in |
| Opt-in directive | none | `'use client'` at top of file |
| Can use `useState`/`useEffect`? | No | Yes |
| Can use event handlers (`onClick`)? | No | Yes |
| Can be `async`/await DB queries? | Yes | No |
| Can import server-only code (fs, db)? | Yes | No |
| Ships JS to browser? | No | Yes |

A client component is not "client-only" — it still renders to HTML on the server on the initial request. The `'use client'` directive just marks the **boundary** at which JavaScript hydration starts.

## The Boundary Rule

`'use client'` at the top of a file marks that file **and every module it imports** as part of the client bundle. The crucial implication:

> **You don't sprinkle `'use client'` everywhere. You push it as far down the tree as possible.**

If `app/questions/new/page.tsx` is `'use client'`, every component it imports gets bundled too. If only `QuestionAuthorForm.tsx` is `'use client'`, the page shell and any siblings stay server-rendered.

## What Crosses the Boundary

Props passed from a server parent to a client child must be **serializable**:

- Yes: strings, numbers, booleans, null, arrays/objects of those, plain Dates.
- No: functions, class instances, Maps/Sets, JSX with components (children is special-cased and works).
- Special: `children` prop can be server-rendered content "sloted" into a client component.

```tsx
// app/questions/new/page.tsx  (server component — no directive)
import { QuestionAuthorForm } from '@/components/questions/QuestionAuthorForm';
import { getCurrentUser } from '@/lib/auth';

export default async function NewQuestionPage() {
  const user = await getCurrentUser(); // server-only call — fine here
  return (
    <main>
      <h1>Author a question</h1>
      {/* user is plain JSON → safe to pass to a client component */}
      <QuestionAuthorForm authorEmail={user.email} />
    </main>
  );
}
```

```tsx
// components/questions/QuestionAuthorForm.tsx
'use client';

import { useForm } from 'react-hook-form';

export function QuestionAuthorForm({ authorEmail }: { authorEmail: string }) {
  const form = useForm({ /* ... */ });
  // useState, onClick, react-hook-form all valid here
  return <form>{/* ... */}</form>;
}
```

## Composition Pattern: Server Shell Around Client Island

Prefer a small client island inside a server shell over making the whole page client:

```tsx
// Good — only the interactive bits hydrate
<ServerShell>
  <ServerSidebar />
  <ClientForm />         {/* client island */}
  <ServerFooter />
</ServerShell>
```

You can even pass server-rendered JSX as `children` to a client component:

```tsx
// FormCard.tsx
'use client';
export function FormCard({ children }: { children: React.ReactNode }) {
  const [collapsed, setCollapsed] = useState(false);
  return (
    <div>
      <button onClick={() => setCollapsed(c => !c)}>Toggle</button>
      {!collapsed && children}
    </div>
  );
}

// page.tsx (server)
<FormCard>
  <ServerOnlyContent /> {/* still server-rendered, not hydrated */}
</FormCard>
```

## Example / Worked Scenario

For the question authoring page:

- `app/questions/new/page.tsx` — **server**. Just renders a heading and the form.
- `components/questions/QuestionAuthorForm.tsx` — **client**. Uses `useForm`, `useFieldArray`, `onSubmit` handlers, `useState` for upload progress.
- `components/questions/ChoiceFieldArray.tsx` — **client** (imported by the form; no need to re-declare directive, but doing so is harmless).
- `components/questions/ImageDropzone.tsx` — **client**. Needs `onDragOver`, `onDrop`, `XMLHttpRequest`.
- `components/ui/Button.tsx` (shadcn) — **client** (shadcn adds the directive for you).

The page ships only the form-subtree's JS to the browser. The heading and any future server-fetched sidebar content remain HTML-only.

## Common Pitfalls

- **`'use client'` at the top of every file "just in case."** This silently expands the client bundle. Audit periodically with `pnpm build` and look at the route's First Load JS.
- **Passing a function prop from a server component to a client component.** TypeScript will compile; Next.js will throw at runtime: "Functions cannot be passed directly to Client Components." If you need a callback, define it inside a client component.
- **Calling an async function in a client component body.** Client components can't be `async function`. Use `useEffect` + an inner async function, or React Query / SWR.
- **Importing a server-only utility (e.g., `fs`, `pg` client) from a client component.** The bundler will pull it into the client and fail at build. Use `import 'server-only'` at the top of server-only modules to fail fast with a clear error.

## Key Takeaways
- Components in `app/` are server components by default; opt into client with `'use client'` at the top of the file.
- Push the boundary as far down the tree as possible — a small client island inside a server shell is the goal.
- Props crossing the boundary must be serializable JSON; functions don't cross.
- For Day 9: page = server, form + dropzone = client. Don't `'use client'` the page itself.

---
*Prerequisites: [01-nextjs-16-app-router-fundamentals.md](01-nextjs-16-app-router-fundamentals.md).*
