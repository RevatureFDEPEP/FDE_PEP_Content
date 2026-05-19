# Optimistic Updates vs Server Confirmation

> *Day 14: Test-Taking Page (Interactive) — PEP 4-Week Curriculum*
> *Week 3: Quiz-Taking Slice (Sessions, Scoring, Results)*

## Overview
Every mutating action on the test-taking page is a choice between two postures: "show the new state immediately and hope the server agrees" (optimistic) or "wait for the server, then show the new state" (confirmed). Optimistic feels faster and lies more; confirmed feels slower and tells the truth. There is no universally right answer — there is a per-action right answer, driven by how reversible the action is, how often it fails, and what the user loses if the UI gets it wrong. D14 has two clear cases — autosave and submit — and they go in opposite directions. This topic builds the decision framework, then nails it down for the page.

## The Two Postures Defined

**Optimistic update:** the client updates its local state to reflect the intended new state *before* the server has confirmed. The network call runs in the background; on success, nothing further is needed; on failure, the UI must roll back and surface an error.

**Confirmed update (also called pessimistic or server-confirmed):** the client shows a loading indicator, waits for the server response, and only then updates local state. On failure, the UI was never lying — it just stays in the prior state and shows an error.

In a typical CRUD app, optimistic is associated with React Query's `onMutate` + rollback in `onError`. Confirmed is associated with the more straightforward `mutateAsync` followed by `await`. Both are baseline patterns; the choice between them is what this topic is about.

## The Decision Table

For every mutating action, evaluate four dimensions:

| Dimension | Pushes toward optimistic | Pushes toward confirmed |
|---|---|---|
| **Reversibility** if the server rejects | Trivial rollback (just re-set local state) | Hard to roll back (UI already showed consequences) |
| **Failure rate** | Low (idempotent retries usually succeed) | High (validation errors, conflicts likely) |
| **User cost of seeing the wrong state briefly** | Low (cosmetic) | High (decisions made on false info) |
| **User cost of perceived latency** | High (waiting feels broken) | Low (user knows it's a heavyweight action) |

Score the action across these four; the dominant dimension picks the posture.

## D14's Two Actions, Scored

### Autosave per question

- **Reversibility:** trivial. The local state is "I selected option B"; on rollback we just mark the autosave as failed and the candidate sees a red save indicator. The selection itself stays.
- **Failure rate:** low. D12's endpoint is idempotent (Topic 4 idempotency keys), and the network is the only realistic source of failure.
- **User cost of wrong state briefly:** low. The candidate sees their selection highlighted regardless of save status; the save indicator is a small badge in the corner.
- **User cost of latency:** high. If every checkbox click waited 200ms for a server round-trip before highlighting, the page would feel unresponsive.

**Verdict: optimistic.** Highlight the selection immediately, fire the save in the background, surface a rollback only via the save-status badge.

### Submit-and-lock the attempt

- **Reversibility:** *not reversible*. D12's submission locks the attempt server-side; once we tell the candidate "submitted, here's your score," there's no taking it back.
- **Failure rate:** non-trivial. The session might have expired (`410 Gone`), the attempt might already be submitted from another tab (`409 Conflict`), the candidate might have unanswered questions, the network might be offline.
- **User cost of wrong state briefly:** *catastrophic*. Showing "Submitted!" and then having to retract is a trust-destroying UX failure.
- **User cost of latency:** acceptable. Candidates understand that "submit" is a heavyweight action; a 500ms wait with a clear spinner is fine.

**Verdict: confirmed.** Show a spinner, wait for the server, only show the locked state on the server's confirmation. Topic 6 handles the UX details.

### The decision table for D14, summarized

| Action | Posture | Why |
|---|---|---|
| Selecting/deselecting an option (visual highlight) | Optimistic (purely local) | Pre-network UI state; doesn't even hit the server |
| Autosaving the answer to backend | Optimistic with rollback indicator | Cheap to roll back; latency cost too high otherwise |
| Submitting the entire attempt | Confirmed | Irreversible; failure modes are real |
| Navigating between questions | N/A (pure client state) | No network involved |
| Timer expiration auto-submit | Confirmed | Same reasoning as manual submit |

## Implementing Optimistic Autosave

The reducer (Topic 1) tracks autosave status per question. The optimistic posture means the local "selected options" updates *before* the network call begins, and the autosave status moves through `saving → saved` or `saving → error`. The candidate's *selection* never depends on save success — that's what makes this safely optimistic.

```ts
// Pseudocode in the AutosaveWatcher from Topic 4
async function flush(questionId: string, selected: number[]) {
  // OPTIMISTIC: the selection is already updated in RHF's store and reflected in the UI.
  // We only update the save *status*.
  dispatch({ type: "AUTOSAVE_START", questionId });
  try {
    await api.submitAnswer(sessionId, { question_id: questionId, selected, idempotency_key: keyFor(questionId) });
    dispatch({ type: "AUTOSAVE_SUCCESS", questionId });
  } catch (e) {
    // ROLLBACK: not of the selection (the candidate's intent is preserved), but of the
    // "this is safe" signal. The candidate sees a red badge and Topic 7's retry kicks in.
    dispatch({ type: "AUTOSAVE_ERROR", questionId });
  }
}
```

The candidate's mental model: "My answers are saved unless I see a problem badge." Optimistic UI works because the rollback is *informational*, not destructive. We never un-select an option behind the candidate's back.

## Implementing Confirmed Submit

Submit is the inverse: nothing changes in the UI's notion of "this attempt is finished" until the server says so. The submit-status reducer from Topic 1 (`idle → submitting → submitted | error`) is the state machine.

```ts
// Submit handler in <TestRunner>
const onSubmit = methods.handleSubmit(async (values) => {
  dispatchSubmit({ type: "SUBMIT_START" });
  try {
    const result = await api.submitSession(session.session_id, {
      answers: values,
      idempotency_key: `${session.session_id}:submit:1`,
    });
    // CONFIRMED: only now do we mark submitted and store the locked timestamp.
    dispatchSubmit({ type: "SUBMIT_SUCCESS", lockedAt: result.locked_at });
  } catch (e) {
    dispatchSubmit({ type: "SUBMIT_ERROR", error: classifyError(e) });
  }
});
```

While `submitStatus === "submitting"`, the UI:

- Disables the submit button (Topic 6 again).
- Disables question navigation (the candidate can't edit answers mid-submit).
- Shows a spinner near the submit button.
- Does *not* show a success state, lock indicator, or score preview.

Only after `SUBMIT_SUCCESS` do all of those flip on. There is no in-between "probably submitted" state shown to the candidate.

## The Hybrid Case: Showing The Selection Locally Before Autosave

Selecting an option has two layers of optimism:

1. **The radio/checkbox highlights immediately** because we update RHF's field store synchronously in the `onChange` handler. The UI doesn't even *look* at autosave status to render the selection — that data lives in RHF.
2. **The autosave fires in the background**, optimistically marking itself as in-progress, then resolving.

The candidate sees the selection highlighted immediately (layer 1), then sees a brief "saving..." then "saved" badge (layer 2). The two are independent, which is the whole point — local UI responsiveness doesn't block on the network, and save status is its own readable signal.

## Why The Naive "Always Optimistic" Posture Fails

A common shortcut — "just make everything optimistic, it's snappier" — gets you bitten on submit. Imagine implementing submit optimistically:

1. Click "Submit."
2. UI immediately shows "Submitted! Score: pending..."
3. Background POST returns `409 Conflict` (already submitted from another tab).
4. UI now has to retract "Submitted!" and reveal — what? A previously hidden submission's score? An error?

This is exactly the "wrong state briefly" failure mode that the decision table called out. The fix isn't a better retraction UX; the fix is to not show the false state in the first place.

Conversely, the naive "always confirmed" posture makes autosave feel terrible. Wait 200ms after every checkbox click before highlighting? Unusable.

The discipline is per-action evaluation.

## Telling The Candidate The Truth

Whatever posture you pick, the UI must accurately reflect *which posture is in play*. Optimistic UI needs a save-status indicator so the candidate can tell when their work isn't actually persisted. Confirmed UI needs a loading indicator so the candidate knows the action is in flight, not silently dropped.

Concretely, the test-taking page has:

- **Per-question save badge** (Topic 4 reducer state): green check, gray "saving...", red "save failed" with retry hint.
- **Submit button states** (Topic 6): "Submit," "Submitting...", disabled-while-locked, error message.
- **Top-of-page banner** for global failures (Topic 7): offline, repeated save failures.

Each of these is the honest signal corresponding to its action's posture.

## Common Mistakes

- **Optimistic submit.** As above — irreversible, has real failure modes, retraction destroys trust. Always confirm.
- **No rollback UI for optimistic actions.** Optimism without a "this might not have worked" signal is just lying. The save-status badge isn't optional.
- **Confirming actions that can be safely optimized.** If every UI interaction waits for a server round-trip, the page feels unresponsive. Inventory which actions are reversible and cheap to roll back.
- **Updating both local *and* server state in a confirmed action only on success.** If the server returns a slightly different shape than you predicted (e.g., timestamps, IDs), trust the server response over your prediction. Read the response, then update from it.
- **Optimistically updating a *different* user's view of state.** Optimism is for the user who initiated the action. Other users on the same data (e.g., a trainer watching D18's dashboard) must see only confirmed state.

## Key Takeaways
- The optimistic-vs-confirmed choice is per-action, driven by reversibility, failure rate, cost of wrong state, cost of latency — not a global setting.
- D14's autosave is optimistic (cheap rollback via a status badge); D14's submit is confirmed (irreversible, real failure modes, "Submitted!" must be the truth).
- Optimistic UI requires an explicit "this might not have worked" signal — a save badge, an error toast, something. Optimism without that signal is just lying.
- Selecting an option is *doubly* optimistic: the highlight is purely local (no network), the autosave is optimistic in the background.
- Submit shows no "submitted" state until the server confirms, including the locked timestamp from D12's response.

---
*Prerequisites: day-11-server-authoritative-state, day-12-idempotency-for-retried-mutations, day-12-state-finalization-and-immutability-patterns, day-14-client-side-temporal-state-timers-and-autosave. Forward references: day-14-submit-and-lock-ux-patterns, day-14-graceful-network-failure-handling.*
