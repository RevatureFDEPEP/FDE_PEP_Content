# Component Composition with Tailwind and shadcn/ui

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
`rev-eval-ai`'s frontend uses Tailwind for utilities and shadcn/ui for accessible primitives. shadcn/ui is not a npm dependency — it's a CLI that copies source code into your repo (`components/ui/`), so you own and can modify every primitive. Today you'll compose `Button`, `Input`, `Label`, `Select`, `RadioGroup`, `Checkbox`, and the shadcn `Form` wrapper (which integrates with react-hook-form) into a coherent question authoring UI.

## How shadcn/ui Works

Run once per primitive:

```bash
pnpm dlx shadcn@latest add button input label select radio-group checkbox form textarea
```

This drops files into `components/ui/`:

```
components/ui/
  button.tsx
  input.tsx
  label.tsx
  select.tsx
  radio-group.tsx
  checkbox.tsx
  form.tsx       # react-hook-form integration helpers
  textarea.tsx
```

Each file is a thin wrapper around a Radix UI primitive with Tailwind classes. You can edit them in place — that's the point. The PEP variant already has these checked in.

## The `Form` Wrapper (RHF Integration)

shadcn's `form.tsx` provides `<Form>`, `<FormField>`, `<FormItem>`, `<FormLabel>`, `<FormControl>`, `<FormMessage>` — these wire RHF's `Controller` + `formState.errors` to accessible markup automatically:

```tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Button } from '@/components/ui/button';
import { RadioGroup, RadioGroupItem } from '@/components/ui/radio-group';
import { Checkbox } from '@/components/ui/checkbox';
import { questionSchema, type QuestionFormValues } from '@/lib/schemas/question';

export function QuestionAuthorForm() {
  const form = useForm<QuestionFormValues>({
    resolver: zodResolver(questionSchema),
    defaultValues: {
      type: 'single_select',
      prompt: '',
      choices: [{ text: '', correct: false }, { text: '', correct: false }],
    },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="prompt"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Prompt</FormLabel>
              <FormControl>
                <Textarea placeholder="What is 2 + 2?" rows={3} {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="type"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Question type</FormLabel>
              <FormControl>
                <RadioGroup
                  value={field.value}
                  onValueChange={field.onChange}
                  className="flex gap-6"
                >
                  <label className="flex items-center gap-2">
                    <RadioGroupItem value="single_select" /> Single select
                  </label>
                  <label className="flex items-center gap-2">
                    <RadioGroupItem value="multi_select" /> Multi select
                  </label>
                </RadioGroup>
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <ChoicesSection form={form} />

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? 'Saving…' : 'Save question'}
        </Button>
      </form>
    </Form>
  );
}
```

`<FormMessage />` automatically renders the zod/RHF error for the surrounding field. No manual `errors.prompt && <p>` boilerplate.

## A Reusable Choice Row

```tsx
import { Input } from '@/components/ui/input';
import { Checkbox } from '@/components/ui/checkbox';
import { Button } from '@/components/ui/button';
import { Trash2 } from 'lucide-react';

function ChoiceRow({
  index,
  form,
  type,
  onRemove,
  canRemove,
}: {
  index: number;
  form: UseFormReturn<QuestionFormValues>;
  type: 'single_select' | 'multi_select';
  onRemove: () => void;
  canRemove: boolean;
}) {
  return (
    <div className="flex items-center gap-3 rounded-md border p-3">
      <FormField
        control={form.control}
        name={`choices.${index}.correct`}
        render={({ field }) =>
          type === 'multi_select' ? (
            <Checkbox checked={field.value} onCheckedChange={field.onChange} />
          ) : (
            <input
              type="radio"
              name="correctChoice"
              checked={field.value}
              onChange={() => {/* see conditional-rendering topic */}}
              className="h-4 w-4"
            />
          )
        }
      />
      <FormField
        control={form.control}
        name={`choices.${index}.text`}
        render={({ field }) => (
          <FormItem className="flex-1">
            <FormControl>
              <Input placeholder={`Choice ${index + 1}`} {...field} />
            </FormControl>
            <FormMessage />
          </FormItem>
        )}
      />
      <Button
        type="button"
        variant="ghost"
        size="icon"
        onClick={onRemove}
        disabled={!canRemove}
        aria-label="Remove choice"
      >
        <Trash2 className="h-4 w-4" />
      </Button>
    </div>
  );
}
```

## Tailwind Composition Tips

Tailwind classes pile up — keep them readable:

- **Group with `space-y-*` / `gap-*`** on parents instead of repeating margins on children.
- **Use `cn(...)` helper** (`lib/utils.ts`, included by shadcn) to merge conditional classes:

  ```tsx
  import { cn } from '@/lib/utils';
  <Button className={cn('w-full', isDanger && 'bg-red-600 hover:bg-red-700')} />
  ```

- **Lean on shadcn variants** (`variant="ghost"`, `variant="destructive"`, `size="icon"`) rather than ad-hoc Tailwind on `Button`. The variants ensure consistent focus rings, disabled states, and dark-mode behavior.
- **Layout containers**: prefer `container mx-auto max-w-3xl py-8` on the outermost `main` for a centered form; let children fill width.

## Accessibility for Free

Because shadcn primitives wrap Radix, you get:

- `FormLabel` linked to the input via `htmlFor` / `id` automatically.
- `aria-describedby` pointing to the `FormMessage` for screen readers.
- `aria-invalid="true"` on inputs with errors.
- Keyboard navigation (Tab, arrow keys in RadioGroup, Space to toggle Checkbox).

Don't undo this by replacing `<FormControl>` with raw `<input>` — the wiring goes away.

## Example / Worked Scenario

You compose the form in roughly 90 minutes:

1. Scaffold the page (server component) and import `QuestionAuthorForm` from `components/questions/`.
2. Build `QuestionAuthorForm` with `<Form>` wrapper, prompt `Textarea`, type `RadioGroup`.
3. Extract `ChoiceRow` and a `ChoicesSection` that uses `useFieldArray`.
4. Pipe `form.watch('type')` to switch row UI between radio and checkbox.
5. Drop in the eventual `ImageDropzone` between prompt and choices.
6. The submit `<Button>` wires to `form.formState.isSubmitting`.

Final rendering is consistent with the rest of the app's chrome because every primitive uses the same Tailwind theme tokens.

## Common Pitfalls

- **Treating shadcn primitives as a node_modules dependency.** They're in your repo. If a button doesn't behave right, open `components/ui/button.tsx` and read the source.
- **Replacing `<FormControl>` with a raw element** to "simplify" — loses all the a11y wiring.
- **Forgetting `value`/`onValueChange` on Radix-based components.** They're not standard HTML — `RadioGroup` uses `value`/`onValueChange`, `Checkbox` uses `checked`/`onCheckedChange`. Standard `onChange` does nothing.
- **Class explosion on a single element.** If you write `className="flex items-center justify-between gap-4 rounded-md border p-3 bg-card hover:bg-muted/50 transition-colors"`, extract a component or a named class. 5+ utility classes on one element is a smell.

## Key Takeaways
- shadcn/ui copies primitives into `components/ui/` — you own and can edit them.
- Use the `<Form>` / `<FormField>` wrappers to integrate with react-hook-form; `<FormMessage>` renders errors automatically.
- Radix-based primitives use `value`/`onValueChange` and `checked`/`onCheckedChange`, not standard `onChange`.
- Compose layouts with parent-level `space-y`/`gap` and the `cn` helper; lean on shadcn variants over ad-hoc Tailwind.

---
*Prerequisites: [03-form-state-management-with-react-hook-form.md](03-form-state-management-with-react-hook-form.md), [04-client-side-validation-with-zod.md](04-client-side-validation-with-zod.md).*
