# Data Visualization With Charting Libraries (recharts)

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

The results page needs one chart: a visualization of per-question correctness or time-taken across the attempt. A 10-question quiz with bullet-point colors is more legible than a table when you're scanning for "which questions did I miss"; that's the user value. The technical choice is **recharts** — a React-native charting library built on D3 primitives, with a declarative component API that fits server-rendered React naturally. The library is one new dependency, and the cohort writes one new client component. Topic 4 covers picking the right chart type for the data, wiring recharts into the App Router model, sizing it responsively, and making it accessible enough to clear Topic 7's a11y floor.

## Why recharts, And Why One Chart

The brownfield frontend doesn't yet have a charting library. Several reasonable choices exist:

- **recharts** — React components, declarative, healthy ecosystem, ~95KB gzipped, easy to style with the existing Tailwind tokens.
- **visx** — Airbnb's lower-level D3 wrapper; more flexible, more boilerplate per chart.
- **Chart.js + react-chartjs-2** — Canvas-based, performant on large datasets, less Reactish API.
- **Plotly / Highcharts** — heavy, license-encumbered, overkill.

PEP's chart needs are small: one bar chart today, possibly a second on D19's trainer dashboard. recharts is the easiest path to "render a meaningful chart in 30 lines of JSX" and the easiest to read in a code review. Pin to a specific version; lock the dep before D17 starts so 25 candidates aren't all hitting npm at once.

```bash
cd frontend && npm install recharts
```

Update `package.json` and commit the lockfile alongside the chart component.

## Picking The Chart Type For The Data

The data is `per_question: [{ question_id, correct, elapsed_seconds }, ...]`. Two reasonable charts:

1. **Bar chart of correctness.** X-axis = question number, Y-axis = 0 or 1 (correct), bar color encodes pass/fail. Easy to scan.
2. **Bar chart of time-taken.** X-axis = question number, Y-axis = elapsed seconds, color encodes correctness. Dense — communicates "you spent 90 seconds and still got it wrong on Q4."

Option 2 is the better information-per-pixel choice. The chart shows *both* dimensions (time and correctness) instead of one. The cohort builds option 2; option 1 is the MVF fallback if recharts integration takes too long.

A line chart would be wrong: question order isn't continuous; there's no trend to follow. Pie charts are wrong for >3 categories. Heatmaps are overkill for 10 data points. Bar chart is the right call here.

## The Client Component

recharts measures the DOM to size the chart. That means it must run in the browser, which means `"use client"`. It's the only client island on the results page (Topic 1 is emphatic on this point).

```tsx
// frontend/app/results/[sessionId]/ResultsChart.tsx
"use client";

import {
  Bar,
  BarChart,
  CartesianGrid,
  Cell,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from "recharts";

type Question = {
  question_id: string;
  correct: boolean;
  elapsed_seconds: number;
};

type Props = { questions: Question[] };

const CORRECT_FILL = "var(--chart-correct, #16a34a)"; // tailwind green-600
const INCORRECT_FILL = "var(--chart-incorrect, #dc2626)"; // tailwind red-600

export function ResultsChart({ questions }: Props) {
  const data = questions.map((q, i) => ({
    label: `Q${i + 1}`,
    seconds: q.elapsed_seconds,
    correct: q.correct,
    questionId: q.question_id,
  }));

  return (
    <figure
      className="w-full"
      aria-label="Time spent on each question, colored by correctness"
    >
      <ResponsiveContainer width="100%" height={280}>
        <BarChart data={data} margin={{ top: 8, right: 16, bottom: 24, left: 8 }}>
          <CartesianGrid strokeDasharray="3 3" stroke="var(--border)" />
          <XAxis
            dataKey="label"
            stroke="var(--foreground)"
            label={{ value: "Question", position: "insideBottom", offset: -8 }}
          />
          <YAxis
            stroke="var(--foreground)"
            label={{
              value: "Seconds",
              angle: -90,
              position: "insideLeft",
              style: { textAnchor: "middle" },
            }}
          />
          <Tooltip
            cursor={{ fill: "var(--muted)" }}
            contentStyle={{
              background: "var(--background)",
              border: "1px solid var(--border)",
              borderRadius: 6,
            }}
            formatter={(value: number, _name, ctx) => {
              const correct = (ctx.payload as { correct: boolean }).correct;
              return [`${value}s — ${correct ? "Correct" : "Incorrect"}`, ""];
            }}
          />
          <Bar dataKey="seconds" radius={[4, 4, 0, 0]}>
            {data.map((d, i) => (
              <Cell
                key={d.questionId}
                fill={d.correct ? CORRECT_FILL : INCORRECT_FILL}
              />
            ))}
          </Bar>
        </BarChart>
      </ResponsiveContainer>
      <figcaption className="sr-only">
        Bar chart with one bar per question. Green bars indicate correct
        answers, red bars indicate incorrect. Bar height is seconds spent.
      </figcaption>
    </figure>
  );
}
```

Specific decisions to call out:

- **`ResponsiveContainer`** is what makes the chart fit its parent. Set `width="100%"` and a fixed `height` (or `aspect`). It uses `ResizeObserver` under the hood — works on initial render and on viewport resize without manual wiring.
- **`<Cell />` for per-bar coloring.** recharts' `<Bar>` takes a single `fill` by default; the `<Cell>` children override per-row so each bar's color encodes its `correct` boolean. This is the idiomatic way to do conditional coloring.
- **CSS variables for colors.** `var(--chart-correct, #16a34a)` lets the design system override the chart palette without editing the chart component. Fallbacks are explicit hex so the chart renders even if the variable isn't defined yet.
- **Tooltip text includes the correctness verbally**, not just visually. A user with red-green color blindness still needs to know which bars are correct; the tooltip is one of the disambiguators (Topic 7 covers more).
- **`<figure>` + `<figcaption>`** is the semantic wrapper for a chart. `aria-label` on the figure is the short description; the `sr-only` figcaption is the long-form description for assistive tech. This is the Topic 7 a11y floor.

## Sizing And Responsiveness

The chart is inside a Card in the `md:col-span-2` region of the grid. On mobile (single column), the card is full-width — the chart is ~330px wide at typical phone widths. On desktop, it's roughly two-thirds of a 1024px page — ~640px wide.

`ResponsiveContainer` handles the width. The height is fixed at 280px, which is the right target for 10 bars on phone *and* desktop. Two notes:

- **Don't make the height responsive too.** A chart that gets shorter on mobile becomes unreadable; one that gets taller wastes scroll. A fixed 280px works at both breakpoints.
- **Watch for SSR width-of-zero issues.** `ResponsiveContainer` measures the parent's width *on the client*. During the first render the width is unknown; recharts renders nothing until it measures. The skeleton from Topic 2 covers the gap.

## Handling Edge Cases

- **No questions** (`questions.length === 0`). The chart shouldn't render at all. Guard with a check and render a "no questions to display" message instead.
- **One question.** A bar chart with one bar looks weird but works; don't special-case below 5.
- **All correct** or **all incorrect**. Single-color chart. Legend still important so the candidate knows what color means what. Add a legend or rely on the tooltip; for PEP we use the tooltip + figcaption.
- **Very long elapsed times** (someone walked away for 20 minutes). Cap the Y-axis with `domain={[0, 600]}` or let recharts auto-scale; both have trade-offs. Auto-scale is the default and probably right for PEP — a 20-minute bar makes the outlier visible, which is information.

## Performance

10 bars is nothing. 100 bars is nothing for recharts. 10,000 bars would be a problem — switch to a canvas-based library at that point. PEP never crosses that threshold, so don't preemptively optimize.

Recharts re-renders the SVG on every prop change. For the results page that's fine — the data is loaded once and doesn't update. If a future iteration adds live updates (it shouldn't for results), wrap the chart in `React.memo` to skip re-renders when props are referentially equal.

## Integrating With Topics 2 And 3

- **Suspense (Topic 2).** The chart is inside `ResultsBreakdown`, which suspends on the data fetch. The skeleton shows a placeholder block (Topic 2 sized it 64-rem tall to match the chart's height). When the data arrives, the real chart renders.
- **Error boundary (Topic 3).** recharts can throw if the data shape is unexpected (e.g., `NaN` in `elapsed_seconds`). Wrap the chart in `SectionErrorBoundary` so the rest of the page (the score summary, the table) still renders even if the chart bombs.

The chart isn't load-bearing for the candidate's understanding of their score — the table has the same data in tabular form. That's by design: the chart is a visualization layered *on top of* an accessible data structure, not the primary representation.

## Anti-Patterns

- **Putting recharts in a server component.** The library needs the DOM. `"use client"` directive at the top of the chart file, always.
- **Hardcoded width/height in pixels without `ResponsiveContainer`.** Chart overflows on mobile, leaves whitespace on desktop. Use `ResponsiveContainer` width=100%.
- **Color-only encoding.** Red bars and green bars look identical to red-green colorblind users (~8% of men). Pair color with shape, label, or tooltip text. Topic 7 elaborates.
- **No legend or label.** A bar chart with bars but no axis labels is a Rorschach test. Always label both axes.
- **`<div>` wrapper instead of `<figure>`.** Loses the semantic association between the chart and its caption.
- **`alt` text on a chart `<figure>`.** `alt` is for `<img>`. Use `aria-label` on the figure plus `<figcaption>` for the text alternative.
- **Letting recharts inherit colors from a global stylesheet that changes underneath you.** Lock chart colors via CSS variables or constants in the component file. The chart's appearance should not depend on an unrelated stylesheet edit.

## Key Takeaways

- recharts is the chart library; one bar chart of time-taken-per-question, colored by correctness, is the deliverable.
- The chart must be `"use client"` — recharts measures the DOM. Wrap it in `<ResponsiveContainer width="100%" height={280}>` for sizing.
- Use `<Cell>` for per-bar conditional coloring; reference CSS variables for theme override.
- Wrap the chart in `<figure>` + `<figcaption>`, with `aria-label` for short description and a screen-reader-only long-form caption.
- Color is not enough — pair it with text (in tooltips, in the caption, in the table beside the chart) for accessibility and colorblind users.

---
*Prerequisites: day-13-client-components-for-stateful-interactivity, day-17-server-components-for-data-heavy-pages, day-17-error-boundary-patterns.*
