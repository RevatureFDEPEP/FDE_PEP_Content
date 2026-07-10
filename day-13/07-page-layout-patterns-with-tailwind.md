# Page Layout Patterns With Tailwind

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The test-taking page has roughly four regions: a header strip (test title, candidate identity, eventually a server-anchored timer), a question stem with optional metadata, an answer area whose shape depends on the question type (Topic 7), and a navigation bar at the bottom (prev/next/submit). It should be readable on a 13" laptop, not get lost on a 27" display, and degrade gracefully on a tablet if a candidate insists on using one. This file covers composing that layout with Tailwind utility classes and shadcn/ui primitives — the conventions the inherited frontend already uses — so the page fits the existing visual language rather than feeling bolted on.

## What's Already In The Repo

The brownfield frontend ships with:

- **Tailwind CSS v4**, configured via `app/globals.css` with the `@import "tailwindcss"` directive and a small set of design tokens (`--background`, `--foreground`, `--primary`, `--muted`, etc.) defined under `@theme inline`.
- **shadcn/ui** primitives in `components/ui/` — Button, Card, RadioGroup, Checkbox, Label, Separator, etc. These are *copy-paste* components owned by the repo, not a versioned dependency. Edit freely.
- **A `cn()` utility** in `lib/utils.ts` that merges Tailwind classes with `clsx` + `tailwind-merge` semantics. Use it whenever a component takes a `className` prop.

Don't add new component libraries today; compose with what's there.

## The Page Skeleton

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";

import { useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Separator } from "@/components/ui/separator";
import { QuestionView } from "./QuestionView";
import type { Session } from "@/lib/api/types";

type Props = { session: Session };

export function TestRunner({ session }: Props) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const total = session.questions.length;
  const question = session.questions[currentIndex];

  return (
    <main className="mx-auto flex min-h-[calc(100vh-4rem)] w-full max-w-3xl flex-col gap-6 px-4 py-6 md:px-6">
      {/* Header strip */}
      <header className="flex items-baseline justify-between">
        <h1 className="text-2xl font-semibold tracking-tight">Test in progress</h1>
        <span className="text-sm text-muted-foreground">
          Question {currentIndex + 1} of {total}
        </span>
        {/* D14: timer slot goes here */}
      </header>

      <Separator />

      {/* Question card */}
      <Card className="flex-1">
        <CardHeader>
          <CardTitle className="text-lg font-medium leading-relaxed">
            {question.stem}
          </CardTitle>
        </CardHeader>
        <CardContent>
          <QuestionView question={question} />
        </CardContent>
      </Card>

      {/* Navigation footer */}
      <footer className="flex items-center justify-between gap-2">
        <Button
          variant="outline"
          onClick={() => setCurrentIndex((i) => Math.max(0, i - 1))}
          disabled={currentIndex === 0}
        >
          Previous
        </Button>
        <Button
          onClick={() => setCurrentIndex((i) => Math.min(total - 1, i + 1))}
          disabled={currentIndex === total - 1}
        >
          Next
        </Button>
      </footer>
    </main>
  );
}
```

The layout choices and what each is doing:

| Utility | Purpose |
|---|---|
| `mx-auto max-w-3xl` | Center the column, cap at ~48rem so line lengths stay readable on wide screens |
| `min-h-[calc(100vh-4rem)]` | Fill the viewport minus the global header (4rem assumed) so the navigation footer sits at the bottom |
| `flex flex-col gap-6` | Vertical stack with consistent spacing — easier to maintain than per-element `mb-*` |
| `px-4 py-6 md:px-6` | Tighter padding on mobile, breathe on tablet+ |
| `flex-1` on the card | Card stretches to fill available space, pushing the footer down |
| `items-baseline` on header | Align "Test in progress" baseline with "Question X of Y" — typographic polish |
| `text-muted-foreground` | Token-driven color from globals.css; works in dark mode automatically |

## The Layout Pattern: Three-Region Vertical Stack

The page is, structurally, three regions stacked vertically inside a centered column:

```
┌────────────────────────────────────────┐
│  HEADER  (test title, progress, timer) │
├────────────────────────────────────────┤
│                                        │
│  QUESTION CARD                         │
│  (stem + answer area)                  │
│  flex-1 — grows to fill                │
│                                        │
├────────────────────────────────────────┤
│  NAVIGATION  (prev / next / submit)    │
└────────────────────────────────────────┘
```

This is the simplest layout that meets the requirements and the one to default to. Resist the temptation to make it a grid; flex-col with `flex-1` on the middle child is exactly right.

For the timer slot (D14), it slides into the header on the right:

```tsx
<header className="flex items-baseline justify-between">
  <div>
    <h1 className="text-2xl font-semibold">Test in progress</h1>
    <p className="text-sm text-muted-foreground">Question {currentIndex + 1} of {total}</p>
  </div>
  <Timer expiresAt={session.expires_at} serverNow={session.server_now} />
</header>
```

Reserve the visual space today so adding the timer tomorrow doesn't reflow the page.

## Why The Card Wraps The Question

Two reasons to use the `<Card>` primitive around the question stem and answer area:

1. **Visual grouping.** The card creates a clear container, so the question "feels" like a discrete unit and not a slab of text on the page background.
2. **Consistency with the rest of the app.** The inherited dashboard and admin pages use cards for grouped content. Matching the existing pattern means the test-taking page doesn't look like a different app.

If a stylistic outlier is desired (e.g., a "minimalist mode" with no card chrome), gate it behind a future setting rather than baking it in.

## Spacing Discipline

Tailwind makes it tempting to sprinkle margins everywhere. Resist. The rule:

- **Parents own spacing between siblings.** Use `gap-*` on the flex/grid parent. Children don't carry `mb-*` "just in case".
- **Children own internal padding.** Inside a `<Card>`, the `CardHeader` and `CardContent` provide their own padding; don't add `p-*` from the outside.
- **Consistent scale.** Stick to the Tailwind default scale (`gap-2`, `gap-4`, `gap-6`, `gap-8`). Avoid `gap-[7px]` or other arbitrary values without a reason.

Applied to our page: `gap-6` between header, card, and footer; `gap-2` between footer buttons; `space-y-3` inside `QuestionView` between the stem and the options block. No `mb-*` anywhere.

## Responsive Breakpoints

The default Tailwind breakpoints (`sm`, `md`, `lg`, `xl`, `2xl`) are designed mobile-first: base classes apply at all sizes, prefixed classes apply at and above the breakpoint. For the test-taking page, the only adjustments needed are padding:

```tsx
className="mx-auto w-full max-w-3xl px-4 py-6 md:px-6 lg:py-8"
```

- Below `md` (768px): `px-4 py-6` — tight padding for narrow viewports.
- `md` and up: `px-6` — more breathing room.
- `lg` and up: `py-8` — extra vertical space on desktop.

A 25-person cohort will be 95% on laptops. Don't over-engineer for mobile; do the table-stakes responsive padding and move on.

## Dark Mode

Tailwind v4 + the inherited shadcn theme already supports dark mode via the `dark:` variant and CSS variables. As long as you use the token classes (`bg-background`, `text-foreground`, `text-muted-foreground`, `border-border`, `bg-primary`, etc.) instead of literal colors (`bg-white`, `text-gray-700`), dark mode "just works" when the system preference flips. The page above uses `text-muted-foreground` and lets the `<Card>` primitive handle its own background — no special-casing needed.

The anti-pattern is reaching for literal grays:

```tsx
// BAD: doesn't respond to dark mode
<p className="text-gray-500">Question 1 of 10</p>

// GOOD: token-driven, dark-mode-aware
<p className="text-muted-foreground">Question 1 of 10</p>
```

## Using `cn()` For Conditional Classes

When you need to conditionally apply styles — disabled states, error states, selected states — use the `cn()` helper:

```tsx
import { cn } from "@/lib/utils";

<button
  className={cn(
    "rounded-md border px-4 py-2 text-sm transition-colors",
    "hover:bg-accent hover:text-accent-foreground",
    isSelected && "border-primary bg-primary/10 text-primary",
    isDisabled && "cursor-not-allowed opacity-50",
  )}
/>
```

`cn()` strips falsy values, merges duplicates intelligently (`bg-red-500 bg-blue-500` → `bg-blue-500`), and avoids the string-concat hell of `${base} ${isSelected ? "..." : ""}`. Use it on any component that takes a `className` prop or has conditional styling.

For Topic 7 (polymorphic question rendering), `cn()` is what makes the SingleSelect and MultiSelect option states clean.

## Common Mistakes

- **Putting the page in a `<div>` instead of `<main>`.** Semantic landmarks help screen readers and document outline. `<main>`, `<header>`, `<footer>`, `<nav>` mean what they say.
- **Hard-coded heights.** `h-96` on the question card sometimes overflows, sometimes leaves gaps. Use `flex-1` and let the parent control the height.
- **Inline `style={{ ... }}` for one-off colors.** Either add a token or use a Tailwind utility. Inline styles bypass the design system.
- **Mixing `gap-*` on a parent with `mb-*` on children.** Pick one. Parents-own-spacing is the cleaner rule.
- **Using `text-gray-*` literals.** Breaks dark mode, breaks rebranding, drifts from the rest of the app. Use the token classes.
- **Forgetting `mx-auto` on the centered column.** Without `mx-auto` (or `m-auto`), `max-w-3xl` left-aligns the column on wide screens.

## Key Takeaways
- The page is a three-region vertical flex stack: header, question card (`flex-1`), navigation footer — centered in a `max-w-3xl` column.
- Use shadcn primitives (`Card`, `Button`, `Separator`) for visual consistency with the inherited app; compose with Tailwind utility classes, not custom CSS.
- Parents own spacing (`gap-*`), children own internal padding — pick one and stick to it.
- Use token classes (`text-muted-foreground`, `bg-background`) instead of literal colors so dark mode works for free.
- Use `cn()` from `lib/utils` whenever classes are conditional or a component takes a `className` prop.
- Reserve visual space for tomorrow's timer slot in the header so adding it doesn't reflow the page.

---
*Prerequisites: [05-component-composition-with-tailwind-and-shadcn-ui.md](../day-09/05-component-composition-with-tailwind-and-shadcn-ui.md), [05-component-composition-with-tailwind-and-shadcn-ui.md](../day-09/05-component-composition-with-tailwind-and-shadcn-ui.md). Forward references: [08-polymorphic-component-rendering-for-variant-data-types.md](08-polymorphic-component-rendering-for-variant-data-types.md), [04-client-side-temporal-state-timers-and-autosave.md](../day-14/04-client-side-temporal-state-timers-and-autosave.md).*
