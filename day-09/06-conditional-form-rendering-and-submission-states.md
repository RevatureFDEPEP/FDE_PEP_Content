# Conditional Form Rendering and Submission States

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
Single-select and multi-select share most fields but diverge on choice UI — single-select uses radio buttons (exactly one correct), multi-select uses checkboxes (≥2 correct). The form must react to the user's type selection in real time and surface a clear submission state machine (idle → submitting → success | error). This topic ties together `watch`, conditional rendering, and disciplined state management for submission UX.

## Reading Form Values Live with `watch`

`form.watch('type')` returns the current value and re-renders the component whenever it changes:

```tsx
'use client';
import { useForm, useFieldArray } from 'react-hook-form';

export function QuestionAuthorForm() {
  const form = useForm<QuestionFormValues>({ /* ... */ });
  const type = form.watch('type'); // 'single_select' | 'multi_select'

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* ...prompt, type selector... */}
      {type === 'single_select' ? <SingleSelectChoices form={form} />
                                : <MultiSelectChoices form={form} />}
    </form>
  );
}
```

`watch` triggers re-renders, so call it sparingly at the form root and pass values down. For deeply nested reads inside a child, consider `useWatch` which scopes subscriptions:

```tsx
import { useWatch } from 'react-hook-form';
const type = useWatch({ control: form.control, name: 'type' });
```

## Conditional Choice UI

Single-select: a radio group where exactly one choice can be `correct: true`. Multi-select: independent checkboxes.

```tsx
function SingleSelectChoices({ form }: { form: UseFormReturn<QuestionFormValues> }) {
  const { fields, append, remove } = useFieldArray({ control: form.control, name: 'choices' });
  const correctIndex = form.watch('choices').findIndex((c) => c.correct);

  const setCorrect = (idx: number) => {
    const choices = form.getValues('choices').map((c, i) => ({
      ...c,
      correct: i === idx,
    }));
    form.setValue('choices', choices, { shouldValidate: true });
  };

  return (
    <div className="space-y-2">
      {fields.map((field, i) => (
        <div key={field.id} className="flex items-center gap-2">
          <input
            type="radio"
            name="correctChoice"
            checked={correctIndex === i}
            onChange={() => setCorrect(i)}
          />
          <input
            {...form.register(`choices.${i}.text`)}
            placeholder={`Choice ${i + 1}`}
            className="flex-1 border rounded px-2 py-1"
          />
          <button type="button" onClick={() => remove(i)} disabled={fields.length <= 2}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => append({ text: '', correct: false })}>
        Add choice
      </button>
    </div>
  );
}
```

```tsx
function MultiSelectChoices({ form }: { form: UseFormReturn<QuestionFormValues> }) {
  const { fields, append, remove } = useFieldArray({ control: form.control, name: 'choices' });
  return (
    <div className="space-y-2">
      {fields.map((field, i) => (
        <div key={field.id} className="flex items-center gap-2">
          <input type="checkbox" {...form.register(`choices.${i}.correct`)} />
          <input
            {...form.register(`choices.${i}.text`)}
            placeholder={`Choice ${i + 1}`}
            className="flex-1 border rounded px-2 py-1"
          />
          <button type="button" onClick={() => remove(i)} disabled={fields.length <= 2}>
            Remove
          </button>
        </div>
      ))}
      <button type="button" onClick={() => append({ text: '', correct: false })}>
        Add choice
      </button>
    </div>
  );
}
```

## Resetting Cross-Type State on Switch

When a trainer switches from multi-select (3 correct) to single-select, the form is now invalid by zod's rules. Decide your UX: either let zod surface the error, or proactively reset on type change.

```tsx
const type = form.watch('type');

useEffect(() => {
  // when type changes, clear all `correct` flags
  const cleared = form.getValues('choices').map((c) => ({ ...c, correct: false }));
  form.setValue('choices', cleared, { shouldValidate: false });
}, [type]);
```

Be deliberate: resetting silently can frustrate users who switched accidentally. A small toast ("Choices reset because question type changed") is often the right move.

## Submission State Machine

Model submission as an explicit state, not three booleans:

```tsx
type SubmitState =
  | { status: 'idle' }
  | { status: 'submitting' }
  | { status: 'success'; questionId: string }
  | { status: 'error'; message: string };

const [submitState, setSubmitState] = useState<SubmitState>({ status: 'idle' });

const onSubmit = async (data: QuestionFormValues) => {
  setSubmitState({ status: 'submitting' });
  try {
    const res = await fetch('/api/questions', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });
    if (!res.ok) {
      const errorBody = await res.json().catch(() => ({}));
      throw new Error(errorBody.detail?.[0]?.msg ?? `HTTP ${res.status}`);
    }
    const created = await res.json();
    setSubmitState({ status: 'success', questionId: created._id });
    form.reset(); // clear fields for next entry
  } catch (e) {
    setSubmitState({ status: 'error', message: (e as Error).message });
  }
};
```

Render based on state:

```tsx
<Button type="submit" disabled={submitState.status === 'submitting'}>
  {submitState.status === 'submitting' ? 'Saving…' : 'Save question'}
</Button>

{submitState.status === 'success' && (
  <p className="text-green-700">Question {submitState.questionId} saved.</p>
)}
{submitState.status === 'error' && (
  <p className="text-red-600">Could not save: {submitState.message}</p>
)}
```

Using a discriminated union for `SubmitState` (just like our zod schema) means TypeScript narrows correctly and you can't accidentally render the questionId when the status is `'error'`.

## Example / Worked Scenario

A trainer:

1. Lands on `/questions/new`, sees the form in `idle` state, type defaulted to single-select with radio UI.
2. Toggles to multi-select — `useEffect` clears `correct` flags, choice UI swaps to checkboxes.
3. Fills prompt + 4 choices, marks 2 correct, clicks Save.
4. State transitions: `idle` → `submitting` (button shows "Saving…", disabled).
5. Backend returns 201; state → `success`; green message renders with the new question id; `form.reset()` clears the form for the next entry.

On a 422:

1. Steps 1–4 as above.
2. Response is 422 with `{ detail: [{ loc: ['choices'], msg: '...' }] }`.
3. State → `error`; red banner shows the message; form retains its values so the trainer can correct.

## Common Pitfalls

- **Calling `watch` deep in a leaf component.** It subscribes the entire form to re-render. Use `useWatch` with a `name` to scope, or pass the value as a prop from the parent.
- **Tracking submission with multiple booleans** (`isLoading`, `isSuccess`, `isError`). They drift into illegal states (`isLoading: true, isSuccess: true`). A discriminated union prevents this by construction.
- **Forgetting to reset the form on success.** Trainers in a batch-entry flow will rage-quit if every save leaves stale text in the prompt.
- **Letting the submit button stay enabled during submit.** Users will double-click and create duplicates. Disable on `status === 'submitting'`.

## Key Takeaways
- `form.watch('type')` (or `useWatch` for scoped subscriptions) drives conditional rendering of choice UI.
- On type switch, reset cross-type state (e.g., `correct` flags) deliberately — silent resets are confusing.
- Model submission as a discriminated-union state: idle | submitting | success | error.
- Disable the submit button while submitting; surface errors inline with the message from the backend.

---
*Prerequisites: [03-form-state-management-with-react-hook-form.md](03-form-state-management-with-react-hook-form.md), [04-client-side-validation-with-zod.md](04-client-side-validation-with-zod.md).*
