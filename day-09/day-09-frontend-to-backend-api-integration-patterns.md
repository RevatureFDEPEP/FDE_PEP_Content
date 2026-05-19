# Frontend-to-Backend API Integration Patterns

> *Day 9: Question Authoring Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 2: Operationalize + First Vertical Slice (Question Authoring)*

## Overview
The Next.js client must POST a question payload to Day 8's `question-management-service` (via the api-gateway), handle 201s by surfacing the new id, and translate FastAPI's 422 validation errors back into per-field RHF errors. We cover client-side `fetch` (the primary pattern for this slice because 422 inline-error handling is easier) and mention Server Actions as an alternative.

## Choosing the Integration Pattern

Two options in Next.js 16:

| Pattern | When to use |
|---|---|
| **Client-side `fetch` from a client component** | Need progress, optimistic UI, fine-grained error mapping (our case — 422 → field errors). |
| **Server Action** (`'use server'` function called from a form) | Simple form posts with redirect-on-success; no real-time progress. |

For Day 9 we use client-side `fetch` because the 422 mapping to per-field errors is far more ergonomic when you control the response handling in the same client component that owns the RHF instance.

## The Backend Contract (Recap of Day 8)

```
POST {API_BASE}/api/questions
Content-Type: application/json

Body (single-select):
{
  "type": "single_select",
  "prompt": "What is 2 + 2?",
  "imageKey": "questions/abc123.png",  // optional
  "choices": [
    { "text": "3", "correct": false },
    { "text": "4", "correct": true  },
    { "text": "5", "correct": false }
  ]
}

201 Created:
{ "_id": "01HXYZ...", "type": "single_select", ... }

422 Unprocessable Entity:
{
  "detail": [
    { "loc": ["body", "choices"], "msg": "...", "type": "value_error" }
  ]
}
```

## Where the API Lives

Set the base URL via environment variable, not a hardcoded literal:

```bash
# .env.local
NEXT_PUBLIC_API_BASE_URL=http://localhost:8080  # api-gateway
```

```ts
// lib/api.ts
export const API_BASE = process.env.NEXT_PUBLIC_API_BASE_URL!;
```

`NEXT_PUBLIC_*` env vars are inlined into the client bundle at build time. Anything not prefixed is server-only.

## A Typed `fetch` Wrapper

Wrap `fetch` so every call has consistent error handling:

```ts
// lib/api.ts
import { API_BASE } from './config';

export type ApiError = {
  status: number;
  detail: Array<{ loc: (string | number)[]; msg: string; type: string }>;
};

export async function apiPost<TReq, TRes>(path: string, body: TReq): Promise<TRes> {
  const res = await fetch(`${API_BASE}${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
    credentials: 'include', // if api-gateway uses cookie auth
  });

  if (res.status === 201 || res.status === 200) {
    return (await res.json()) as TRes;
  }

  // FastAPI errors arrive as { detail: [...] } for 422, sometimes a string for 4xx
  const payload = await res.json().catch(() => ({}));
  const err: ApiError = {
    status: res.status,
    detail: Array.isArray(payload.detail)
      ? payload.detail
      : [{ loc: [], msg: String(payload.detail ?? res.statusText), type: 'http_error' }],
  };
  throw err;
}
```

## Mapping 422 Errors to React-Hook-Form Fields

Day 8's `loc` paths look like `['body', 'choices', 0, 'text']`. We strip the leading `'body'` and feed the rest to RHF's `setError`:

```ts
// lib/form-errors.ts
import { UseFormSetError, FieldValues, Path } from 'react-hook-form';
import type { ApiError } from './api';

export function applyApiErrorsToForm<T extends FieldValues>(
  err: ApiError,
  setError: UseFormSetError<T>
) {
  for (const issue of err.detail) {
    const path = issue.loc.filter((seg) => seg !== 'body').join('.') as Path<T>;
    if (path) {
      setError(path, { type: 'server', message: issue.msg });
    } else {
      setError('root.serverError' as Path<T>, { type: 'server', message: issue.msg });
    }
  }
}
```

This maps `['body', 'choices', 0, 'text']` → `choices.0.text` — RHF's exact path syntax.

## Putting It All Together in the Form

```tsx
'use client';
import { apiPost } from '@/lib/api';
import { applyApiErrorsToForm } from '@/lib/form-errors';
import type { QuestionFormValues } from '@/lib/schemas/question';

const onSubmit = async (data: QuestionFormValues) => {
  setSubmitState({ status: 'submitting' });
  try {
    const created = await apiPost<QuestionFormValues, { _id: string }>(
      '/api/questions',
      data
    );
    setSubmitState({ status: 'success', questionId: created._id });
    form.reset();
    router.refresh();
  } catch (e) {
    const err = e as ApiError;
    if (err.status === 422) {
      applyApiErrorsToForm(err, form.setError);
      setSubmitState({ status: 'error', message: 'Please fix the highlighted fields.' });
    } else {
      setSubmitState({
        status: 'error',
        message: err.detail?.[0]?.msg ?? `Server error (${err.status}).`,
      });
    }
  }
};
```

## CORS, Auth, and the API Gateway

Direct browser → `question-management-service` calls bypass the gateway and lose cross-cutting concerns. Always route through `api-gateway` (the FastAPI BFF added in Week 1) which handles:

- CORS headers for `http://localhost:3000`.
- Auth token forwarding from the `Authorization` cookie/header.
- Service discovery (Next.js doesn't need to know the internal hostname of `question-management-service`).

If you see `Access-Control-Allow-Origin` errors in the browser console, check the gateway's CORS middleware, not the downstream service.

## Server Action Alternative (Brief)

For completeness — a Server Action for the same submit:

```tsx
// app/questions/new/actions.ts
'use server';
import { revalidatePath } from 'next/cache';

export async function createQuestion(data: QuestionFormValues) {
  const res = await fetch(`${process.env.API_BASE_URL}/api/questions`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  if (!res.ok) {
    return { ok: false as const, error: await res.json() };
  }
  revalidatePath('/questions');
  return { ok: true as const, data: await res.json() };
}
```

Useful when you don't need progress reporting and want the secret-y `API_BASE_URL` to stay server-side. For our drag-drop upload UX, the client `fetch` approach is the better fit.

## Example / Worked Scenario

A trainer submits a multi-select with only 1 correct choice. They've bypassed zod somehow (e.g., correct flag changed via setValue without revalidation). The flow:

1. `apiPost` POSTs the payload.
2. Backend's `model_validator` rejects → 422 with `detail: [{ loc: ['body', 'choices'], msg: 'multi-select must have at least 2 correct', type: 'value_error' }]`.
3. `applyApiErrorsToForm` sets `errors.choices = { type: 'server', message: '...' }`.
4. The choices block re-renders with the inline error visible.
5. `submitState` flips to `error`; the trainer fixes the field; on next change, RHF clears the server error and they can resubmit.

## Common Pitfalls

- **Hardcoding `http://localhost:8080`.** Breaks in any non-local environment. Use `NEXT_PUBLIC_API_BASE_URL`.
- **Forgetting `credentials: 'include'`** when the gateway uses cookies. The browser silently omits the cookie and you get 401s you can't reproduce in Postman.
- **Not stripping `'body'` from `loc`.** RHF can't find a field named `body.choices.0.text` and your errors silently vanish into `errors.root`.
- **Calling backend services directly instead of via api-gateway.** You'll fight CORS forever and re-implement auth in three places.
- **Treating `res.ok` as "data is good."** A 200 with `{ detail: 'something weird' }` is rare but possible. Validate the response with zod if you're paranoid.

## Key Takeaways
- Use client-side `fetch` for this slice — easier 422 mapping than Server Actions.
- Wrap `fetch` in a typed helper that normalizes errors into a known `ApiError` shape.
- Map FastAPI's `detail[].loc` (minus the `'body'` prefix) to RHF field paths via `setError`.
- Route through `api-gateway`; never call downstream services directly from the browser.
- `NEXT_PUBLIC_*` env vars expose values to the client bundle; everything else is server-only.

---
*Prerequisites: day-9-conditional-form-rendering-and-submission-states, day-8 backend CRUD topics.*
