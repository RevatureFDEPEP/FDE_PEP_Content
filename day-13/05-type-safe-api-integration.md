# Type-Safe API Integration (TypeScript Types Matching Backend Schemas)

> *Day 13: Test-Taking Page (Frontend Skeleton) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
The backend speaks Pydantic; the frontend speaks TypeScript. Both describe the same wire shape — the D11 session contract, the D12 answer contract, the D8 question discriminator — and any drift between them is a runtime crash waiting for the worst possible moment to fire (the candidate hits "Submit", the response shape doesn't match the type, the page goes white). This topic covers two workflows for keeping the two sides aligned: hand-written TypeScript interfaces that mirror the Pydantic models, and `openapi-typescript` codegen driven off FastAPI's auto-generated `/openapi.json`. The recommendation up front: **adopt codegen**. The hand-written approach is worth understanding so you can read it in other codebases and recognize when it's drifted.

## What "Type Drift" Actually Looks Like

Backend changes a field:

```python
# Before
class QuestionOut(BaseModel):
    options: list[OptionOut]

# After (backend dev renamed to avoid a clash)
class QuestionOut(BaseModel):
    choices: list[OptionOut]
```

Frontend type, untouched:

```ts
type Question = { options: Option[]; ... };
```

TypeScript still compiles, frontend still builds, end-to-end test still passes against the *staging* backend (which hasn't deployed yet). Once production deploys: `question.options` is `undefined`, `.map(...)` throws, `<QuestionView>` crashes, error boundary lights up.

The compiler can't catch what it doesn't know about. Type drift is the silent failure mode of every "frontend talks to backend" system, and it's worth more engineering investment than people typically give it.

## Workflow A: Hand-Written Interfaces

You write a `lib/api/types.ts` file by hand, eyeballing the Pydantic models in the backend repo and mirroring them.

```ts
// frontend/lib/api/types.ts

// Mirrors test_management_service/app/schemas/session.py SessionOut
export type SessionId = string; // UUID, validated at boundary
export type QuestionId = string;
export type IsoTimestamp = string; // ISO 8601, e.g. "2026-05-19T14:32:01Z"

export type Option = {
  id: number;
  label: string;
};

// Discriminator — matches D8 question_type field
export type Question =
  | {
      type: "single_select";
      question_id: QuestionId;
      stem: string;
      options: Option[];
    }
  | {
      type: "multi_select";
      question_id: QuestionId;
      stem: string;
      options: Option[];
    };

export type Session = {
  session_id: SessionId;
  session_token: string; // opaque
  test_id: string;
  server_now: IsoTimestamp;
  expires_at: IsoTimestamp;
  questions: Question[];
};

// Mirrors D12 AnswerSubmit / AnswerResponse
export type AnswerSubmit = {
  question_id: QuestionId;
  selected_options: number[];
};

export type AnswerResponse = {
  score: number; // 0.0–1.0
  next_question: Question | null;
  session_state: "active" | "completed";
};
```

What this buys you:

- **Compile-time checking inside the frontend.** Component props are typed, state is typed, `fetch().then((r) => r.json() as Session)` is typed.
- **Zero build-time dependency on the backend repo.** Frontend can build standalone, useful in early-stage projects where backends churn.
- **Reads like documentation.** A new frontend dev sees the file and understands the API surface in 30 seconds.

What it doesn't buy you:

- **No protection against backend drift.** If the backend renames `options` to `choices`, this file is wrong and nothing tells you until runtime.
- **Maintenance burden scales with API surface.** 4 endpoints today is fine. 40 endpoints with 100 schemas, and every PR touches both sides — it doesn't scale.
- **No discoverability of new endpoints.** New backend endpoint added Monday, frontend dev finds out Friday in code review.

For PEP, hand-written is the *starting* point — you'll use it Day 13 because the codegen tool needs to be added to the project. The migration to codegen happens later in the week or in Week 4.

## Workflow B: `openapi-typescript` Codegen

FastAPI auto-generates an OpenAPI 3.x schema document at `/openapi.json` from your Pydantic models. The `openapi-typescript` npm package consumes that JSON and emits TypeScript types — one giant `paths` interface plus all the schema types.

```bash
# Add to frontend
npm install --save-dev openapi-typescript

# Generate types into lib/api/openapi.ts
npx openapi-typescript http://localhost:8000/openapi.json -o lib/api/openapi.ts
```

The generated file looks like this (excerpt):

```ts
// frontend/lib/api/openapi.ts  (GENERATED — do not edit)
export interface paths {
  "/sessions": {
    post: operations["createSession"];
  };
  "/sessions/{session_id}/answer": {
    post: operations["submitAnswer"];
  };
  // ...
}

export interface components {
  schemas: {
    SessionOut: {
      session_id: string;
      session_token: string;
      test_id: string;
      server_now: string;
      expires_at: string;
      questions: components["schemas"]["QuestionOut"][];
    };
    QuestionOut: components["schemas"]["SingleSelectQuestion"] | components["schemas"]["MultiSelectQuestion"];
    SingleSelectQuestion: {
      type: "single_select";
      question_id: string;
      stem: string;
      options: components["schemas"]["OptionOut"][];
    };
    // ...
  };
}

export interface operations {
  createSession: {
    requestBody: {
      content: { "application/json": { test_id: string } };
    };
    responses: {
      201: { content: { "application/json": components["schemas"]["SessionOut"] } };
      401: { content: { "application/json": components["schemas"]["HTTPError"] } };
    };
  };
}
```

Wrap that with friendlier aliases in a small `types.ts`:

```ts
// frontend/lib/api/types.ts (HAND-WRITTEN aliases for ergonomics)
import type { components, operations } from "./openapi";

export type Session = components["schemas"]["SessionOut"];
export type Question = components["schemas"]["QuestionOut"];
export type SingleSelectQuestion = components["schemas"]["SingleSelectQuestion"];
export type MultiSelectQuestion = components["schemas"]["MultiSelectQuestion"];
export type AnswerSubmit = operations["submitAnswer"]["requestBody"]["content"]["application/json"];
export type AnswerResponse = NonNullable<
  operations["submitAnswer"]["responses"]["200"]["content"]
>["application/json"];
```

Wire it into your dev workflow:

```json
// frontend/package.json
{
  "scripts": {
    "gen:api": "openapi-typescript http://localhost:8000/openapi.json -o lib/api/openapi.ts",
    "predev": "npm run gen:api",
    "prebuild": "npm run gen:api"
  }
}
```

Now `npm run dev` regenerates types from the running backend before starting Next.js, and a build fails if generation fails. If a backend dev renames `options` → `choices`, the next `npm run dev` regenerates `openapi.ts`, the frontend `QuestionView` that reads `question.options` immediately fails to compile, and the drift is caught at compile-time on the frontend dev's machine.

## Why Codegen Wins For PEP

Three reasons:

1. **Single source of truth.** The Pydantic models are the contract. Hand-written interfaces create a *second* source that must be kept in sync; codegen makes the contract one-way (backend defines, frontend derives).
2. **Drift becomes a compile error, not a runtime crash.** A field rename surfaces in `tsc` immediately. A type change surfaces immediately. A removed endpoint surfaces immediately.
3. **It's automatable in CI.** A GitHub Actions step that runs `openapi-typescript` against a backend container and `git diff --exit-code lib/api/openapi.ts` will fail any PR that drifts the contract without regenerating the types.

The honest cost: codegen requires the backend to be available (locally running, or a checked-in `openapi.json` snapshot). For PEP this is fine — `docker-compose up` brings the backend up locally, and the generation script hits `localhost:8000`. For teams where the backend is in a separate repo and not always running, the snapshot-in-repo approach works too: commit `openapi.json`, regenerate on demand.

## Runtime Validation Sits Alongside, Not Instead Of, Types

TypeScript types are erased at runtime. The compiler tells you "this should be a `Session`" but doesn't *enforce* it — if the backend returns garbage, your code happily treats the garbage as a `Session` until something explodes.

For belt-and-suspenders safety on the critical paths, add a runtime parser:

```ts
import { z } from "zod";

const OptionSchema = z.object({ id: z.number().int(), label: z.string() });
const QuestionSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("single_select"), question_id: z.string().uuid(), stem: z.string(), options: z.array(OptionSchema) }),
  z.object({ type: z.literal("multi_select"), question_id: z.string().uuid(), stem: z.string(), options: z.array(OptionSchema) }),
]);
const SessionSchema = z.object({
  session_id: z.string().uuid(),
  session_token: z.string(),
  test_id: z.string().uuid(),
  server_now: z.string(),
  expires_at: z.string(),
  questions: z.array(QuestionSchema),
});

export type Session = z.infer<typeof SessionSchema>; // type derived from the schema
```

`z.infer` gives you the TypeScript type for free, and `SessionSchema.parse(jsonBody)` throws if the response doesn't match. Belt-and-suspenders: codegen types prevent drift you can prevent at build time; Zod parses catch drift the codegen missed (or schemas the backend deploys without regen). For Day 13 the codegen types are enough; layer Zod in Week 4 as a hardening exercise.

## Recommendation

For PEP Day 13:

- **Start hand-written** for the four shapes you need today (`Session`, `Question`, `AnswerSubmit`, `AnswerResponse`) — fastest path to a working page.
- **Migrate to codegen by end of Week 3.** Add `openapi-typescript` to `package.json`, wire the `predev`/`prebuild` scripts, replace the hand-written types with re-exports from the generated file. The migration is a 20-minute exercise and prevents drift for the rest of the program.
- **Add Zod parsing on critical paths** in Week 4 capstone hardening — at least on the session mint and the answer-submit response.

## Common Mistakes

- **Generating types from a stale `openapi.json` committed months ago.** The whole point is freshness. Pin the regeneration to the dev/build lifecycle.
- **Editing the generated file by hand.** Top of file says "do not edit"; honor it. Aliases go in a hand-written `types.ts`.
- **Using `as Session` casts** without a runtime parse. Casts lie to the compiler; runtime data does what it wants.
- **Mirroring Pydantic field naming inconsistently** (camelCase on the TS side when the backend is snake_case). Don't translate; mirror exactly. If you want camelCase for the UI, do the translation in one explicit layer.
- **Forgetting that the discriminator field name has to be a literal string union**, not just `string`. Discriminated unions in TS require literal types on the discriminator — codegen handles this; hand-written types frequently get it wrong.

## Key Takeaways
- Backend Pydantic models and frontend TypeScript types describe the same wire format; drift between them fails at runtime.
- Two workflows: hand-written interfaces (low friction, no drift protection) or `openapi-typescript` codegen from FastAPI's `/openapi.json` (build-time drift detection).
- Recommended for PEP: hand-written for Day 13 expediency, migrate to codegen by end of Week 3.
- Pair generated types with Zod runtime parsing on critical paths (session mint, answer submit) for defense against drift the build didn't catch.
- Make the discriminator field a literal union (`"single_select" | "multi_select"`) so discriminated rendering (Topic 7) is type-safe.

---
*Prerequisites: [01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md](../day-08/01-modeling-polymorphic-data-discriminated-unions-tagged-enums-single-shape.md), day-9-typescript-fundamentals-for-react, [06-pydantic-request-response-modeling.md](../day-11/06-pydantic-request-response-modeling.md). Forward references: [08-polymorphic-component-rendering-for-variant-data-types.md](08-polymorphic-component-rendering-for-variant-data-types.md), [06-submit-and-lock-ux-patterns.md](../day-14/06-submit-and-lock-ux-patterns.md).*
