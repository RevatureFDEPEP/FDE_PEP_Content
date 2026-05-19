# Form State Collection With react-hook-form

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D9 used react-hook-form (RHF) to collect a single question authoring form — one component, one `useForm`, one submit. The test-taking page is different: the candidate answers many questions across many navigations, but logically it's *one* submission with *one* validation pass at the end. This topic re-applies RHF in that multi-step posture. The Topic 1 reducer holds the navigation index and autosave status; RHF holds the actual answer values in its own internal field store and exposes them through `Controller` and `getValues`. The two state systems coexist because they answer different questions — the reducer answers "what is the UI showing right now," and RHF answers "what does the candidate intend to submit."

## Why Use RHF At All When We Have A Reducer

A reasonable objection: the reducer already tracks an `answers` Map. Why introduce a second state owner?

Three reasons:

1. **RHF integrates with zod (Topic 3) for free.** A schema-validated form gives us a single validation point on submit; rolling that on top of a reducer means re-implementing field-level error tracking by hand.
2. **`Controller` cleanly bridges custom UI inputs (radio groups, checkbox arrays) to a form's field store.** D13's polymorphic `<SingleSelectQuestion>` and `<MultiSelectQuestion>` become RHF-managed without changing their public API.
3. **`getValues()`, `formState.isDirty`, and `formState.dirtyFields` are useful for autosave (Topic 4).** Only dirty questions get re-sent.

The reducer remains the source of truth for *UI ephemera* — which question is currently visible, autosave status per question, submit lifecycle. RHF owns *what the candidate has actually answered*. The two stay synchronized via `Controller`.

## Schema First (D9 Pattern, Multi-Question Variant)

D9 had one form, one schema. Here the schema describes the *entire submission* — every question's answer, keyed by ID.

```ts
// frontend/app/take/[testId]/schema.ts
import { z } from "zod";
import type { SessionQuestion } from "@/lib/api/types";

/** Per-question answer shape: an array of selected option IDs. Empty array = unanswered. */
const answerSchema = z.array(z.number().int().nonnegative());

/** Build a schema dynamically from the session's questions so the field names line up. */
export function buildSessionFormSchema(questions: SessionQuestion[]) {
  const shape: Record<string, typeof answerSchema> = {};
  for (const q of questions) {
    shape[q.question_id] = answerSchema;
  }
  return z.object(shape);
}

export type SessionFormValues = Record<string, number[]>;
```

The schema is *built* from the session because the question IDs aren't known at compile time. This is one of the few cases where a dynamically constructed zod schema is genuinely the right tool — the shape comes from server data.

Default values must also be derived from the session: one empty array per question, so that `Controller` for an unanswered question still has a defined value (preventing the dreaded "changing an uncontrolled input to be controlled" warning).

```ts
export function defaultValuesFor(questions: SessionQuestion[]): SessionFormValues {
  return Object.fromEntries(questions.map((q) => [q.question_id, []]));
}
```

## Wiring `useForm` In `<TestRunner>`

```tsx
// frontend/app/take/[testId]/TestRunner.tsx
"use client";

import { useForm, FormProvider } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { useMemo } from "react";
import { buildSessionFormSchema, defaultValuesFor, type SessionFormValues } from "./schema";

export function TestRunner({ session }: Props) {
  const schema = useMemo(() => buildSessionFormSchema(session.questions), [session.questions]);
  const defaultValues = useMemo(() => defaultValuesFor(session.questions), [session.questions]);

  const methods = useForm<SessionFormValues>({
    resolver: zodResolver(schema),
    defaultValues,
    mode: "onSubmit",       // validate on submit only — instant feedback comes from Topic 3 manually
    shouldUnregister: false, // KEEP unfocused question values around when we navigate away
  });

  // ...reducer wiring from Topic 1...

  return (
    <FormProvider {...methods}>
      <QuestionView ... />
      <Navigation ... />
      <SubmitButton ... />
    </FormProvider>
  );
}
```

Two RHF options here are non-default and critical:

- **`shouldUnregister: false`.** RHF's default is to *unregister* a field's value when its `Controller` unmounts. On a paged test-taking flow, navigating from Q3 to Q4 unmounts the Q3 `Controller` — which under the default behavior would wipe Q3's answer. Setting `shouldUnregister: false` keeps every field's value in the form store regardless of mount state. This is the "single logical form across many UI steps" knob.
- **`mode: "onSubmit"`.** We don't want red-bordered required errors flashing on a question the candidate hasn't even reached. We validate the whole form on submit, surface a count of unanswered questions then, and use zod parsing directly elsewhere when we want instant per-field feedback.

`FormProvider` lets nested components reach `useFormContext()` without prop-drilling `methods`. With many questions across many files (`<QuestionView>`, `<SingleSelectQuestion>`, etc.), that matters.

## `Controller` Around The Polymorphic Question Components

The D13 polymorphic `<QuestionView>` takes `selected: number[]` and `onChange: (selected: number[]) => void`. `Controller` adapts that signature to RHF's field interface:

```tsx
// frontend/app/take/[testId]/QuestionView.tsx
"use client";

import { Controller, useFormContext } from "react-hook-form";
import type { SessionQuestion } from "@/lib/api/types";
import { SingleSelectQuestion } from "./SingleSelectQuestion";
import { MultiSelectQuestion } from "./MultiSelectQuestion";

type Props = { question: SessionQuestion; disabled: boolean };

export function QuestionView({ question, disabled }: Props) {
  const { control } = useFormContext();

  return (
    <Controller
      control={control}
      name={question.question_id}
      render={({ field }) => {
        const common = {
          question,
          selected: field.value ?? [],
          onChange: field.onChange,
          disabled,
        };
        switch (question.type) {
          case "single_select":
            return <SingleSelectQuestion {...common} />;
          case "multi_select":
            return <MultiSelectQuestion {...common} />;
          default: {
            const _exhaustive: never = question;
            return null;
          }
        }
      }}
    />
  );
}
```

The polymorphic dispatch from D13 is preserved exactly. The `Controller` is responsible for piping the field's current value into the question component and routing the question component's `onChange` callback back into the RHF field store. Neither `SingleSelectQuestion` nor `MultiSelectQuestion` know RHF exists.

## Reading Across Questions For Submit

The submit handler reads *all* values, validates via the schema (zodResolver runs automatically), and POSTs the result. The shape RHF gives back matches the schema, so the API client takes it directly.

```tsx
const onSubmit = methods.handleSubmit(async (values) => {
  // `values` is SessionFormValues: { [question_id]: number[] }
  // Already zod-validated.
  await api.submitSession(session.session_id, { answers: values });
});

// In JSX
<form onSubmit={onSubmit}>...</form>
```

`handleSubmit` wraps the inner function: it runs the resolver, only calls the inner function if validation passes, and on failure populates `formState.errors` for display.

## How RHF And The Reducer Cooperate For Autosave (Preview Of Topic 4)

Topic 4 will debounce per-question saves. RHF makes that easy: `methods.watch(question.question_id)` returns the current field value reactively, and `methods.formState.dirtyFields[question.question_id]` tells us whether the candidate has touched it since the form was initialized.

```tsx
// Inside <TestRunner>, sketch only — full implementation in Topic 4.
const currentAnswer = methods.watch(question.question_id);
useDebouncedEffect(() => {
  if (methods.formState.dirtyFields[question.question_id]) {
    autosaveAnswer(question.question_id, currentAnswer);
  }
}, [currentAnswer], 400);
```

The reducer doesn't have to redundantly track answer values; it tracks *autosave status*. RHF tracks values. Each does one job.

## Navigation Doesn't Mount/Unmount The Form

The form lives in `<TestRunner>` (the root client component). Navigating between questions remounts only the `<QuestionView>` subtree — the form itself stays alive. Combined with `shouldUnregister: false`, this means every answer the candidate has entered is preserved even though only one question is visible at a time. This is the "without state loss" guarantee from D13 made concrete at the form layer.

## Common Mistakes

- **Forgetting `shouldUnregister: false`.** Without it, navigating away from a question wipes its answer from the form store. The reducer-tracked `answers` Map would still have the value (if you also wrote it there) — but on submit, RHF's `getValues()` would return only the currently-mounted question's value. Painful bug to track down.
- **Building the schema inline in the render body.** `buildSessionFormSchema(session.questions)` allocates a new schema object on every render and re-creates the resolver, which thrashes RHF's internal caching. Wrap in `useMemo`.
- **Skipping default values.** Without them, the first time a candidate touches a question, RHF emits a "changing from uncontrolled to controlled" warning, and field-level dirty tracking becomes inconsistent.
- **Trying to make the reducer the sole answer-state owner.** Doable, but then you lose `zodResolver` integration and have to hand-roll field-level error tracking and dirty detection. RHF earns its place by being good at exactly that.
- **Using `mode: "onChange"`** on this page. With 30+ fields it turns every keystroke into a full-form re-validation pass. Validate on submit.

## Key Takeaways
- The test-taking page is one logical form spanning many UI steps; RHF with `shouldUnregister: false` is the right primitive for that posture.
- Build the schema and default values dynamically from the session's questions — the field names are server data, not compile-time constants.
- `Controller` adapts the D13 polymorphic question components to RHF's field interface without modifying them.
- The reducer owns UI ephemera (current index, autosave status, submit lifecycle); RHF owns answer values. They cooperate, they don't compete.
- `FormProvider` + `useFormContext` keeps nested components from prop-drilling `methods` through three layers.

---
*Prerequisites: day-9-form-state-management-with-react-hook-form, day-13-polymorphic-component-rendering-for-variant-data-types, day-14-react-state-management-patterns-usestate-vs-usereducer. Forward references: day-14-client-side-validation-with-zod, day-14-client-side-temporal-state-timers-and-autosave, day-14-submit-and-lock-ux-patterns.*
