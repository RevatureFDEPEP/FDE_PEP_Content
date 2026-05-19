# Client-Side Validation with Zod (Mirroring Backend Schemas)

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Day 8 built Pydantic v2 schemas on the backend with discriminated unions and `model_validator` rules — exactly one correct choice for single-select, at least two for multi-select. Today we mirror those rules in **zod**, the TypeScript-side equivalent of Pydantic, so the user gets instant inline feedback without a round-trip. The backend remains the source of truth; zod is the client-side fast path.

## Zod = Pydantic for TypeScript

Both libraries solve the same problem: declarative schemas that validate at runtime and produce a typed value. The cognitive mapping is direct:

| Pydantic v2 | Zod |
|---|---|
| `BaseModel` | `z.object({ ... })` |
| `Field(min_length=1)` | `z.string().min(1)` |
| `Literal["a", "b"]` | `z.enum(['a', 'b'])` |
| `Discriminated Union` | `z.discriminatedUnion('type', [...])` |
| `@model_validator` | `.refine()` / `.superRefine()` |
| `model.model_validate(data)` | `schema.parse(data)` / `safeParse(data)` |

## Mirroring Day 8's Discriminated Union

Recall Day 8's shape (from `question-management-service`):

```python
# Backend (Pydantic, Day 8)
class SingleSelectQuestion(BaseModel):
    type: Literal["single_select"]
    prompt: str = Field(min_length=1)
    choices: list[Choice]  # validator: exactly one correct

class MultiSelectQuestion(BaseModel):
    type: Literal["multi_select"]
    prompt: str = Field(min_length=1)
    choices: list[Choice]  # validator: at least two correct

Question = Annotated[
    Union[SingleSelectQuestion, MultiSelectQuestion],
    Field(discriminator="type"),
]
```

The zod mirror in `lib/schemas/question.ts`:

```ts
import { z } from 'zod';

const choiceSchema = z.object({
  text: z.string().min(1, 'Choice text is required'),
  correct: z.boolean(),
});

const baseQuestion = z.object({
  prompt: z.string().min(1, 'Prompt is required'),
  imageKey: z.string().optional(), // S3/MinIO object key, set after upload
  choices: z.array(choiceSchema).min(2, 'At least 2 choices required'),
});

const singleSelectSchema = baseQuestion.extend({
  type: z.literal('single_select'),
}).refine(
  (q) => q.choices.filter((c) => c.correct).length === 1,
  {
    message: 'Single-select questions must have exactly one correct choice',
    path: ['choices'],
  }
);

const multiSelectSchema = baseQuestion.extend({
  type: z.literal('multi_select'),
}).refine(
  (q) => q.choices.filter((c) => c.correct).length >= 2,
  {
    message: 'Multi-select questions must have at least two correct choices',
    path: ['choices'],
  }
);

export const questionSchema = z.discriminatedUnion('type', [
  singleSelectSchema,
  multiSelectSchema,
]);

export type QuestionFormValues = z.infer<typeof questionSchema>;
```

Note `z.infer<typeof questionSchema>` — you get the TypeScript type **for free** from the schema. No duplicate type declaration.

## Wiring to react-hook-form via `zodResolver`

```bash
pnpm add zod @hookform/resolvers
```

```tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { questionSchema, type QuestionFormValues } from '@/lib/schemas/question';

export function QuestionAuthorForm() {
  const form = useForm<QuestionFormValues>({
    resolver: zodResolver(questionSchema),
    defaultValues: {
      type: 'single_select',
      prompt: '',
      choices: [
        { text: '', correct: false },
        { text: '', correct: false },
      ],
    },
    mode: 'onBlur',
  });

  // form.formState.errors is now typed and populated by zod
  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <input {...form.register('prompt')} />
      {form.formState.errors.prompt && (
        <p className="text-red-600 text-sm">
          {form.formState.errors.prompt.message}
        </p>
      )}
      {/* ... */}
    </form>
  );
}
```

When the user blurs the prompt field empty, zod throws, `zodResolver` translates the issue to RHF's error map, and the message renders inline.

## Discriminated Union and `refine` Paths

Cross-field validation lives in `.refine()` (single check) or `.superRefine()` (multiple issues at once). The `path` option places the error against a specific form field so RHF surfaces it correctly:

```ts
}).refine(
  (q) => q.choices.filter((c) => c.correct).length === 1,
  {
    message: 'Single-select must have exactly one correct choice',
    path: ['choices'],   // <-- error attaches to `choices`, not the form root
  }
);
```

Without `path`, the issue lands at the form root (`errors.root`) and you have to render it as a banner rather than inline.

## `safeParse` for Server Response Validation

When the backend returns a 422 with its own error payload (Day 8 used loc/fragment), you can also use zod to validate the response shape before consuming it:

```ts
const errorPayload = z.object({
  detail: z.array(z.object({
    loc: z.array(z.union([z.string(), z.number()])),
    msg: z.string(),
    type: z.string(),
  })),
});

const result = errorPayload.safeParse(await res.json());
if (result.success) {
  // map result.data.detail back to RHF field errors
}
```

## Example / Worked Scenario

A trainer selects multi-select, fills the prompt, adds 4 choices, marks only one correct. They blur out of the last choice:

1. RHF triggers validation via `zodResolver`.
2. The multi-select branch's `.refine` runs: `choices.filter(c => c.correct).length === 1`, which is **not** `>= 2`.
3. zod emits an issue with `path: ['choices']` and message "Multi-select questions must have at least two correct choices".
4. `form.formState.errors.choices.message` is populated; the UI renders it under the choices block.
5. The trainer ticks a second correct box → next change cycle re-runs zod → error clears → submit button enables.

## Common Pitfalls

- **Schema drift from the backend.** The whole point of mirroring is identical rules. When you tighten the backend (e.g., add `max_length=500` to prompt), update the zod schema in the same PR. Consider extracting an OpenAPI client to automate this.
- **Forgetting `path` on `.refine`.** The error lands at root and looks broken. Always set `path` for cross-field rules.
- **Using `z.union` instead of `z.discriminatedUnion`.** Plain unions try every branch silently, produce worse errors, and lose TypeScript narrowing on the `type` field.
- **Calling `schema.parse(...)` outside a try/catch.** `parse` throws on failure; `safeParse` returns `{ success, data | error }`. Inside RHF you don't touch either directly — `zodResolver` handles it.
- **Skipping zod and relying only on backend 422s.** The form feels sluggish (full network round-trip for "prompt required"). Mirror locally, treat backend as authority.

## Key Takeaways
- Zod is to TypeScript what Pydantic is to Python: declarative runtime schemas with type inference.
- `z.discriminatedUnion('type', [...])` mirrors Pydantic's discriminated union exactly.
- `.refine(fn, { message, path })` mirrors `@model_validator` for cross-field rules.
- Wire to react-hook-form with `zodResolver(schema)`; `z.infer` gives you the form values type.
- Backend is the source of truth; zod is the fast path — keep them in sync when either changes.

---
*Prerequisites: day-9-form-state-management-with-react-hook-form, day-8 backend question schemas.*
