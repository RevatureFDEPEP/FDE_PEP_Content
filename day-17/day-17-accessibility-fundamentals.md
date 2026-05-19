# Accessibility Fundamentals (Semantic HTML, ARIA, Keyboard Navigation)

> *Day 17: Results Slice (Frontend) — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Accessibility — usually abbreviated **a11y** — is the practice of building UIs that work for users with vision, motor, cognitive, or hearing differences, and incidentally for everyone else (the keyboard power-user, the screen-magnifier user, the user with a wonky trackpad). The results page is a good place to introduce the practice in PEP because it has the four patterns most often gotten wrong: a data table, a chart, interactive controls, and a fetched-data loading state. Topic 7 covers the **a11y floor** — what every page in the substrate should clear without exception — and points out where this page can go further. Treat it as orientation depth, not exhaustive coverage; the goal is a candidate who knows what to check and which checks they're not doing yet.

## The Floor For This Page

A working definition of "minimum acceptable a11y" for the results page:

1. **Semantic HTML by default.** `<table>` for the per-question data, `<h1>` for the page title, `<main>` around the content, `<button>` (not `<div onClick>`) for actions.
2. **Skip-to-content link** at the top of the layout so keyboard users can bypass the global header.
3. **Visible focus rings** on every interactive element. Never set `outline: none` without an equivalent replacement.
4. **Labels on chart axes and a text alternative for the chart itself.**
5. **`role="status"` on loading skeletons, `role="alert"` on error UIs**, with appropriate live-region politeness.
6. **Color is paired with text.** No information conveyed by color alone.
7. **Keyboard reachability for every interactive control.** Tab through the page; everything reaches focus in a sensible order; Enter/Space activates buttons; Esc dismisses tooltips/popovers.

These are the targets for every page from D17 onward. Most are one-line implementations; a few (the chart text alternative, focus management on the retry button) need a paragraph of thought.

## Semantic HTML Is Most Of The Battle

The single highest-leverage a11y move is **using the right HTML element**. Browsers and assistive tech ship with deep knowledge about what `<table>`, `<button>`, `<nav>`, `<main>`, `<h1>` mean. Using them gets you free keyboard semantics, free screen-reader announcements, free document structure.

For the results page:

- **`<main id="main">`** wraps the page content. Screen readers offer a "jump to main content" landmark; sighted keyboard users use the skip link to land here.
- **`<h1>Your Results</h1>`** is the page title. Exactly one `<h1>` per page. Subsequent section titles are `<h2>` (the CardTitles render `<h3>` by default in shadcn — that's fine; the level isn't load-bearing as long as nesting is consistent).
- **`<table>`** (rendered by shadcn's `Table` primitive) for the per-question data. Crucial: NVDA and VoiceOver have a "table navigation" mode that announces row/column headers as the user arrows around cells. Stacked `<div>`s lose this entirely.
- **`<button type="button">`** for the retry button in the error boundary. Default `<button>` inside a `<form>` is `type="submit"` — set the type explicitly to avoid surprises.
- **`<figure>` + `<figcaption>`** wrapping the chart (Topic 4 already covered this).
- **`<dl>` / `<dt>` / `<dd>`** for the metadata key/value pairs in the headline card (Topic 5).

If a candidate finds themselves writing `<div role="button" onClick={...} tabIndex={0}>`, the question to ask is: *why isn't this a `<button>`?* Almost always the answer is "no reason"; convert it.

## The Skip Link

Keyboard users navigate via Tab. On a page with a global header containing N navigation links, every page entry requires N tabs before reaching the content. A **skip link** is the standard mitigation: a focusable link that's visually hidden until focused, sitting as the first focusable element.

```tsx
// frontend/components/SkipLink.tsx
export function SkipLink() {
  return (
    <a
      href="#main"
      className="
        sr-only
        focus:not-sr-only
        focus:fixed focus:top-2 focus:left-2 focus:z-50
        focus:rounded focus:bg-background focus:px-4 focus:py-2
        focus:text-sm focus:font-medium focus:shadow
        focus:ring-2 focus:ring-primary
      "
    >
      Skip to main content
    </a>
  );
}
```

Place it as the first child of the root layout:

```tsx
// frontend/app/layout.tsx
<body>
  <SkipLink />
  <SiteHeader />
  <main id="main">{children}</main>
</body>
```

`sr-only` hides the link from sighted users; `focus:not-sr-only` reveals it when keyboard-focused. The result: a sighted user never sees the skip link unless they tab into it; a keyboard user sees it as the first focusable element on every page.

This is a one-time addition to the layout. After D17, every page benefits.

## Focus Management

Tailwind's `focus-visible:` utilities apply only when the browser determines the focus arrived via keyboard, not from a mouse click. Use `focus-visible:` for visible rings:

```tsx
// in shadcn's button base class
"focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2"
```

shadcn's Button primitive already includes these. Don't override them to nothing. If a designer asks for "no ugly focus rings," the answer is "we can customize the ring color, not remove it."

Specific focus rules for the results page:

- **After clicking "Try again"** in the error boundary, focus should remain on the retry button or move to the new heading. The default (focus stays on the button, content re-renders around it) is fine here because the user is still in the same conceptual place.
- **After the data finishes loading**, focus does *not* need to jump anywhere — the user wasn't focused on anything specific during the skeleton. Don't auto-focus content.
- **Inside the chart tooltip**, don't trap focus. The tooltip is hover/focus-revealed; it dismisses on blur. Recharts' default behavior is correct.

## ARIA, Used Sparingly

The first rule of ARIA is: **don't use ARIA if a semantic element does the job.** Adding `role="button"` to a `<div>` is worse than just using `<button>`; `aria-label` on a button that already has visible text is redundant.

That said, a few ARIA attributes are load-bearing on this page:

- **`role="status"` + `aria-live="polite"`** on the loading skeleton (Topic 2). Announces "Loading your results" to screen readers when the skeleton appears.
- **`role="alert"`** on the error boundary fallback (Topic 3). Screen readers announce alert-role content immediately upon insertion, interrupting whatever they were saying — the right behavior for "something failed."
- **`aria-label`** on the chart's `<figure>` (Topic 4). Provides a short text equivalent of the chart's content.
- **`scope="col"`** on table header cells. Tells screen readers that these are column headers (vs. row headers).
- **`aria-busy="true"`** can be set on a container while its data is loading — useful in some patterns, but redundant if you already have `role="status"` plus the skeleton inside the Suspense boundary. Skip for now.

That's it. Five attributes total across the page. ARIA is for the cases semantic HTML can't cover.

## Chart Accessibility

Charts are the hardest thing on this page to make accessible because their information is geometric, not textual. The minimum-acceptable approach from Topic 4 is:

```tsx
<figure aria-label="Time spent on each question, colored by correctness">
  <ResponsiveContainer width="100%" height={280}>
    <BarChart>{/* ... */}</BarChart>
  </ResponsiveContainer>
  <figcaption className="sr-only">
    Bar chart with one bar per question. Green bars indicate correct
    answers, red bars indicate incorrect. Bar height is seconds spent.
  </figcaption>
</figure>
```

The `aria-label` is the short description (announced immediately). The `sr-only` figcaption is the long description (available to screen-reader users who navigate into the figure). The figcaption summarizes the chart's content without claiming each data point — the *adjacent table* (Topic 5) is the screen-reader-friendly representation of the same data. The chart visualizes; the table itemizes; the two together cover everyone.

That's the design pattern to internalize: **the chart is the visualization layer over an underlying data structure that's already accessible.** If the chart is the only representation of the data, you've built an inaccessible page.

## Keyboard Navigation Tab Order

Tab through the page; the order should be predictable. For the results page:

1. Skip link (focused first; reveals itself).
2. Global header nav links (if any are focusable).
3. Page content begins at `<main>`. The h1 is not focusable.
4. The first focusable element inside the page content is... whatever the user can interact with. On the results page, that might be the chart (recharts adds keyboard interactivity for tooltips on some axes), the retry button (only in error state), or nothing.
5. If nothing on the page is interactive, Tab moves to the page footer or browser chrome.

A page with no focusable interactive elements is fine. It means the user reads, doesn't click. The results page is largely that — read your score, see your chart, move on.

## Tooling

The cohort should know about, and run, **axe DevTools** in their browser at least once on each page. It catches:

- Missing alt text on images.
- Insufficient color contrast.
- Form fields missing labels.
- Improper ARIA usage.

It doesn't catch:

- Logical tab order.
- Whether the chart description is actually useful.
- Whether the page is comprehensible.

Automated tools are a floor, not a ceiling. The follow-up is **manual keyboard navigation** (tab through the page; can you reach everything?) and **screen-reader smoke test** (turn on VoiceOver on Mac, NVDA on Windows; does the page make sense?). Each candidate should do one screen-reader smoke pass before EoD. That's the orientation goal.

## What We're Not Doing Today

Out of scope for PEP a11y; flag for Phase 2 or later:

- **WCAG 2.2 AA conformance audit.** A real audit takes hours per page; we're aiming for "the floor is clean."
- **Reduced-motion media query.** `prefers-reduced-motion: reduce` to disable the skeleton's pulse animation. Worth adding; trivial; just out of scope today.
- **High-contrast mode testing.** Windows Forced Colors Mode and similar; matters for some user populations.
- **Internationalization a11y.** Language attributes (`lang="en"` on `<html>`) — the substrate sets this; localization is Phase 2.
- **Full keyboard-only chart exploration.** Recharts' default keyboard support for bars is limited. A truly accessible chart might expose each bar as a focusable element with `aria-describedby` linking to the data. Out of scope; the parallel table covers screen-reader users.

The point of naming these is honesty: PEP teaches the floor, signposts the gaps, doesn't pretend to have done more.

## Anti-Patterns

- **`<div onClick={...}>` for buttons.** Doesn't get keyboard activation, focus styling, or screen-reader announcement for free. Use `<button>`.
- **`outline: none` with no replacement.** Erases the focus ring. Keyboard users can't see where they are.
- **`aria-label="image"` on an actual image.** Either the image is decorative (`alt=""`) or it has meaningful content (`alt="..."`). Don't paper over with vague labels.
- **`role="button"` on a `<button>`.** Redundant; can confuse some screen readers. Native elements don't need their own roles.
- **Color-only conveyance.** Red = incorrect, green = correct, no text. Colorblind users see two equally gray bars.
- **Auto-focusing content on page load** to "save users a click." Screen-reader users lose their place; sighted users get a focus ring on something they didn't ask about. Don't.
- **`<table>` for layout.** The opposite mistake — using `<table>` to grid-out non-tabular content. Tells screen readers the content is data when it isn't.
- **Skipping the screen-reader smoke test.** axe DevTools passing doesn't mean the page is usable. Five minutes with VoiceOver finds problems no automated tool will.

## Key Takeaways

- The a11y floor: semantic HTML, skip link, visible focus rings, labeled chart with text alternative, `role="status"` / `role="alert"` for state changes, color paired with text, full keyboard reachability.
- Semantic HTML does most of the work. `<table>` for the table, `<button>` for buttons, `<main>` / `<h1>` / `<figure>` / `<figcaption>` for structure.
- ARIA fills the gaps semantic HTML can't: `aria-live` regions for status updates, `aria-label` for charts, `scope="col"` for table headers. Used sparingly.
- The chart is accessible because the adjacent table contains the same data in tabular form. The figure caption summarizes the visualization, not the data.
- Run axe DevTools as a floor; do a manual keyboard + screen-reader smoke pass to validate. Automated tools don't catch logical structure.

---
*Prerequisites: day-13-page-layout-patterns-with-tailwind, day-17-detail-view-ui-patterns-for-structured-data, day-17-data-visualization-with-charting-libraries-recharts.*
