# Cohort Retrospective Practice

> *Day 20: Capstone Demo Day — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

After the last capstone presentation lands, the cohort runs a **whole-cohort retrospective** — not the per-week retros that closed W2 and W3, but a single end-of-PEP session reflecting on the entire four weeks. This is the closing artifact of the program. The output goes into the PEP retro archive, gets reviewed by program leadership before the next cohort, and gets shared with Phase 2 trainers as "what this group wants you to know."

The retrospective is also the cohort's last shared experience before they bridge into Phase 2. It's the moment where four weeks of individual struggle becomes a collective record. The skill being practiced — contributing observations that are *actionable* and *specific* — is one of the higher-leverage professional skills in any engineering career. Most retros decay into vague gratitude or unfocused complaint; the discipline this topic teaches is the inverse.

## Objective

Contribute to a cohort-level retrospective with actionable observations.

## Format

A 60-minute retro, run by a trainer, with the full cohort present. Phase 2 trainers may sit in as observers. Structure:

| Time | Segment | Purpose |
|---|---|---|
| 0:00–0:05 | Frame the session | Re-state the rules: specific, actionable, no piling on individuals. |
| 0:05–0:20 | Silent generation | Each cohort member writes observations on sticky notes (or digital equivalent) in three buckets: *worked, change, flag for Phase 2*. |
| 0:20–0:35 | Affinity clustering | Trainer groups similar observations on a board; cohort discusses the clusters, not individual notes. |
| 0:35–0:50 | Prioritization | Cohort dot-votes top items in each bucket. Three to five clear winners per bucket. |
| 0:50–1:00 | Phase 2 handoff | What does this cohort want Phase 2 trainers to know? Captured separately. |

Sixty minutes is long enough to do this well, short enough that energy doesn't decay. The trainer's job is enforcement of structure, not contribution of content.

## The Three Buckets

### "What worked"

Things to keep doing — for this cohort and for the next. Specific, not platitudes.

*Useful examples:*
- "The 5 seeded CI bugs on D3–D4 forced reading config files instead of guessing. Bisecting from logs was a skill I didn't have before."
- "D11–D12's pessimistic-locking work made transactional thinking concrete. I'd never have learned this from reading a textbook."
- "The slice-integration days (D10, D15) caught wiring bugs that unit tests missed. Worth keeping."
- "The AI Tooling Thread being labeled as Unit 0 across all 20 days kept it visible. If it were one isolated topic, I'd have forgotten it."
- "Trainer doing live PR review on shared screen on D7 was the first time I understood what a code-review-as-mentorship looks like."

*Not useful examples:*
- "It was fun." (Not actionable.)
- "Everyone was nice." (Not actionable.)
- "I learned a lot." (Not specific.)

### "What to change"

Things this cohort would do differently. Aimed at the program, not at individuals.

*Useful examples:*
- "Day 6's reverse-proxy topic landed before we had real cross-service traffic; would have been better paired with D11 when we actually needed it."
- "The brownfield substrate had four backend services. Day 1 took most of the morning just to read them all. Either pre-reading the night before, or a smaller substrate."
- "Vitest didn't get its own intro day; we hit it cold on D14 when we needed it. Even one hour of dedicated Vitest fundamentals would have helped."
- "We had unclear conventions for branch names early. By W3 we'd settled, but a 'branch naming for PEP' note in the README on D1 would've saved confusion."
- "Daily AI tooling micro-prompts didn't always land — sometimes we already knew that day's technique, sometimes it was unrelated to the day's work. A more demand-pull pattern (we ask, we get) might be better than push."

*Not useful examples:*
- "There was too much to learn." (Vague.)
- "The trainer didn't explain enough." (Not actionable, and unfair without specifics.)
- "Wednesday was too long." (Not specific to what should change.)

### "Flag for Phase 2"

Things this cohort wants the Phase 2 trainers to know — about their preparation, gaps, or expectations.

*Useful examples:*
- "We're solid on FastAPI service scaffolding but never wrote a Lambda. The first time we hit async job patterns in Phase 2, expect it to be new for everyone."
- "We did SQL by hand a lot; nobody used an ORM seriously beyond SQLAlchemy basics. If Phase 2 introduces query patterns expecting deeper ORM fluency, plan accordingly."
- "We've used Claude Code daily for four weeks. Most of us can drive it well; some of us still don't read the diffs carefully enough. A re-emphasis on diff-reading in Week 1 of Phase 2 wouldn't be wasted."
- "Frontend coverage was lighter than backend coverage. We're competent with Next.js server components and basic forms, but state-management patterns beyond simple use-state are not deeply held."
- "The cohort settled into stable work patterns by W3. If Phase 2 starts with a hard reset (new team configs, new conventions), expect a productivity dip in the first few days."

*Not useful examples:*
- "Be patient with us." (Vague.)
- "We're tired." (Not actionable.)
- "Tell them we worked hard." (Not informational.)

## Rules That Make A Retro Useful

These rules are re-stated at the start of the session and enforced by the trainer:

1. **Specific, not vague.** "The pacing was tough" is not useful. "D11–D13 ran heavy on transactions and timer concurrency; D14 added the frontend autosave race on top; by D15 the cohort was visibly tired" is useful.

2. **About the program, not individuals.** Retrospectives that name specific cohort members for praise or criticism produce dynamics that hurt next time. Talk about the work, the substrate, the trainer's approach, the schedule — not about Jamie or Priya specifically.

3. **No piling on.** If someone raises a concern, the trainer doesn't ask "does anyone else feel that way?" Once it's surfaced, it's surfaced. The dot-vote step lets the cohort indicate weight without forcing public agreement.

4. **One pen per cluster.** During discussion of an affinity cluster, only one person speaks at a time. The trainer holds the pen (literal or metaphorical).

5. **No defending in real time.** If a cohort member observes "the X day felt rushed," the trainer doesn't immediately explain *why* it was paced that way. Defensiveness shuts down the rest of the session. Notes get captured; the response (if any) comes later, in the program-level review.

6. **Action items have owners or get dropped.** If an item under "change" doesn't have a clear owner (the program lead, the trainer pool, the curriculum author), it gets parked. Unowned action items decay; the cohort learns the retro doesn't lead anywhere; future retros become theater.

## The Phase 2 Handoff Note

The final 10 minutes are explicit: **what does this cohort want Phase 2 trainers to know?** This is captured as a separate artifact and handed to Phase 2 before their cohort starts.

The note has the shape:

```
TO: Phase 2 trainers
FROM: PEP cohort [date]
SIZE: 25 candidates
WHAT WE BUILT: [one-line summary of the four-slice app]

STRENGTHS GOING IN:
- [3-5 specific things this cohort is solid on]

GAPS GOING IN:
- [3-5 specific things this cohort hasn't deeply done]

WHAT WE WANT YOU TO KNOW:
- [3-5 cohort-specific observations about working style, prior experience, energy level, etc.]
```

The note is not a critique or a defense; it's a transfer of context. The Phase 2 trainers shouldn't have to re-learn the cohort from scratch.

## What Makes A Good Observation

A useful rubric for the cohort to self-check as they write notes:

- **Does it point at a specific thing?** A day, a topic, an exercise, a substrate decision, a trainer behavior. If it's "everything was X," it's not specific enough.
- **Could someone outside the cohort act on it?** If only a person who lived through the four weeks would understand the observation, it needs more context.
- **Is it about the experience, not the people?** "The morning session on D6 was hard to follow" is about the experience. "Trainer was confusing" is about the person — less useful and harder to act on.
- **Is it honest?** A cohort that produces only positive observations isn't being honest; a cohort that produces only complaints isn't either. Both extremes are signals of dysfunction.

## Common Retro Failure Modes

### Failure 1: The Gratitude Avalanche

Every observation is "the trainers were great." Possibly true; useless for improvement. Trainer intervenes: "What's one specific thing you'd want the program to do differently for the next cohort?"

### Failure 2: The Grievance Spiral

One observation surfaces a complaint; others pile on; the room's energy turns sour. Trainer intervenes: "Let's capture this, dot-vote on it, and move on. If it lands in the top items, we'll come back to it."

### Failure 3: The Vague Cloud

"It was a lot." "The pacing was hard." "Things were unclear." None of these are actionable. Trainer asks: "Can you point at a day or a topic?"

### Failure 4: The Performance For Trainers

Cohort members say what they think the trainer wants to hear. Watch for unanimous positivity or unanimous negativity — both are signals the cohort is performing rather than reflecting. Trainer: "What's one thing you'd say if I weren't in the room?"

### Failure 5: The Decision Capture That Goes Nowhere

The cohort prioritizes items, and then the items vanish. By next cohort, the same problems recur. Mitigation: every prioritized item has an explicit owner and a follow-up date, captured in the retro artifact. Items without an owner get dropped on the spot rather than orphaned.

## The Artifact

The retro produces a written document, stored in the PEP retro archive (location: program shared drive). Format:

```
PEP Retro — Cohort [date]
Size: 25 / Trainers: [names] / Substrate: rev-eval-ai (PEP variant) / Curriculum: PEP_4Week_DailyTopics_v2.3

What worked (top 3-5, with vote counts):
1. [...]

What to change (top 3-5, with owners and dates):
1. [...]

Flag for Phase 2 (transfer note):
1. [...]
```

This file is the canonical record. It gets compared to prior cohorts' retros to spot recurring issues, and to track whether prior "what to change" items were actually addressed.

## Why This Matters Beyond PEP

Retrospectives are a professional skill. Every engineering org runs them; very few run them well. The cohort that practices the discipline — specific, actionable, not personal, with owners — graduates with a skill that compounds across their career.

The Phase 2 program will run more retros, with more weight on action items. The 10-week intensive's complexity creates more retro material per week than PEP did per month. A cohort that already knows how to retro well saves Phase 2 trainers the work of teaching it.

## Anti-Patterns

- **Letting silence drag.** A 30-second silence is fine; a 3-minute silence kills the room. Trainer primes with a low-stakes prompt to restart.
- **The trainer dominating.** The retro is the cohort's; the trainer's job is structural, not contributory.
- **Writing notes for someone else.** Each cohort member writes for themselves. No one writes "I'm sure Priya thinks X" — Priya speaks for Priya.
- **Treating the retro as theater.** If the cohort suspects nothing will come of the retro, they'll stop putting effort in. The trainer's commitment to *actually doing something* with the output is what makes it real.
- **Skipping the silent-generation phase.** If discussion starts before everyone has written, the loudest voices set the agenda and quieter cohort members don't contribute.
- **Combining "what worked" and "what to change."** They're different cognitive modes. Separate them; the quality of both improves.
- **Letting the Phase 2 handoff become a complaint letter.** It's a *context transfer*, not a grievance log. Reframe if needed.

## Connecting Back

D10 and D15 introduced retrospectives at the per-week scale. Today scales the practice up to the per-program scale. The discipline is the same — specific observations, actionable items, named owners — but the time horizon and audience are larger. This is the cohort's last shared exercise of PEP, and the outputs influence the next cohort and Phase 2.

## Key Takeaways

- 60-minute structured retro: frame → silent generation → affinity clustering → prioritization → Phase 2 handoff.
- Three buckets: *worked*, *change*, *flag for Phase 2*. Keep them separate cognitive exercises.
- Observations must be specific, actionable, and about the program — not vague, not about individuals.
- Items under "change" need owners and dates, or they get dropped. Unowned action items decay the retro into theater.
- The Phase 2 handoff note is a context transfer, not a grievance log. Strengths, gaps, and cohort-specific notes for the trainers receiving this group.
- Common failure modes: gratitude avalanche, grievance spiral, vague cloud, performance for trainers, decision capture going nowhere. Trainer's job is to interrupt each.
- The artifact lives in the PEP retro archive and gets compared across cohorts to track program improvement.

---
*Prerequisites: day-10-retrospective-and-continuous-improvement-practice, day-15-retrospective-and-continuous-improvement-practice, day-20-pep-to-phase-2-bridge-the-ai-surface-coming-next.*
