# Detail-View UI Patterns For Structured Data

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

A detail view is a page dedicated to one entity — one user's results, one quiz attempt — and it has a different rhythm than a list view or a form. The user landed here to *understand* something. The layout's job is to walk their eye from the headline (the number that summarizes everything) through the supporting evidence (the per-item breakdown) to the visualization (the shape of the data at a glance). The results page is a textbook detail view, and there's a well-worn pattern: **header summary → tabular breakdown → chart**, composed from a small set of reusable primitives. Topic 5 names that pattern, picks the shadcn components that implement it, and gives the cohort a structure they can replicate on D19's trainer dashboard.

## The Three-Region Pattern

A detail view of aggregated data, almost universally, has three regions:

1. **Headline.** One or two numbers that answer "the question the page exists to answer." For the results page: the score (8/10) and the time taken (12m 03s). The candidate's first second on the page should land on this.
2. **Breakdown.** The structured-data evidence behind the headline. For the results page: a table with one row per question, columns for question number, result, time. Anything the candidate might want to scrutinize lives here.
3. **Visualization.** The same data, re-presented graphically. Useful for pattern-spotting ("I spent way too long on Q4") in a way the table makes harder.

These three regions exist in roughly that priority order. On a 1024px desktop they can sit side-by-side or stack; on a 360px phone they always stack vertically, with the headline first. Topic 6 covers the responsive details; Topic 5 is the composition.

A fourth region — *related actions* or *navigation back to context* — sometimes appears as a footer (e.g., "back to my dashboard" or "view full attempt history"). For the results page, the global header has the dashboard link, so we don't duplicate.

## The shadcn Primitives Available

The brownfield frontend ships with shadcn/ui components in `components/ui/`. The pieces you compose with:

- **`Card`, `CardHeader`, `CardTitle`, `CardDescription`, `CardContent`** — the container primitive. One card per logical region.
- **`Table`, `TableHeader`, `TableBody`, `TableRow`, `TableHead`, `TableCell`** — semantic-HTML wrapper components that render `<table>` and friends with consistent spacing. Topic 7 covers why `<table>` matters for accessibility.
- **`Separator`** — a horizontal rule with the right opacity and spacing tokens. Use sparingly.
- **`Badge`** — small inline pill for status (Correct/Incorrect). Useful in the table.

Don't reach for a new component library today. If something needed isn't in `components/ui/`, copy-paste a new shadcn primitive following the existing patterns; don't introduce a competing system.

## The Full Composition

```tsx
// frontend/app/results/[sessionId]/ResultsBreakdown.tsx
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { ResultsChart } from "./ResultsChart";
import { fetchReportForSession } from "./lib/fetchReport";

type Props = { sessionId: string };

export async function ResultsBreakdown({ sessionId }: Props) {
  const { attempt } = await fetchReportForSession(sessionId);

  return (
    <div className="grid grid-cols-1 gap-6 md:grid-cols-3">
      {/* Region 1: Headline */}
      <Card className="md:col-span-1">
        <CardHeader>
          <CardDescription>Your score</CardDescription>
          <CardTitle className="text-4xl font-semibold">
            {attempt.score}
            <span className="text-base font-normal text-muted-foreground">
              {" "}
              / {attempt.max_score}
            </span>
          </CardTitle>
        </CardHeader>
        <CardContent>
          <dl className="space-y-2 text-sm">
            <div className="flex justify-between">
              <dt className="text-muted-foreground">Test</dt>
              <dd className="font-medium">{attempt.test_name}</dd>
            </div>
            <div className="flex justify-between">
              <dt className="text-muted-foreground">Time taken</dt>
              <dd className="font-medium">{formatDuration(attempt.elapsed_seconds)}</dd>
            </div>
            <div className="flex justify-between">
              <dt className="text-muted-foreground">Submitted</dt>
              <dd className="font-medium">{formatDate(attempt.submitted_at)}</dd>
            </div>
          </dl>
        </CardContent>
      </Card>

      {/* Region 3: Visualization (visually right of headline on desktop) */}
      <Card className="md:col-span-2">
        <CardHeader>
          <CardTitle>Per-question time</CardTitle>
          <CardDescription>
            Bar color shows correctness; bar height shows seconds spent.
          </CardDescription>
        </CardHeader>
        <CardContent>
          <ResultsChart questions={attempt.per_question} />
        </CardContent>
      </Card>

      {/* Region 2: Breakdown (full width below) */}
      <Card className="md:col-span-3">
        <CardHeader>
          <CardTitle>Question-by-question</CardTitle>
        </CardHeader>
        <CardContent>
          <Table>
            <caption className="sr-only">
              Detailed per-question result, time spent, and correctness for this attempt.
            </caption>
            <TableHeader>
              <TableRow>
                <TableHead scope="col">#</TableHead>
                <TableHead scope="col">Result</TableHead>
                <TableHead scope="col" className="text-right">Time</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {attempt.per_question.map((q, i) => (
                <TableRow key={q.question_id}>
                  <TableCell className="font-medium">Q{i + 1}</TableCell>
                  <TableCell>
                    <Badge variant={q.correct ? "default" : "destructive"}>
                      {q.correct ? "Correct" : "Incorrect"}
                    </Badge>
                  </TableCell>
                  <TableCell className="text-right">
                    {formatDuration(q.elapsed_seconds)}
                  </TableCell>
                </TableRow>
              ))}
            </TableBody>
          </Table>
        </CardContent>
      </Card>
    </div>
  );
}

function formatDuration(seconds: number): string {
  const m = Math.floor(seconds / 60);
  const s = seconds % 60;
  return m > 0 ? `${m}m ${s}s` : `${s}s`;
}

function formatDate(iso: string): string {
  return new Date(iso).toLocaleString(undefined, {
    dateStyle: "medium",
    timeStyle: "short",
  });
}
```

A few choices to slow down on:

- **The headline card uses `<dl>` for the metadata.** Test name, time taken, submitted-at are a list of label/value pairs — that's exactly what description lists exist for. More semantic than nested divs; better for screen readers (Topic 7).
- **`CardDescription` is the small text above the big number.** That inverts the typical "title then description" pattern, but for headline cards (where the number *is* the point) it reads better. The label is the description; the number is the title.
- **Visualization on the right at desktop, below headline on phone.** The grid columns (`md:col-span-1` and `md:col-span-2`) put the chart next to the score on wide viewports. On phone, both stack full-width. Topic 6 covers the grid math.
- **Table spans all three columns.** It's tabular data; it wants horizontal space. Forcing it into one-third of the page would crush column widths.
- **`<caption>` inside the `<Table>`.** Sighted users see the surrounding `CardTitle` ("Question-by-question"); screen readers benefit from a table-level caption that describes what's in the rows. `sr-only` keeps it out of the visual layout.
- **`<Badge>` for Correct/Incorrect.** Two-character labels in a colored pill are scannable. Don't use a green checkmark icon by itself — icon-only is hard for screen readers and ambiguous if the colors don't render.

## Density And Whitespace

Detail views feel professional when:

- The headline number has *room around it* — at least one card padding on every side, no other UI competing for attention nearby.
- The table rows have consistent row height (~40px at default `Table` styling) and don't get crowded with extra controls.
- Cards have visible borders or backgrounds (shadcn's default) so the regions feel like distinct objects, not a single wall of text.

The Tailwind defaults plus shadcn's component padding generally get this right. Resist the urge to override `padding` everywhere — the design system gets the spacing scale correct most of the time.

## Variability In The Data

Real attempts vary. The component has to handle:

- **All 10 questions answered** (the common case).
- **Fewer than 10 questions** (some tests are 5 questions). Table just has fewer rows; chart has fewer bars. Nothing else changes.
- **A question with `elapsed_seconds: 0`** (the candidate moved past without engaging). Bar has zero height — recharts handles this gracefully. Table cell shows "0s". No special case.
- **Very long question text** in a future iteration. Right now the table doesn't show the question stem — just the index. If D19 adds a "review your answers" expansion, that's a different detail-view pattern (master-detail or expand-in-place), and Topic 5 stays out of it for now.

The point: a good detail-view structure doesn't break when the data shape varies within its declared range. If the data shape *changes* (e.g., a new column), revisit the layout intentionally rather than papering over with `overflow: hidden`.

## Reusing The Pattern On D19

The trainer dashboard on D19 also wants a detail view — per-test aggregate, per-candidate rows. The same three-region pattern applies: headline (test name, pass rate), breakdown (one row per candidate), visualization (distribution of scores). The shadcn primitives are identical; the data shape is different. Naming the pattern explicitly today makes that reuse a one-paragraph thing on D19, not a redesign.

## Anti-Patterns

- **Stacked `<div>`s for tabular data.** Loses semantic meaning, breaks keyboard navigation in some browsers, makes the screen-reader announcement nonsensical. `<table>` for rows-and-columns data, always.
- **Buried headline.** Putting the most important number — the score — below the chart or below the table is structurally wrong. First scroll, top-left card.
- **Cards within cards within cards.** Nesting Card primitives reads as visual noise. One card per region; if you need sub-sections inside a card, use a `Separator` and a heading, not a nested Card.
- **Mixing list and detail in one page.** "Here's the candidate's last attempt, and a list of all their other attempts" is two pages. Detail views are about one thing.
- **Reskinning the table cells with `<div>` and `display: grid` for layout flexibility.** You think you're getting flexibility; you're getting a broken accessibility tree. Stay with `<table>`.
- **Hardcoded English text in the table headers but data-driven labels in the chart.** Mix one or the other. For PEP, all UI strings are English; defer i18n to Phase 2 but flag the seam.
- **No semantic association between chart and its description.** `<figure>` + `<figcaption>` wraps the chart; `aria-label` on the figure repeats the key info. Topic 7 reinforces.

## Key Takeaways

- The detail-view pattern is **headline → breakdown table → visualization**, composed from shadcn `Card`, `Table`, and `Badge` primitives.
- The headline card gets the score-as-Title; supporting metadata lives in a `<dl>` description list inside the card body.
- Tabular data uses `<table>` semantics (via shadcn `Table`) with a `sr-only` `<caption>`; never use stacked `<div>`s.
- The pattern is reusable: D19's trainer dashboard reuses the same three-region structure with different data.
- Resist over-customization — shadcn defaults and Tailwind spacing tokens get density and whitespace right; only override when there's a specific reason.

---
*Prerequisites: [07-page-layout-patterns-with-tailwind.md](../day-13/07-page-layout-patterns-with-tailwind.md), [01-server-components-for-data-heavy-pages.md](01-server-components-for-data-heavy-pages.md), [04-data-visualization-with-charting-libraries-recharts.md](04-data-visualization-with-charting-libraries-recharts.md).*
