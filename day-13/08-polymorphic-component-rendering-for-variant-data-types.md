# Polymorphic Component Rendering For Variant Data Types

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
A `Question` from the D11 contract is either a single-select (one correct answer, render a radio group) or a multi-select (one or more correct answers, render checkboxes). Two shapes that ought to share the stem-and-options layout but diverge on the answer control. The wrong move is a single sprawling component with `if (type === "single") {...} else {...}` interleaved through it; the right move is **discriminated rendering** — switch on the `type` discriminator at one site and dispatch to two focused child components. This file covers the discriminated-union TypeScript pattern that came from Topic 4, the `switch` dispatch idiom, and the two leaf components for the answer controls.

## The Discriminator From The Backend

D8 established the Pydantic discriminator (`question_type: Literal["single_select", "multi_select"]`); Topic 4 mirrors it as a TypeScript discriminated union:

```ts
// frontend/lib/api/types.ts
export type Option = { id: number; label: string };

export type SingleSelectQuestion = {
  type: "single_select";
  question_id: string;
  stem: string;
  options: Option[];
};

export type MultiSelectQuestion = {
  type: "multi_select";
  question_id: string;
  stem: string;
  options: Option[];
};

export type Question = SingleSelectQuestion | MultiSelectQuestion;
```

The literal `type` field is the **discriminator**. Once TypeScript sees a check on it inside a control-flow construct, the type narrows automatically inside each branch. This is the whole reason the wire format includes `type` at all.

## The Dispatch Site

`QuestionView` is the polymorphic component. Its only job is to switch on `type` and render the right child. Everything else — the stem, the surrounding card chrome — lives in the parent (`TestRunner`, see Topic 6).

```tsx
// frontend/app/take/[testId]/QuestionView.tsx
"use client";

import type { Question } from "@/lib/api/types";
import { SingleSelectQuestion } from "./SingleSelectQuestion";
import { MultiSelectQuestion } from "./MultiSelectQuestion";

type Props = {
  question: Question;
  selected: number[];
  onChange: (selected: number[]) => void;
};

export function QuestionView({ question, selected, onChange }: Props) {
  switch (question.type) {
    case "single_select":
      return (
        <SingleSelectQuestion
          question={question}      // narrowed to SingleSelectQuestion
          selected={selected[0] ?? null}
          onChange={(id) => onChange(id === null ? [] : [id])}
        />
      );
    case "multi_select":
      return (
        <MultiSelectQuestion
          question={question}      // narrowed to MultiSelectQuestion
          selected={selected}
          onChange={onChange}
        />
      );
    default:
      // Exhaustiveness check — see below.
      return assertNever(question);
  }
}

function assertNever(x: never): never {
  throw new Error(`Unhandled question type: ${JSON.stringify(x)}`);
}
```

Inside `case "single_select":`, TypeScript narrows `question` from `Question` to `SingleSelectQuestion`. Inside `case "multi_select":`, it narrows to `MultiSelectQuestion`. The child components can be typed with the narrow type and TypeScript will enforce that the dispatch is correct.

The two child components own different interaction models:

- **Single-select:** at most one option selected. The prop shape is `selected: number | null`.
- **Multi-select:** zero or more selected. The prop shape is `selected: number[]`.

`QuestionView` translates between the parent's uniform `number[]` representation (Topic 8 holds *all* answers as `number[]` to keep the state structure consistent) and each child's natural shape.

## The Exhaustiveness Check

`assertNever(x: never)` is the **exhaustiveness pattern**. If a third question type — say `"free_text"` — gets added to the discriminator and the `switch` doesn't handle it, `question` in the `default` branch will no longer be of type `never`, and `assertNever(question)` becomes a compile error:

```
Argument of type 'FreeTextQuestion' is not assignable to parameter of type 'never'.
```

That's the *whole point*: forgetting to extend the dispatch when the backend adds a new question type is caught at compile time, not at runtime. Don't write the `default` as a fallback render — write it as an exhaustiveness gate.

## The Single-Select Leaf

```tsx
// frontend/app/take/[testId]/SingleSelectQuestion.tsx
"use client";

import { RadioGroup, RadioGroupItem } from "@/components/ui/radio-group";
import { Label } from "@/components/ui/label";
import type { SingleSelectQuestion as SingleSelectQuestionType } from "@/lib/api/types";

type Props = {
  question: SingleSelectQuestionType;
  selected: number | null;
  onChange: (id: number | null) => void;
};

export function SingleSelectQuestion({ question, selected, onChange }: Props) {
  return (
    <RadioGroup
      value={selected === null ? "" : String(selected)}
      onValueChange={(val) => onChange(val === "" ? null : Number(val))}
      className="space-y-3"
    >
      {question.options.map((option) => {
        const id = `q-${question.question_id}-opt-${option.id}`;
        return (
          <div key={option.id} className="flex items-start gap-3">
            <RadioGroupItem value={String(option.id)} id={id} className="mt-1" />
            <Label htmlFor={id} className="cursor-pointer text-base font-normal leading-snug">
              {option.label}
            </Label>
          </div>
        );
      })}
    </RadioGroup>
  );
}
```

Two practical notes:

1. **shadcn's `RadioGroup` takes string values.** Convert to/from number at the boundary. The option IDs from the backend are numeric.
2. **The `Label htmlFor` connection** means clicking the label text selects the radio — important for usability and accessibility. Don't skip it.

## The Multi-Select Leaf

```tsx
// frontend/app/take/[testId]/MultiSelectQuestion.tsx
"use client";

import { Checkbox } from "@/components/ui/checkbox";
import { Label } from "@/components/ui/label";
import type { MultiSelectQuestion as MultiSelectQuestionType } from "@/lib/api/types";

type Props = {
  question: MultiSelectQuestionType;
  selected: number[];
  onChange: (selected: number[]) => void;
};

export function MultiSelectQuestion({ question, selected, onChange }: Props) {
  const toggle = (optionId: number, checked: boolean) => {
    if (checked) {
      onChange([...selected, optionId].sort((a, b) => a - b));
    } else {
      onChange(selected.filter((id) => id !== optionId));
    }
  };

  return (
    <div className="space-y-3">
      <p className="text-sm text-muted-foreground">Select all that apply.</p>
      {question.options.map((option) => {
        const id = `q-${question.question_id}-opt-${option.id}`;
        const isChecked = selected.includes(option.id);
        return (
          <div key={option.id} className="flex items-start gap-3">
            <Checkbox
              id={id}
              checked={isChecked}
              onCheckedChange={(checked) => toggle(option.id, checked === true)}
              className="mt-1"
            />
            <Label htmlFor={id} className="cursor-pointer text-base font-normal leading-snug">
              {option.label}
            </Label>
          </div>
        );
      })}
    </div>
  );
}
```

The multi-select adds two affordances the single-select doesn't:

- **"Select all that apply."** Explicit instruction. Without it, candidates default-assume single-select and miss the multi-select semantics.
- **Sorted selection.** `[...selected, optionId].sort()` keeps the array canonical. The D12 scoring algorithm uses set semantics so order doesn't matter, but a canonical order makes idempotency-key hashing stable and debugging output predictable.

`onCheckedChange` from shadcn's Checkbox passes a `CheckedState` (which can be `"indeterminate"`); we narrow to `checked === true` because we're not using the tri-state.

## Why Two Components Instead Of One With Conditional Rendering

A perfectly-functional alternative is one component that handles both:

```tsx
// One component, branching internally — works but worse.
export function QuestionView({ question, selected, onChange }) {
  const Control = question.type === "single_select" ? RadioGroup : "div";
  // ...100 lines of `if (question.type === "single_select")` interspersed
}
```

Three reasons the two-component split wins:

1. **Each component has one concern.** SingleSelect doesn't know multi-select exists, and vice versa. Adding a third question type means a third sibling, not surgery in the joint file.
2. **Props are narrower and easier to type.** SingleSelect's `selected` is `number | null`. Multi's is `number[]`. The union of those is awkward; the split keeps each prop type honest.
3. **Test surface is smaller.** Vitest tests (D14) for the single-select component don't need to set up multi-select fixtures and vice versa.

The cost is a slightly larger file count. Trade you make happily.

## When Polymorphic Rendering Doesn't Apply

The pattern fits when:

- There's a *small, closed* set of variants (2–5).
- Variants differ in *interaction model*, not just in styling.
- A discriminator field is present on the wire.

It doesn't fit when:

- Variants differ only in cosmetic ways (e.g., a "color" field). Use a prop, not a polymorphic component.
- The set is open-ended (plugin systems, user-generated types). Reach for a registry pattern.
- The variants share *no* structure. Two unrelated UIs that happen to live on the same page should just be two separate components rendered at separate sites.

For D13 the conditions are met cleanly: closed set, different interaction model, discriminator present.

## Adding A Third Question Type Later

Hypothetical: D20 introduces `"free_text"`. The change is mechanical:

1. **Add the type** to `Question` union in `types.ts` (or regenerate via openapi-typescript).
2. **Create `FreeTextQuestion.tsx`** alongside the existing two.
3. **Add a `case "free_text":` arm** to the `switch` in `QuestionView`.
4. **TypeScript will fail to compile until step 3 is done** because `assertNever(question)` no longer satisfies `question: never`.

The exhaustiveness check is what makes step 4 cheap. Without it, a missing case fails silently — the user sees a blank space where the free-text input should be, no error, no clue.

## Common Mistakes

- **No exhaustiveness check.** A `default: return null;` swallows missing cases. Use `assertNever`.
- **Discriminator as `string` instead of literal union.** TypeScript can't narrow on `type: string`; it can narrow on `type: "single_select" | "multi_select"`. The literal types are mandatory.
- **Passing the union type into the child component.** The child accepts the *narrow* type. Type checking inside the child becomes pointless if it has to re-narrow.
- **Branching with `if/else` instead of `switch`.** Works, but `switch` over the discriminator reads better and pairs naturally with the exhaustiveness check.
- **Sorting the multi-select selection inside the scoring code, not the UI.** Canonicalize at the source so all downstream consumers (idempotency-key hashing, debug logs) see consistent data.
- **Forgetting `htmlFor`/`id` on labels.** Clicking the text should select the option. It's a non-negotiable usability + a11y baseline.

## Key Takeaways
- Use the discriminated-union `type` field from the wire format as the switch key; TypeScript narrows the type automatically inside each `case`.
- One dispatch site (`QuestionView`) plus one focused leaf component per variant (`SingleSelectQuestion`, `MultiSelectQuestion`) beats a single sprawling conditional component.
- `assertNever(x: never)` in the `default` branch turns "forgot to handle a new variant" into a compile error.
- Translate at the dispatch boundary: the parent holds uniform `number[]` state, single-select children see `number | null`, multi-select see `number[]`.
- Pair every shadcn `RadioGroupItem`/`Checkbox` with a `<Label htmlFor>` so the label text is part of the click target.

---
*Prerequisites: [01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md](../day-08/01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md), [05-component-composition-with-tailwind-and-shadcn-ui.md](../day-09/05-component-composition-with-tailwind-and-shadcn-ui.md), [05-type-safe-api-integration.md](05-type-safe-api-integration.md). Forward references: [09-multi-step-ui-navigation-without-state-loss.md](09-multi-step-ui-navigation-without-state-loss.md), [06-submit-and-lock-ux-patterns.md](../day-14/06-submit-and-lock-ux-patterns.md).*
