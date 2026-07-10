# Filtering And Search UX Patterns

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D18's aggregate endpoint already accepts query parameters: `test_id`, `from`, `to`. The frontend now needs UI that drives those parameters — a test selector dropdown, a date-range picker, and the discipline to thread those choices back into the URL so the dashboard view is *shareable* (the trainer pastes a URL in Slack, the recipient sees the same filtered view) and *reversible* (the back button restores the previous filter set). Three patterns make this work and the cohort should leave with all three in their vocabulary: **debounced inputs** so typing doesn't fire a hundred requests, **URL-synced state** so the URL is the source of truth, and **server-component-friendly filtering** so the same SSR rendering path serves filtered and unfiltered states. Filters are deceptively easy to get wrong — race conditions, lost state on refresh, jittery network — and getting them right is one of the highest-value polish-day investments.

## Objective

Implement filter and search UX that's debounced, URL-synced, and shareable.

## The Three Properties Of A Good Filter

1. **Debounced.** Typing "JavaScript" into a search box should not fire 10 network requests; it should fire one, ~300ms after the user stops typing. Otherwise the network is thrashed and the results flicker.
2. **URL-synced.** The current filter set lives in the URL query string. Refresh the page → same filter applied. Share the URL → same filter applied. Back button → previous filter.
3. **SSR-friendly.** Because URL is the source of truth, the server component can read `searchParams` and pre-render the filtered view. No client-side "loading...then filter" flicker on initial load.

These three together make the difference between a filter that *works* and a filter that *feels solid*.

## URL As Source Of Truth

The instinct from prior frameworks is to put filter state in React state and hold it locally. Resist. Two reasons:

- **Refresh loses state.** A trainer filters to test_id=t_001, refreshes, the dropdown resets. Bad.
- **No share.** A trainer wants to send "look at this view" to a colleague. With local state, the URL doesn't change and there's nothing to send.

URL state fixes both. Use Next.js's `useSearchParams` to read the current filter and `router.replace` to write new ones without adding a history entry:

```tsx
// frontend/app/admin/reports/FilterBar.tsx
"use client";

import { useRouter, useSearchParams, usePathname } from "next/navigation";
import { useCallback } from "react";

export function FilterBar({ tests }: { tests: { id: string; title: string }[] }) {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const setParam = useCallback(
    (key: string, value: string | null) => {
      const params = new URLSearchParams(searchParams.toString());
      if (value == null || value === "") {
        params.delete(key);
      } else {
        params.set(key, value);
      }
      router.replace(`${pathname}?${params.toString()}`);
    },
    [pathname, router, searchParams],
  );

  const currentTest = searchParams.get("test_id") ?? "";
  const currentFrom = searchParams.get("from") ?? "";
  const currentTo = searchParams.get("to") ?? "";

  return (
    <div className="flex flex-wrap gap-3 rounded-lg border border-slate-200 bg-slate-50 p-3">
      <select
        aria-label="Filter by test"
        value={currentTest}
        onChange={(e) => setParam("test_id", e.target.value || null)}
        className="rounded border px-2 py-1 text-sm"
      >
        <option value="">All tests</option>
        {tests.map((t) => (
          <option key={t.id} value={t.id}>{t.title}</option>
        ))}
      </select>
      <input
        type="date"
        aria-label="From date"
        value={currentFrom}
        onChange={(e) => setParam("from", e.target.value || null)}
        className="rounded border px-2 py-1 text-sm"
      />
      <input
        type="date"
        aria-label="To date"
        value={currentTo}
        onChange={(e) => setParam("to", e.target.value || null)}
        className="rounded border px-2 py-1 text-sm"
      />
      <button
        onClick={() => router.replace(pathname)}
        className="rounded border px-3 py-1 text-sm text-slate-600 hover:bg-slate-100"
      >
        Clear filters
      </button>
    </div>
  );
}
```

Two important details:

- `router.replace` not `router.push` for filter changes. Filters shouldn't bloat the back-button history with every keystroke. `push` is for navigations the user wants to undo *one at a time* — filter changes aren't that.
- Empty value → `params.delete(key)` rather than `set(key, "")`. A URL with `?test_id=` is uglier and parses differently than `?` with no key at all.

## Debouncing Text Input

Date pickers and dropdowns fire one event per selection — no debounce needed. A text search input (e.g., "find candidates by name" — not in PEP scope today but plausible) fires on every keystroke and must be debounced.

A minimal `useDebouncedValue` hook:

```ts
// frontend/lib/hooks/useDebouncedValue.ts
import { useEffect, useState } from "react";

export function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);

  return debounced;
}
```

Used in a search component:

```tsx
const [draft, setDraft] = useState(searchParams.get("q") ?? "");
const debounced = useDebouncedValue(draft, 300);

useEffect(() => {
  setParam("q", debounced || null);
}, [debounced, setParam]);
```

The `draft` updates on every keystroke (input feels responsive); the URL only updates after 300ms of quiet (network doesn't thrash). 300ms is the conventional sweet spot — short enough to feel snappy, long enough to swallow keystrokes mid-word.

For a single-input PEP dashboard, `use-debounce` from npm is also fine if the cohort prefers a library. The hand-rolled version is six lines and worth understanding.

## Server-Component Filtering

The dashboard page is a server component. It receives `searchParams` from Next.js and uses them in the fetch:

```tsx
// frontend/app/admin/reports/page.tsx (excerpt)
import { fetchAggregate, fetchTests } from "@/lib/api/reports";
import { FilterBar } from "./FilterBar";
import { AdminReportsView } from "./AdminReportsView";

export default async function AdminReportsPage({
  searchParams,
}: {
  searchParams: { test_id?: string; from?: string; to?: string };
}) {
  // (Role check from Topic 1 omitted here for brevity.)
  const [data, tests] = await Promise.all([
    fetchAggregate(searchParams),
    fetchTests(),
  ]);

  return (
    <main className="mx-auto max-w-6xl space-y-6 p-6">
      <FilterBar tests={tests} />
      <AdminReportsView data={data} />
    </main>
  );
}
```

The flow:

1. User changes a filter → `router.replace('?test_id=t_001')`.
2. Next.js detects the URL change and re-renders the server component with new `searchParams`.
3. `fetchAggregate` is called with the new params → backend filters → fresh data.
4. The page re-streams with the new data.

No client-side fetch. No `useEffect`. No "I'm loading" flicker on the existing data area while the new data arrives — Next.js handles the transition. This is the SSR-friendly model and it's a big quality-of-life upgrade over CSR filtering.

Pair with a `<Suspense>` boundary (Topic 5) and a loading skeleton, so the small in-flight gap during the re-fetch is filled with a placeholder rather than a flash of empty.

## Why URL-Synced Is The Right Default For Dashboards

A trainer is debugging "why is the JavaScript test failing so many candidates?" They filter by `test_id=t_001`, copy the URL, paste it into Slack to their colleague. The colleague clicks, sees the same filtered view, can immediately discuss the same data. This pattern is gold for collaborative debugging and impossible with local state.

Similarly: a trainer bookmarks a "JavaScript Fundamentals, last 30 days" view as their default morning check. Local state can't be bookmarked. URL state can.

The cohort should think of dashboard filters as *view URLs* the user can share, not as transient client state.

## Resetting Filters

A "Clear filters" button restores the unfiltered view by navigating to the bare path:

```tsx
router.replace(pathname);
```

That removes all search params. A subtler choice: should "Clear filters" navigate via `push` (so the back button restores the filtered view) or `replace` (so the history doesn't include the filter state at all)? PEP uses `replace` for filter changes and `push` for "Clear" so the trainer can back-button to their last filtered view. Either choice is defensible; pick one and document.

## Edge Cases

A few worth handling:

- **Invalid params from a hand-edited URL.** A trainer types `?test_id=banana` directly. The backend either returns empty or 422s on validation. The frontend should not crash — if the empty/empty state is handled (Topic 5), this is automatic.
- **Future date filters.** `to` set to a date in the future is meaningless but not invalid. Either accept it silently or display "as of today" in the result text.
- **`from > to`.** Validate either on the frontend (show "From must be before To") or on the backend (422). PEP validates on both, but the user-facing message should come from the frontend.
- **Empty result for filter combo.** Covered in Topic 5's empty-state work.

## URL Hygiene

Two small touches:

- **Ordering of keys.** `?from=2026-05-01&test_id=t_001` and `?test_id=t_001&from=2026-05-01` are functionally identical but textually different. `URLSearchParams` preserves insertion order, so be consistent — either sort keys alphabetically before stringifying, or always set them in the same order. Helps with caching (CDNs sometimes treat key order as cache-distinguishing) and copy-paste-comparing two URLs.
- **Default values aren't in the URL.** If "All tests" is the default for the test selector, don't put `?test_id=` in the URL — just omit it. The default doesn't need to be encoded.

## Anti-Patterns

- **Local state for filters.** Refresh loses state, no share, no bookmark. Bad default.
- **`router.push` on every filter change.** History pollution; back button cycles through every keystroke. Use `replace`.
- **No debounce on a text input.** Network thrash, flickery results. 300ms is the conventional debounce.
- **Filtering client-side.** Fetch all data, filter in JS. Works for 10 rows, breaks at 10,000. Filter on the backend (D18's endpoint already supports it).
- **Two sources of truth.** Filter state in URL *and* in local React state, kept in sync via `useEffect`. Inevitable drift; one source only.
- **No "Clear filters" button.** Trainer is stuck building a new URL by hand. Always provide an escape hatch.
- **Date inputs with no timezone clarity.** `from=2026-05-01` could mean midnight UTC or midnight local. Document and be consistent — backend should treat date-only inputs as the user's day in UTC, with a note in the API docs.

## Connecting Back To D18

D18's filter and sort parameter design built the *server-side contract* for filtering: which params, what they mean, how they validate. Today's work is the *client-side surface* that drives those params — and the URL is the bridge. The trainer's mental model: "the URL is my filter, the backend honors it, the chart redraws." That's the senior-frontend mental model the cohort should leave with.

## Key Takeaways

- The URL is the source of truth for filter state. Local state for filters is a smell.
- `useSearchParams` + `router.replace` is the Next.js pattern for URL-synced filters.
- Debounce text inputs at ~300ms with `useDebouncedValue` or `use-debounce`. Dropdowns and date pickers don't need it.
- Server components read `searchParams` and re-fetch on URL change — no client-side fetch needed, no loading flicker beyond Suspense fallback.
- Filter URLs are shareable, bookmarkable, and back-buttonable. That's a big UX win over local state.
- Defaults are omitted from the URL, not encoded as empty values. Keys ordered consistently. "Clear filters" navigates to the bare path.

---
*Prerequisites: [04-client-components-for-stateful-interactivity.md](../day-13/04-client-components-for-stateful-interactivity.md), [01-server-components-for-data-heavy-pages.md](../day-17/01-server-components-for-data-heavy-pages.md), [02-filter-and-sort-parameter-design-for-scale.md](../day-18/02-filter-and-sort-parameter-design-for-scale.md).*
