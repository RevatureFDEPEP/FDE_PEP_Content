# Responsive Layout Patterns With Tailwind

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Candidates check their results on whatever device is in their hand — a 14" laptop in class, a phone on the bus, occasionally a 27" external monitor at home. The same page has to work at all three widths. D13's page-layout topic covered the test-taking page with a narrow `max-w-3xl` constraint because it had a single focal element (the question). The results page is different: three cards' worth of content that benefit from horizontal space when there's space and *must* stack when there isn't. Topic 6 covers Tailwind's responsive utility model, the grid composition that handles the results layout cleanly, and the specific decisions for the score card / chart / table arrangement.

## Tailwind's Mobile-First Model

Tailwind's responsive utilities are **min-width breakpoints**. Unprefixed classes apply at all sizes; prefixed classes (`sm:`, `md:`, `lg:`, `xl:`, `2xl:`) apply *at and above* that breakpoint. The default breakpoint values:

| Prefix | Min width | Typical device |
|---|---|---|
| (none) | 0 | All — write mobile styles here |
| `sm:` | 640px | Large phone in landscape, small tablet |
| `md:` | 768px | Tablet, narrow laptop |
| `lg:` | 1024px | Laptop |
| `xl:` | 1280px | Desktop |
| `2xl:` | 1536px | Wide desktop |

The mental model: **start by writing the phone layout, then add `md:` / `lg:` overrides that expand it.** Not the other way around. Writing "desktop layout then patch for mobile" leads to broken phone experiences because the override goes the wrong direction (you can't easily "un-grid" at small sizes).

For PEP, the breakpoints that matter are `md:` (768px — phone vs. everything-else) and occasionally `lg:` (1024px — laptop vs. narrow laptop). We mostly don't use `sm:`, `xl:`, or `2xl:` — the design doesn't need that many control points.

## The Results Page Grid

The three cards on the results page (score, chart, details table) want different widths at different breakpoints:

- **Phone (< 768px):** All three cards stack vertically, full width. One column.
- **Tablet / laptop (≥ 768px):** Score (1/3 width) next to chart (2/3 width) on the top row; table full-width below.

This is a textbook use of CSS Grid with named column spans. Tailwind's grid utilities express it cleanly:

```tsx
<div className="grid grid-cols-1 gap-6 md:grid-cols-3">
  <Card className="md:col-span-1">{/* Score */}</Card>
  <Card className="md:col-span-2">{/* Chart */}</Card>
  <Card className="md:col-span-3">{/* Table */}</Card>
</div>
```

Reading it left-to-right:

- `grid` — display: grid.
- `grid-cols-1` — at the default (phone) breakpoint, one column. Each child takes the full row.
- `gap-6` — 1.5rem (24px) gap between grid items, both row and column directions.
- `md:grid-cols-3` — at ≥768px, switch to three columns.
- `md:col-span-1`, `md:col-span-2`, `md:col-span-3` — at ≥768px, each child claims that many of the three columns.

At phone width, `grid-cols-1` means there's only one column to span, so the `col-span-N` classes are no-ops; each card is full-width and the cards stack top-to-bottom in source order. At desktop width, the grid is three columns; score takes one, chart takes two, the row fills, and the table card wraps to the next row spanning all three.

That's the entire responsive layout. No media queries written by hand, no `display: none` on duplicated markup, no conditional rendering based on `window.innerWidth`.

## The Page Container

Wrapping the grid is the page's outer container, which constrains the max width and adds horizontal padding:

```tsx
<main className="mx-auto w-full max-w-5xl px-4 py-6 md:px-6">
  <h1 className="text-2xl font-semibold tracking-tight">Your Results</h1>
  <div className="mt-6 grid grid-cols-1 gap-6 md:grid-cols-3">
    {/* cards */}
  </div>
</main>
```

The container utilities:

- `mx-auto` — center horizontally.
- `w-full` — fill the parent width up to the max.
- `max-w-5xl` — 64rem (1024px). Wider than `max-w-3xl` (which D13 used for the test runner) because the results page benefits from horizontal space; the chart wants room to breathe.
- `px-4 py-6 md:px-6` — 1rem horizontal padding on phone (tight; assume thumb-width margins), 1.5rem on tablet+. Vertical padding doesn't need a breakpoint change.

The `max-w-5xl` is the size lever the cohort might want to tune. Too narrow and the chart feels cramped on a wide desktop; too wide and line lengths inside the table become uncomfortable to scan. 1024px is the right default for this content.

## Inside-Card Responsive Details

The cards themselves mostly don't need responsive utilities — shadcn's `Card` defaults handle padding consistently. A few internal pieces do:

**The headline score card.** The big number is `text-4xl` (2.25rem) on phone — already large enough. No `md:text-5xl` upgrade; the number being readable on phone is the priority.

**The table.** On phone, table column widths auto-fit, but the rightmost "Time" column tends to crowd. Fix it with right-alignment on the column:

```tsx
<TableHead scope="col" className="text-right">Time</TableHead>
<TableCell className="text-right">{formatDuration(q.elapsed_seconds)}</TableCell>
```

And let the table handle horizontal overflow if a future column makes it too wide:

```tsx
<div className="overflow-x-auto">
  <Table>{/* ... */}</Table>
</div>
```

The horizontal scroll is an acceptable fallback for tables on phone — better than truncating data or making columns illegibly narrow.

**The chart.** `<ResponsiveContainer width="100%" height={280}>` from Topic 4 already handles width. The fixed height of 280px is the right call at both breakpoints (Topic 4 argued this).

## Container Queries: Not Today

Tailwind v4 supports container queries (`@container` + `@md:`), which size to a parent's width rather than the viewport. They'd let us, e.g., make the chart card adapt based on its own width independent of the screen. PEP doesn't need container queries — the viewport-based breakpoint model handles the layouts we have. Flag them as a tool to know about for future complexity, but don't reach for them today.

## Touch Targets And Hit Areas

A responsive layout isn't only about visual breakpoints. Interactive elements need to be large enough to tap on a phone:

- Buttons (shadcn defaults) are ~36px tall. Above the 24px minimum, below the 44px Apple HIG recommendation. Fine for PEP but worth noting.
- Tap targets should have visible focus states (Topic 7 elaborates).
- Hover states (`hover:bg-muted` on table rows) don't trigger on touch devices; don't rely on hover to convey information.

The results page is mostly read-only — only the chart tooltip (hover) and the retry button (in error states) are interactive. The hover-only chart tooltip is a real limitation on touch; users tap a bar and see the tooltip until they tap elsewhere. Recharts handles tap-to-show on mobile by default; verify in the Playwright mobile profile or with Chrome DevTools' device emulation.

## Testing Across Breakpoints

The D15 Playwright suite can verify layout at multiple viewports:

```ts
// e2e/results.responsive.spec.ts
import { test, expect, devices } from "@playwright/test";

test.describe("results page — mobile", () => {
  test.use({ ...devices["iPhone 13"] });
  test("cards stack vertically", async ({ page }) => {
    await page.goto("/results/sess_abc123");
    const cards = await page.locator('[class*="md:col-span"]').all();
    // All cards full width on mobile — bounding boxes overlap vertically.
    const widths = await Promise.all(cards.map(c => c.boundingBox().then(b => b!.width)));
    expect(new Set(widths).size).toBe(1); // all the same width
  });
});

test.describe("results page — desktop", () => {
  test.use({ viewport: { width: 1280, height: 800 } });
  test("score and chart side-by-side", async ({ page }) => {
    await page.goto("/results/sess_abc123");
    const score = await page.getByText("Your score").locator("..").boundingBox();
    const chart = await page.getByText("Per-question time").locator("..").boundingBox();
    expect(score!.y).toBeCloseTo(chart!.y, -1); // same row
    expect(score!.x).toBeLessThan(chart!.x);    // score to the left
  });
});
```

The mobile assertion checks that cards share the same width (all full-width because they're stacked). The desktop assertion checks that score and chart cards share the same top edge (side-by-side). Smoke tests, not pixel-perfect, but they catch regressions.

## Anti-Patterns

- **Desktop-first overrides.** Writing `flex-row md:flex-col` (the wrong direction for Tailwind's mobile-first model) inverts the intent and confuses the next reader.
- **Hidden mobile content.** `<div className="hidden md:block">` to hide content from phone users is sometimes necessary but is usually a sign the content should be restructured to fit. Try the layout change first; hide only as a last resort.
- **`window.innerWidth` checks in React.** Tailwind responsive classes are CSS, run by the browser, work without JavaScript, render correctly during SSR. JavaScript-driven responsive layout is slower, breaks SSR (hydration mismatch), and fights Tailwind. Don't.
- **Skipping `gap-N` in grid containers.** Without `gap-6`, cards touch each other and look like one big slab. Always set a gap on grid/flex containers.
- **`max-w` everywhere.** The page container has `max-w-5xl`; child cards inherit width from the grid columns. Don't add `max-w` to each card — they'll fight the grid sizing.
- **Magic numbers via arbitrary values.** `w-[842px]` works but doesn't scale. Use Tailwind's spacing scale (`w-3xl`, `max-w-5xl`) so the design system stays coherent.
- **Letting tables overflow without `overflow-x-auto`.** A 600px-wide table on a 360px phone breaks the page layout entirely. Wrap tables in an overflow container.
- **Tiny tap targets.** Buttons smaller than ~36px (Tailwind `h-9`) are hard to tap. shadcn defaults are at this floor; don't downsize them.

## Key Takeaways

- Tailwind is mobile-first: unprefixed classes are the phone layout, `md:` adds at ≥768px, `lg:` at ≥1024px.
- The results page uses a `grid grid-cols-1 md:grid-cols-3` with explicit `md:col-span-1 / 2 / 3` on cards. One layout declaration handles all breakpoints.
- `max-w-5xl` (1024px) is the right container width for a data-heavy page; `max-w-3xl` (from D13) is for narrow focus pages.
- Wrap tables in `overflow-x-auto` to handle phone-width crowding; right-align numeric columns.
- Don't reach for `window.innerWidth` or container queries; CSS breakpoints handle this layout cleanly.

---
*Prerequisites: day-13-page-layout-patterns-with-tailwind, day-17-detail-view-ui-patterns-for-structured-data.*
