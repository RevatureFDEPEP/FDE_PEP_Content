# Client-Side Validation With zod

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
D9 introduced zod as the client-side validator for question authoring. D12's backend wrote the authoritative scoring rules: a single-select answer must be exactly one option ID drawn from the question's option set; a multi-select can be zero or more option IDs drawn from the same set. The client cannot be trusted to enforce these rules — the backend re-validates everything — but the client *should* enforce them anyway, because the alternative is a candidate clicking "Submit," waiting for a network round trip, and seeing a generic 422 error pointing at question 17 of 30. Today's job: mirror the backend's per-question validation rules in zod, run them at the boundary (on selection change and on submit), and surface specific, helpful errors immediately.

## What The Backend Considers Valid (Per D12)

The authoritative rules from D12 scoring:

| Question type | Valid answer shape |
|---|---|
| `single_select` | Exactly one integer; must match an `option_id` from the question |
| `multi_select` | Zero or more distinct integers; each must match an `option_id` from the question |
| Both | Duplicate option IDs in the array are rejected |
| Both | An empty array for `single_select` is *also* rejected at submit; treated as "unanswered" client-side but blocked on submit |

The client must mirror these and add one of its own: **unanswered questions block submit**. The backend accepts an empty answers map (it just scores those questions 0); the UX requires the candidate to confirm "yes, I'm submitting with N unanswered." That's a client concern.

## Building The Per-Question Schema From The Session

The schema must reference the option IDs from the loaded session, so it's built dynamically (similar pattern to Topic 2's full-form schema builder).

```ts
// frontend/app/take/[testId]/answerValidation.ts
import { z } from "zod";
import type { SessionQuestion } from "@/lib/api/types";

/**
 * Per-question answer validator. The accepted shape depends on the question
 * type and the question's specific option ID set.
 */
export function answerSchemaFor(question: SessionQuestion) {
  const validOptionIds = new Set(question.options.map((o) => o.option_id));

  const baseArray = z
    .array(z.number().int().nonnegative())
    .refine((arr) => arr.every((id) => validOptionIds.has(id)), {
      message: "Selection contains an option ID not present in this question",
    })
    .refine((arr) => new Set(arr).size === arr.length, {
      message: "Duplicate option IDs are not allowed",
    });

  switch (question.type) {
    case "single_select":
      return baseArray.length(1, {
        message: "Single-select questions require exactly one answer",
      });

    case "multi_select":
      return baseArray; // zero or more — empty is "unanswered," handled separately

    default: {
      const _exhaustive: never = question;
      throw new Error("Unhandled question type");
    }
  }
}
```

This is the validator for a single question's answer, used in two places:

1. **On selection change** — to surface immediate feedback if the candidate somehow ends up with an invalid selection (rare in normal UI flow, but useful if state is restored from autosave).
2. **On submit** — to block submission if any answered question is invalid.

The `_exhaustive: never` mirrors the reducer pattern from Topic 1 — adding a third question type without updating this builder is a build error.

## Running Validation On Submit

The full-form schema from Topic 2 (`buildSessionFormSchema`) validates *shape*. To validate *content per question*, we layer the per-question schema inside it using `z.record` + `.superRefine`, or by composing the per-question schemas explicitly:

```ts
// frontend/app/take/[testId]/schema.ts (extended from Topic 2)
import { z } from "zod";
import { answerSchemaFor } from "./answerValidation";
import type { SessionQuestion } from "@/lib/api/types";

export function buildSessionFormSchema(questions: SessionQuestion[]) {
  const shape: Record<string, z.ZodType<number[]>> = {};
  for (const q of questions) {
    shape[q.question_id] = answerSchemaFor(q);
  }
  return z.object(shape);
}
```

Because RHF's `zodResolver` (Topic 2) uses this schema as its resolver, every `handleSubmit` automatically runs all per-question validators, collects errors keyed by question_id, and populates `formState.errors[question_id]` for display.

## Surfacing Errors Where The Candidate Sees Them

A submit-time error keyed by `question_id` is useless if the candidate is on a different question. The error UI needs to:

1. Surface a top-of-page banner: "3 questions have problems," with anchor links.
2. Decorate the question navigator (a strip of numbered dots) with red marks for invalid questions.
3. When the candidate navigates to a problem question, show the specific error inline.

```tsx
// frontend/app/take/[testId]/SubmitErrorBanner.tsx
"use client";

import { useFormContext } from "react-hook-form";
import type { SessionFormValues } from "./schema";

type Props = {
  questions: Array<{ question_id: string; index: number }>;
  onJumpTo: (index: number) => void;
};

export function SubmitErrorBanner({ questions, onJumpTo }: Props) {
  const { formState } = useFormContext<SessionFormValues>();
  const errorIds = Object.keys(formState.errors);
  if (errorIds.length === 0) return null;

  const erroredQuestions = questions.filter((q) => errorIds.includes(q.question_id));

  return (
    <div role="alert" className="rounded border border-red-300 bg-red-50 p-4 text-sm">
      <p className="font-semibold">
        {erroredQuestions.length} question{erroredQuestions.length === 1 ? "" : "s"} need attention before you can submit.
      </p>
      <ul className="mt-2 space-y-1">
        {erroredQuestions.map((q) => (
          <li key={q.question_id}>
            <button type="button" onClick={() => onJumpTo(q.index)} className="text-red-700 underline">
              Question {q.index + 1}
            </button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

`onJumpTo` dispatches `{ type: "NAVIGATE", index }` to the reducer from Topic 1. The candidate is one click from the broken question. This is the difference between "your submission failed somewhere" and "fix question 17."

## Unanswered-Question Confirmation On Submit

Unanswered questions aren't zod errors — empty array satisfies `multi_select`'s schema, and a single_select that the candidate never touched has empty array as its default. They need a separate pre-submit check:

```tsx
function countUnanswered(values: SessionFormValues): number {
  return Object.values(values).filter((selected) => selected.length === 0).length;
}

const onSubmit = methods.handleSubmit(async (values) => {
  const unanswered = countUnanswered(values);
  if (unanswered > 0) {
    const ok = window.confirm(
      `You have ${unanswered} unanswered question${unanswered === 1 ? "" : "s"}. ` +
      `Unanswered questions will be scored as zero. Submit anyway?`
    );
    if (!ok) return;
  }
  await api.submitSession(session.session_id, { answers: values });
});
```

For a real product you'd replace `window.confirm` with a styled modal. The point stands: this is a UX-layer check, not a schema-layer check, because the backend genuinely accepts unanswered questions.

## Why Mirror The Backend Rather Than Trust It

Both at once:

- **Mirror** because instant feedback is the user experience win. A single-select with two options checked should refuse the second click locally; waiting for a 422 on submit is bad.
- **Trust the backend** because client validation is theatre against any sophisticated actor. The server's zod-equivalent (Pydantic, in this codebase) is the real gate. The client's job is to make the *honest* candidate's experience pleasant.

When backend and client validation drift, the symptom is usually: backend rejects something the client thinks is fine, candidate sees an unexplained submission failure. The mitigation is to colocate the validation rules — same JSON Schema, same OpenAPI contract, generated types — so the client schema stays in sync. For PEP we settle for "manually keep them in sync and write tests for both," but the principle to internalize is *single source of truth for validation rules, applied at every boundary*.

## Common Mistakes

- **Hardcoding option ID ranges** like `z.number().min(0).max(4)`. Option IDs are arbitrary integers assigned by the question authoring service; range-checking them is wrong. Set-membership against the question's own `options` is the only correct check.
- **Forgetting the duplicate-detection refinement.** A `multi_select` with `[1, 1, 2]` is invalid per D12 scoring (would otherwise let candidates double-count a correct option). The `new Set(arr).size === arr.length` refinement catches it.
- **Conflating "unanswered" with "invalid."** Unanswered is a UX confirmation; invalid is a hard block. Treat them separately or candidates will hit an apparent dead end.
- **Validating on every change with `mode: "onChange"` in RHF.** With 30 questions and per-keystroke validation, the page noticeably stutters. On-submit validation, with the optional per-question check on selection change, is the right balance.
- **Letting client validation be the only validation.** Always re-validate on the backend. Always. This is non-negotiable security/integrity hygiene.

## Key Takeaways
- Mirror D12's backend scoring rules client-side: single_select requires exactly one valid option ID; multi_select requires zero or more distinct valid IDs.
- Build per-question schemas dynamically from each question's own option set; compose them into the full-form schema for `zodResolver`.
- Distinguish *invalid* (schema failure, hard block) from *unanswered* (UX confirmation modal, candidate can still submit) — these are different concerns.
- Submit errors surface with a banner + jump-to-question affordance, because errors keyed by `question_id` are useless if the candidate is on a different question.
- Client validation is for UX; the backend is the real gate. They must agree, but the backend has the final word.

---
*Prerequisites: [04-client-side-validation-with-zod.md](../day-09/04-client-side-validation-with-zod.md), [01-deterministic-scoring-algorithms-exact-match.md](../day-12/01-deterministic-scoring-algorithms-exact-match.md), [02-partial-credit-scoring-algorithms.md](../day-12/02-partial-credit-scoring-algorithms.md), [02-form-state-collection-with-react-hook-form.md](02-form-state-collection-with-react-hook-form.md). Forward references: [06-submit-and-lock-ux-patterns.md](06-submit-and-lock-ux-patterns.md), [07-graceful-network-failure-handling.md](07-graceful-network-failure-handling.md).*
