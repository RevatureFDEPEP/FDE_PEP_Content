# Next.js 16 App Router Fundamentals

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Today's deliverable lands inside a Next.js 16 App Router project — completing a question authoring page that lives at `app/questions/new/page.tsx`. Before we write a single form field, you need a working mental model of where files live, how routing falls out of the folder tree, and which files get rendered on the server versus shipped to the browser. Get this wrong and you'll spend hours wondering why your form throws "useState is not a function."

## Folder-Based Routing

The App Router (introduced in Next.js 13, refined heavily through 14/15/16) replaces the older `pages/` directory with `app/`. **The folder structure *is* the route table** — there is no separate config.

```
app/
  layout.tsx              # root layout — wraps every route
  page.tsx                # → GET /
  questions/
    layout.tsx            # → wraps every /questions/* route
    page.tsx              # → GET /questions
    new/
      page.tsx            # → GET /questions/new   <-- our target
    [id]/
      page.tsx            # → GET /questions/:id   (dynamic segment)
      edit/
        page.tsx          # → GET /questions/:id/edit
```

Special filenames inside any folder:

| File | Purpose |
|---|---|
| `page.tsx` | The route's UI. Without this, the segment is not routable. |
| `layout.tsx` | Wraps `page.tsx` + nested layouts. Persists across navigation. |
| `loading.tsx` | Streaming/Suspense fallback while the segment resolves. |
| `error.tsx` | Error boundary for the segment. Must be a client component. |
| `not-found.tsx` | Rendered when `notFound()` is called. |
| `route.ts` | API route handler (replaces `pages/api/`). |

## Adding the `/questions/new` Route

For Day 9's deliverable you'll create exactly one new file:

```tsx
// app/questions/new/page.tsx
import { QuestionAuthorForm } from '@/components/questions/QuestionAuthorForm';

export const metadata = {
  title: 'New Question — rev-eval-ai',
};

export default function NewQuestionPage() {
  return (
    <main className="container mx-auto max-w-3xl py-8">
      <h1 className="text-2xl font-semibold mb-6">Author a question</h1>
      <QuestionAuthorForm />
    </main>
  );
}
```

That's the entire page. The form itself is a client component (next topic). The page above is a **server component by default** — it ships zero JavaScript for the heading and shell.

## Layouts Compose Top-Down

Each `layout.tsx` from root downward wraps its children. The root layout is mandatory and owns `<html>` and `<body>`:

```tsx
// app/layout.tsx
import './globals.css';
import { Toaster } from '@/components/ui/sonner';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="min-h-screen bg-background font-sans antialiased">
        {children}
        <Toaster />
      </body>
    </html>
  );
}
```

A nested layout for `/questions/*` could add a sidebar or breadcrumbs without re-rendering the root chrome on navigation.

## Linking and Navigation

Client-side navigation uses the `Link` component — it prefetches in production:

```tsx
import Link from 'next/link';

<Link href="/questions/new" className="text-primary underline">
  Author new question
</Link>
```

For imperative navigation (after a successful submit), use the `useRouter` hook from `next/navigation` (note: **not** `next/router` — that's the legacy Pages Router).

```tsx
'use client';
import { useRouter } from 'next/navigation';

const router = useRouter();
// after POST succeeds:
router.push(`/questions/${createdId}`);
router.refresh(); // re-runs server components to pick up the new row
```

## Example / Worked Scenario

You inherit the PEP variant of `rev-eval-ai`. The `app/questions/` folder exists with a placeholder `page.tsx` listing questions, but `app/questions/new/page.tsx` is a stub that just renders `<p>TODO</p>`. Your job today:

1. Replace the stub with the page shown above.
2. Confirm `pnpm dev` serves it at http://localhost:3000/questions/new.
3. Verify the page renders the shell even before `QuestionAuthorForm` is implemented (you scaffold an empty component first, then fill it in).
4. After a successful submit, `router.push('/questions')` + `router.refresh()` so the new question appears in the list.

## Common Pitfalls

- **Importing `useRouter` from `next/router`.** That's the Pages Router. The App Router exports it from `next/navigation` and the API is different (`router.push` exists but `router.query` does not).
- **Forgetting `router.refresh()` after a mutation.** Without it, the list page shows cached server-rendered HTML from before your POST.
- **Putting `'use client'` at the top of `page.tsx`.** Possible but wasteful — it forces the whole route's JS to ship to the browser. Keep `page.tsx` server-rendered and isolate the client island in a child component.
- **Dynamic segment brackets in the wrong place.** `[id]` is a folder name, not a file name. `app/questions/[id].tsx` is a Pages Router pattern and won't work.

## Key Takeaways
- The folder tree under `app/` is the route table — no router config required.
- Each route segment can have `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, and `route.ts`.
- For Day 9, you create one new file: `app/questions/new/page.tsx`. The form is a child client component.
- Use `next/navigation` (not `next/router`) for hooks; pair `router.push` with `router.refresh` after mutations.

---
*Prerequisites: [05-codebase-navigation-conventions.md](../day-01/05-codebase-navigation-conventions.md), [06-tech-stack-comprehension-identifying-whats-in-use-legacy-or-at-risk.md](../day-01/06-tech-stack-comprehension-identifying-whats-in-use-legacy-or-at-risk.md).*
