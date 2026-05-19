# Client Components For Stateful Interactivity

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The page is a server component (Topic 2). The candidate clicks an option, sees it highlight, clicks "Next", sees a different question. None of that — the click handlers, the highlighted state, the local current-index — can live in a server component, because server components don't run in the browser. Topic 3 is about drawing the boundary correctly: what stays on the server, what crosses into the `"use client"` subtree, and how to keep the client surface minimal so we don't drag the whole page into bundle-shipped JavaScript. The D9 server/client split was conceptual; today it becomes a concrete file structure.

## The Rule You Have To Internalize First

Server components don't have `useState`, `useEffect`, `useReducer`, event handlers (`onClick`, `onChange`, `onSubmit`), or browser APIs (`window`, `document`, `localStorage`). They run once on the server, render to HTML, and never run again. The moment you need any of those, you need a client component.

Conversely, server components can do things client components can't: `await` data at the top of the function body, read `cookies()` and `headers()` from `next/headers`, hit the database directly, use Node-only modules.

**The discipline:** push the `"use client"` boundary as far down the tree as you can, then stop. Everything above the boundary is server-side; everything below is bundled and shipped.

## What's Client vs Server On The Test-Taking Page

Walk through the page top-down and tag each piece:

| Concern | Server or Client? | Why |
|---|---|---|
| Read route param `testId` | Server | `params` is an async server-only API |
| Read auth cookie | Server | `cookies()` is server-only; keeps the cookie off the client bundle |
| `POST /sessions` to mint session | Server | One-shot, on the critical path, server fetch (Topic 2) |
| Render outer page chrome (header, container) | Server | Static, no interactivity |
| `currentIndex` state ("which question am I on") | Client | Changes on user action (Topic 8) |
| `answers` Map ("which option(s) did I pick") | Client | Changes on user action; persists across nav |
| Radio / checkbox `onChange` handlers | Client | Event handlers |
| "Prev" / "Next" buttons | Client | `onClick` |
| Question stem and option labels (read-only text) | Either, **rendered by client parent** | They're props of a client component, so they ship in the client bundle regardless |
| Server-anchored timer (D14) | Client | Tomorrow's concern |

The page splits naturally into a server root (`page.tsx`) and one client island (`TestRunner.tsx`). Inside `TestRunner`, *everything* is client — there is no toggling back to server-rendered inside a client subtree.

## File Layout

```
frontend/app/take/[testId]/
├── page.tsx              ← server component, default export
├── TestRunner.tsx        ← client component, "use client"
├── QuestionView.tsx      ← client (renders inside TestRunner)
├── SingleSelectQuestion.tsx  ← client (Topic 7)
├── MultiSelectQuestion.tsx   ← client (Topic 7)
└── Navigation.tsx        ← client (prev/next buttons)
```

Only `page.tsx` is a server component. The four others all carry `"use client"`. You don't need to repeat the directive on each file inside the client subtree — `"use client"` marks an **entry point**, and everything imported transitively is bundled as client too. But there's no harm in being explicit on each, and tools like ESLint plugins reward consistency.

## The `"use client"` Directive

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";

import { useState } from "react";
import type { Session } from "@/lib/api/types";
import { QuestionView } from "./QuestionView";
import { Navigation } from "./Navigation";

type Props = { session: Session };

export function TestRunner({ session }: Props) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const question = session.questions[currentIndex];

  return (
    <div className="space-y-6">
      <QuestionView question={question} />
      <Navigation
        currentIndex={currentIndex}
        total={session.questions.length}
        onPrev={() => setCurrentIndex((i) => Math.max(0, i - 1))}
        onNext={() => setCurrentIndex((i) => Math.min(session.questions.length - 1, i + 1))}
      />
    </div>
  );
}
```

`"use client"` must be the very first line (after optional shebang/encoding lines, but in practice: line 1). It applies to the whole file, not to individual exports. There's no `"use server"` equivalent for marking a single component as client.

## The Boundary Goes One Way

```
page.tsx (server)
   │
   │  props: { session }    ← serializable data crosses
   ▼
TestRunner.tsx (client)
   │
   ▼
QuestionView (client)
   │
   ▼
SingleSelectQuestion (client)  /  MultiSelectQuestion (client)
   │
   ▼
Navigation (client)
```

Once you cross into `"use client"`, you can't import a server component and render it inline. This compiles to an error:

```tsx
// TestRunner.tsx
"use client";
import SomeServerComponent from "./ServerThing"; // ← error if ServerThing is a server component
```

There is one escape hatch: a client component can render server components passed in via `children` (or any prop that is React node). The server-side parent does the rendering; the client component is just a wrapper. We don't need that today, but you'll see the pattern in shadcn-heavy layouts.

```tsx
// Pattern (not used today): server children rendered inside a client wrapper.
<ClientCard>
  <ServerOnlyThing /> {/* rendered by server parent, slot-rendered by client */}
</ClientCard>
```

## Why Bundle Size Matters Here

Every line of code inside the client subtree gets serialized into a JavaScript bundle the browser downloads and parses. Drag the wrong import in and your test-taking page balloons:

```tsx
// QuestionView.tsx — BAD
"use client";
import { generateScoreReport } from "@/lib/scoring/full"; // 200KB of server-only logic
```

`generateScoreReport` is server-only — it's D12's scoring engine. Importing it in a client component pulls the entire module graph (Pydantic-style schemas, database adapters, the works) into the browser bundle. The build won't necessarily fail; the import might "succeed" via tree-shaking or polyfill, but bundle size explodes silently.

**The defense:** organize `lib/` so server-only code lives in `lib/server/` and is gated by a runtime check or naming convention. Tools like `server-only` (an npm package; `import "server-only"` at the top of a module makes the build fail if it ends up client-bundled) make this enforceable. Adopt it for any module that talks to the database or reads secrets.

## When To Resist Adding A Client Component

A common mistake is reflexively reaching for `"use client"` because *something* on the page is interactive. The narrower the boundary, the smaller the bundle.

Bad — the whole page is client because of one button:

```tsx
// page.tsx — BAD: pulls everything into the client bundle
"use client";
export default function TakeTestPage(...) {
  const [open, setOpen] = useState(false);
  return (
    <div>
      <SessionHeader session={session} />     {/* now client too */}
      <QuestionList questions={questions} />  {/* now client too */}
      <button onClick={() => setOpen(true)}>Help</button>
    </div>
  );
}
```

Better — isolate the interactive bit:

```tsx
// page.tsx — server
export default async function TakeTestPage(...) {
  return (
    <div>
      <SessionHeader session={session} />    {/* server */}
      <QuestionList questions={questions} /> {/* server */}
      <HelpButton />                          {/* client island */}
    </div>
  );
}

// HelpButton.tsx — client
"use client";
export function HelpButton() {
  const [open, setOpen] = useState(false);
  return <button onClick={() => setOpen(true)}>Help</button>;
}
```

For the test-taking page, the answer differs from this example because `<TestRunner>` itself owns enough interactive state (question selection, navigation, eventually the timer) that *most* of the content is genuinely client. But the page chrome — header, footer, score-link breadcrumb — stays server.

## Identifying Which Parts Must Be Client

A reliable checklist when designing a page:

1. Does this code call any of `useState`, `useReducer`, `useEffect`, `useRef`, `useMemo`, `useCallback`, `useContext`? → Client.
2. Does it attach event handlers (`onClick`, `onChange`, etc.)? → Client.
3. Does it touch `window`, `document`, `localStorage`, `navigator`, etc.? → Client.
4. Does it use a hook from a third-party library that wraps any of the above (React Query, Zustand, etc.)? → Client.
5. Otherwise → Server. Default to server.

For Day 13's deliverable: server-component `page.tsx` mints the session, hands it to a client-component `TestRunner` that owns the index + answers state and renders the question.

## Common Mistakes

- **`"use client"` not on line 1.** The directive must precede all imports. A blank line above it is fine; a comment above it usually is too, but in older Next.js versions even that broke. Keep it on line 1.
- **Hoisting state to the server because "it's neater".** Server components can't hold state across renders. Each server render is a fresh function call.
- **Importing server-only modules into client components.** Use `import "server-only"` as a guardrail.
- **Re-marking every nested component `"use client"`**. Harmless but noisy. The directive at the entry point is enough.
- **Passing functions or class instances across the boundary.** The serialization will throw. Pass plain data; build the function inside the client component using the data.

## Key Takeaways
- Server components render once on the server with no client JS shipped; client components ship to the browser and own all interactivity.
- The test-taking page is a server-component `page.tsx` plus a single client island, `<TestRunner>`, that owns navigation and answer state.
- `"use client"` marks an *entry point* to the client graph; everything transitively imported becomes client-bundled.
- Push the boundary as deep as possible — keep page chrome server-rendered, isolate interactivity into named client components.
- Use the `server-only` npm helper (or strict folder conventions) to prevent server-only modules from leaking into client bundles.

---
*Prerequisites: day-9-server-vs-client-components, day-13-server-components-for-initial-data-fetching. Forward references: day-13-multi-step-ui-navigation-without-state-loss, day-13-polymorphic-component-rendering-for-variant-data-types, day-14-component-state-management-for-the-test-taking-experience.*
