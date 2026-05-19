# Form State Management with react-hook-form

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
The question authoring form has dynamic choice arrays (2-N options for multi-select), validation that depends on the question type, and an optional image attachment. Managing this with raw `useState` would be painful. `react-hook-form` (RHF) is the de-facto standard for React forms in 2026 — uncontrolled-by-default for performance, with first-class TypeScript and `useFieldArray` for the kind of dynamic lists we need.

## Why react-hook-form

- **Uncontrolled by default** — inputs register with the form via refs, so typing in one field doesn't re-render every other field.
- **Single source of truth** — `form.getValues()`, `form.watch()`, `form.setValue()` give you full programmatic control.
- **Built-in validation hooks** — pairs cleanly with zod (next topic) via `zodResolver`.
- **`useFieldArray`** — append/remove/swap items in array fields without re-implementing keys and indexes.

## The Core Hook: `useForm`

```tsx
'use client';
import { useForm, SubmitHandler } from 'react-hook-form';

type FormValues = {
  prompt: string;
  type: 'single_select' | 'multi_select';
  choices: { text: string; correct: boolean }[];
};

export function QuestionAuthorForm() {
  const form = useForm<FormValues>({
    defaultValues: {
      prompt: '',
      type: 'single_select',
      choices: [
        { text: '', correct: false },
        { text: '', correct: false },
      ],
    },
    mode: 'onBlur', // validate on blur; revalidate on change after first error
  });

  const onSubmit: SubmitHandler<FormValues> = async (data) => {
    // POST to backend (covered in api-integration topic)
    console.log(data);
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* fields go here */}
    </form>
  );
}
```

`form.handleSubmit(onSubmit)` wraps your handler with validation — if validation fails, your `onSubmit` is never called and errors populate `form.formState.errors`.

## Registering Inputs

For a plain input you call `register('fieldName')`:

```tsx
<input
  {...form.register('prompt')}
  className="border rounded px-2 py-1"
/>
{form.formState.errors.prompt && (
  <p className="text-red-600 text-sm">
    {form.formState.errors.prompt.message}
  </p>
)}
```

For custom or controlled components (like shadcn's `Select` or `RadioGroup`), use the `Controller` component or `<FormField>` wrapper:

```tsx
import { Controller } from 'react-hook-form';

<Controller
  control={form.control}
  name="type"
  render={({ field }) => (
    <RadioGroup value={field.value} onValueChange={field.onChange}>
      <RadioGroupItem value="single_select" /> Single select
      <RadioGroupItem value="multi_select" /> Multi select
    </RadioGroup>
  )}
/>
```

## Dynamic Arrays with `useFieldArray`

Choices grow and shrink — a perfect `useFieldArray` job:

```tsx
import { useFieldArray } from 'react-hook-form';

const { fields, append, remove } = useFieldArray({
  control: form.control,
  name: 'choices',
});

return (
  <div className="space-y-2">
    {fields.map((field, index) => (
      <div key={field.id} className="flex gap-2 items-center">
        <input
          {...form.register(`choices.${index}.text`)}
          placeholder={`Choice ${index + 1}`}
          className="flex-1 border rounded px-2 py-1"
        />
        <input
          type="checkbox"
          {...form.register(`choices.${index}.correct`)}
        />
        <button
          type="button"
          onClick={() => remove(index)}
          disabled={fields.length <= 2}
          className="text-red-600"
        >
          Remove
        </button>
      </div>
    ))}
    <button
      type="button"
      onClick={() => append({ text: '', correct: false })}
    >
      Add choice
    </button>
  </div>
);
```

Two crucial points:

1. **Use `field.id` as the React key**, not the array index — RHF generates a stable id per row so React reconciles correctly across reorders/deletes.
2. **Field names use bracket-free dot syntax**: `choices.0.text`, not `choices[0].text`. RHF parses these into nested values.

## Submission State

`form.formState` exposes everything you need:

```tsx
const {
  isSubmitting,   // true while your handleSubmit is running
  isSubmitted,    // true after at least one submit attempt
  isValid,        // current validation status (with onChange mode)
  isDirty,        // any field changed from defaults
  errors,         // { fieldName: { type, message } }
} = form.formState;

<button
  type="submit"
  disabled={isSubmitting || !isValid}
>
  {isSubmitting ? 'Saving…' : 'Save question'}
</button>
```

## Example / Worked Scenario

A trainer opens `/questions/new`. Defaults populate two empty choices. They:

1. Type the prompt — RHF tracks it via `register('prompt')`, no re-renders elsewhere.
2. Switch type to multi-select via `Controller`-wrapped RadioGroup.
3. Click "Add choice" twice — `append({ text: '', correct: false })` adds rows; `fields.length` is now 4.
4. Check two correct boxes — RHF tracks `choices.1.correct` and `choices.2.correct`.
5. Hit submit — `handleSubmit` runs validation (next topic adds zod); if it passes, `onSubmit(data)` fires with a fully typed `FormValues` object.

## Common Pitfalls

- **Using array index as `key`** in `fields.map`. After a `remove`, indexes shift and React mis-reconciles, leaving stale input values. Always use `field.id`.
- **Mixing controlled and uncontrolled patterns.** If you `register` an input but also pass `value={form.watch('foo')}`, you've turned it controlled and lost RHF's perf model. Use either `register` (uncontrolled) or `Controller` (controlled), never both on the same field.
- **Forgetting `type="button"`** on Add/Remove buttons inside the form. Default `type` is `submit`, so clicking "Add choice" submits the form.
- **Reading `form.formState.errors` before submitting in `onChange` mode without setting `mode`.** RHF doesn't validate until you submit by default. Set `mode: 'onBlur'` (or `'onChange'`) in `useForm` for live feedback.

## Key Takeaways
- `useForm<FormValues>()` gives you `register`, `handleSubmit`, `control`, `watch`, `formState`.
- Plain inputs use `register('field')`; controlled UI components use `<Controller>`.
- Dynamic arrays use `useFieldArray` — key by `field.id`, never by index.
- `form.formState` exposes `isSubmitting`, `errors`, `isValid` for UX.

---
*Prerequisites: day-9-server-vs-client-component-boundary.*
