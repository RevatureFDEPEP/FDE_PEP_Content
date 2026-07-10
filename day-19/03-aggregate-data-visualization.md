# Aggregate Data Visualization

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

D17 introduced recharts with a single-series bar chart on the candidate results page: per-question time-taken, color-encoded for correctness. Today the trainer dashboard needs more: aggregate views across *all* candidates and *all* attempts, summarized by test. The right chart for that is a **multi-series bar chart** — for each test, a "correct" bar and an "incorrect" bar side by side — so a trainer can scan the whole cohort and see which tests are running easy and which are stumping people. Building this is mostly an exercise in shaping the data correctly (the harder part) and configuring recharts to render two series (the easier part). This topic also introduces a small but important architectural move: extracting a **reusable chart wrapper** so the candidate-side chart and the trainer-side chart share styling, sizing, accessibility, and tooltip conventions. PEP ships one wrapper and two charts on top of it.

## Objective

Render multi-series aggregate visualizations for the trainer dashboard, sharing a chart wrapper across slices.

## The Data Shape

The `GET /reports/aggregate` response from D18 looks like:

```json
{
  "by_test": [
    {
      "test_id": "t_001",
      "test_title": "JavaScript Fundamentals",
      "attempts": 23,
      "avg_score": 0.71,
      "correct_count": 187,
      "incorrect_count": 76,
      "avg_seconds_per_question": 42.3
    },
    {
      "test_id": "t_002",
      "test_title": "TypeScript Generics",
      "attempts": 19,
      "avg_score": 0.58,
      "correct_count": 124,
      "incorrect_count": 89,
      "avg_seconds_per_question": 56.1
    }
  ],
  "totals": {
    "candidates": 25,
    "attempts": 42,
    "questions_answered": 476
  }
}
```

For the multi-series bar chart, the data is already in the right shape: one row per test, two numeric columns (`correct_count` and `incorrect_count`), one categorical column (`test_title`). recharts takes that array directly.

## The Reusable Chart Wrapper

Before building the dashboard chart, refactor the existing D17 chart into a wrapper. The wrapper owns the cross-cutting concerns:

- Responsive sizing via `ResponsiveContainer`.
- A consistent height (240px default, 320px for the dashboard).
- Tooltip styling that matches the Tailwind theme.
- Accessible labels — a `title` and `desc` element for screen readers.
- The Tailwind padding/border conventions.

```tsx
// frontend/components/charts/ChartCard.tsx
"use client";

import { ReactNode } from "react";
import { ResponsiveContainer } from "recharts";

type Props = {
  title: string;
  subtitle?: string;
  height?: number;
  children: ReactNode;
};

export function ChartCard({ title, subtitle, height = 240, children }: Props) {
  return (
    <section
      className="rounded-lg border border-slate-200 bg-white p-4 shadow-sm"
      aria-label={title}
    >
      <header className="mb-3">
        <h3 className="text-sm font-semibold text-slate-700">{title}</h3>
        {subtitle && (
          <p className="text-xs text-slate-500">{subtitle}</p>
        )}
      </header>
      <div style={{ width: "100%", height }}>
        <ResponsiveContainer>{children}</ResponsiveContainer>
      </div>
    </section>
  );
}
```

The wrapper accepts any recharts chart as `children`. D17's per-question chart becomes:

```tsx
<ChartCard title="Time per Question" subtitle="Bar color encodes correctness">
  <BarChart data={questions}>{/* ... */}</BarChart>
</ChartCard>
```

And the dashboard chart will use the same `ChartCard` with different children. The cohort gets practice with composition over duplication.

## The Multi-Series Bar Chart

```tsx
// frontend/app/admin/reports/AggregateChart.tsx
"use client";

import {
  Bar,
  BarChart,
  CartesianGrid,
  Legend,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";

import { ChartCard } from "@/components/charts/ChartCard";

type TestRow = {
  test_id: string;
  test_title: string;
  correct_count: number;
  incorrect_count: number;
};

export function AggregateChart({ data }: { data: TestRow[] }) {
  return (
    <ChartCard
      title="Correct vs Incorrect by Test"
      subtitle={`${data.length} test${data.length === 1 ? "" : "s"}`}
      height={320}
    >
      <BarChart data={data} margin={{ top: 8, right: 16, bottom: 8, left: 0 }}>
        <CartesianGrid strokeDasharray="3 3" stroke="#e2e8f0" />
        <XAxis
          dataKey="test_title"
          tick={{ fontSize: 11 }}
          interval={0}
          angle={-20}
          textAnchor="end"
          height={60}
        />
        <YAxis tick={{ fontSize: 11 }} />
        <Tooltip
          contentStyle={{
            backgroundColor: "white",
            border: "1px solid #cbd5e1",
            borderRadius: 6,
            fontSize: 12,
          }}
        />
        <Legend wrapperStyle={{ fontSize: 12 }} />
        <Bar dataKey="correct_count" fill="#22c55e" name="Correct" />
        <Bar dataKey="incorrect_count" fill="#ef4444" name="Incorrect" />
      </BarChart>
    </ChartCard>
  );
}
```

The two `<Bar>` elements with different `dataKey` values is what makes this multi-series — recharts groups them per X-axis category automatically. Color choices: green for correct, red for incorrect, the same palette as D17's chart for consistency across slices.

## Why Two Bars And Not Stacked

A reasonable alternative: stack correct and incorrect into one bar per test, so total height shows total attempts and the green portion shows correct. Stacked bars communicate "what proportion was correct?" well; grouped bars communicate "absolute counts of each" better.

PEP picks **grouped** because the trainer's primary question is "how many candidates got each test right?" — an absolute count. Stacked bars would force the trainer to mentally subtract, which is the kind of cognitive load a chart should *remove*. If a future iteration wants "pass rate per test," add a separate chart (or a normalized stacked chart, where each bar sums to 100%) — don't pile interpretations onto one visualization.

The cohort should be able to articulate this trade-off, not just pick one chart type by reflex.

## Sorting And Ordering

Decide an order. recharts renders the bars in the order they appear in the data array. The backend should return tests sorted *something*. Two reasonable choices:

1. **Alphabetical by `test_title`.** Stable, predictable, but doesn't surface signal.
2. **By attempts descending.** The most-taken tests appear first; the trainer's attention lands on the high-volume ones.

PEP picks #2 because the trainer is hunting for outliers and high-impact tests. Implement the sort on the backend, not the frontend — the frontend should not be re-sorting fetched data, because the same data goes into multiple components (chart + table) and they should agree.

If the trainer wants alphabetical, that's a future column-header click on the table; the chart stays at the most-meaningful default.

## A Second Chart: Avg Seconds Per Question

The aggregate endpoint also returns `avg_seconds_per_question`. That's worth a second chart — *how long does each test take on average?* — which the trainer can read alongside the correct/incorrect chart to spot tests that are slow *and* wrong (probably too hard, redesign) or fast *and* wrong (probably ambiguous wording).

```tsx
<ChartCard title="Avg Seconds per Question" height={240}>
  <BarChart data={data}>
    <CartesianGrid strokeDasharray="3 3" stroke="#e2e8f0" />
    <XAxis dataKey="test_title" tick={{ fontSize: 11 }} interval={0} />
    <YAxis tick={{ fontSize: 11 }} />
    <Tooltip />
    <Bar dataKey="avg_seconds_per_question" fill="#3b82f6" name="Seconds" />
  </BarChart>
</ChartCard>
```

Two charts is plenty for the dashboard. More charts is more cognitive load and less polish per chart. Resist the temptation to add a pie chart, a scatter plot, and a sparkline.

## Composing The Dashboard

```tsx
// frontend/app/admin/reports/AdminReportsView.tsx
import { AggregateChart } from "./AggregateChart";
import { AvgTimeChart } from "./AvgTimeChart";
import { TotalsCard } from "./TotalsCard";
import { AggregateTable } from "./AggregateTable";
import { fetchAggregate } from "@/lib/api/reports";

export async function AdminReportsView({
  searchParams,
}: {
  searchParams: { test_id?: string; from?: string; to?: string };
}) {
  const data = await fetchAggregate(searchParams);

  return (
    <main className="mx-auto max-w-6xl space-y-6 p-6">
      <TotalsCard totals={data.totals} />
      <div className="grid grid-cols-1 gap-4 lg:grid-cols-2">
        <AggregateChart data={data.by_test} />
        <AvgTimeChart data={data.by_test} />
      </div>
      <AggregateTable rows={data.by_test} />
    </main>
  );
}
```

The page is a server component (the dashboard data is fetched server-side, like D17's results page); the charts are client islands wrapped in `<ChartCard>`. Layout uses Tailwind's responsive grid: stacked on small screens, two-column on `lg` and up.

## Color And Accessibility

The same a11y floor from D17 applies: green/red alone is not enough for color-blind users. Recharts shows a `<Legend>` with both color swatch and label, and the tooltip surfaces both values on hover. The chart is also keyboard-accessible via recharts' default tab behavior — each bar group can be focused.

Two additions worth shipping on the polish day:

1. **Pattern fill as a redundant signal.** recharts supports SVG pattern fills; the incorrect bars can be filled with diagonal stripes in addition to red. Color-blind users still distinguish.
2. **A textual summary below the chart.** "JavaScript Fundamentals: 187 correct, 76 incorrect (71%)." Screen readers get the same information as sighted users.

PEP ships the textual summary; the pattern fill is a nice-to-have if time permits.

## Performance With Real Data

The aggregate endpoint returns one row per test. PEP has ~10 tests in seed data, ~25 in a realistic state. recharts handles dozens of bars without pagination. If the test count grows to hundreds, the chart becomes unreadable and the right answer is to filter (Topic 4) rather than render all of them. The chart is a *summary*; once the summary stops fitting on screen, it has stopped being a summary.

For the cohort: if the chart would have more than ~20 bars, that's a UX problem to solve at the filter level, not a recharts performance problem.

## Anti-Patterns

- **Recomputing aggregates on the frontend.** The backend's `GET /reports/aggregate` is the source of truth. The frontend reshapes for display; it does not sum, average, or filter.
- **Hardcoding test titles in the chart.** The chart should work for any data shape the endpoint returns. If a new test is added, no frontend change is needed.
- **Inline-styling colors with raw hex.** Define a color palette once (in `tailwind.config.js` or a `lib/colors.ts`) and reference. PEP's "correct = green-500, incorrect = red-500" should be one constant.
- **Two charts that disagree.** If the table and the chart pull from different fetches, they can show different numbers when the dataset changes. One fetch, two views.
- **Stacking the bars without considering the question.** Stacked is right for proportions, grouped for counts. Pick deliberately.
- **Sorting on the frontend.** The table and the chart end up in different orders; the trainer is confused. Sort on the server.

## Connecting Back To D17

D17's recharts chart was *one series, one slice*. Today's chart is *two series, aggregated across the cohort*. The progression mirrors the slice progression: candidate results (one person) → trainer dashboard (everyone). The chart wrapper is the bridge — both slices use the same `<ChartCard>`, the same colors, the same a11y conventions. Consistency across slices is part of the polish-day work.

## Key Takeaways

- Multi-series bar charts in recharts are two `<Bar>` elements with different `dataKey` values; recharts groups them per X-axis category automatically.
- Extract a reusable `<ChartCard>` wrapper to keep sizing, tooltips, and a11y consistent across slices.
- Grouped bars show absolute counts; stacked bars show proportions. Pick based on the question the chart answers.
- Sort, filter, and aggregate on the backend; the frontend reshapes for display only.
- Two charts max per dashboard; resist the urge to add more. Each additional chart adds cognitive load and reduces polish per chart.
- Color is necessary but not sufficient; pair it with a legend, tooltip, and ideally a textual summary for accessibility.

---
*Prerequisites: [04-data-visualization-with-charting-libraries-recharts.md](../day-17/04-data-visualization-with-charting-libraries-recharts.md), [01-server-components-for-data-heavy-pages.md](../day-17/01-server-components-for-data-heavy-pages.md), [03-multi-entity-reporting-query-patterns.md](../day-18/03-multi-entity-reporting-query-patterns.md).*
