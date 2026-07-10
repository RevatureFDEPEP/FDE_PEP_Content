# Live Demo Backup Patterns (Recorded Fallback)

> *Day 19: Trainer Dashboard Slice (Frontend) + Polish — PEP 4-Week Curriculum*
> *Week 4: Results, Trainer Dashboard, Capstone*

## Overview

Topic 6 covered planning the demo: selection, sequencing, risk mitigation in the abstract. This topic is one specific risk mitigation that deserves its own treatment because it's load-bearing for capstone day: **record a backup walk-through** so a live failure doesn't tank the demo. Murphy's law applies — the backend might refuse to start, the wifi might drop, the laptop might crash, a hot reload might wedge the frontend, the seeded data might be stale. None of these are recoverable in real time during an 8-minute demo. The professional answer is *not* to gamble that the live demo will work; it's to have a 5-minute pre-recorded walk-through ready to play if any of the live-demo risks materialize. This is standard practice at every senior engineering org doing live demos for stakeholders, and the cohort should leave PEP with the recording-and-fallback discipline as a permanent habit. Recording is also a useful artifact in its own right: a portfolio-quality video the cohort can share post-demo with recruiters.

## Objective

Record a backup demo walk-through that can substitute for the live demo if it fails.

## Why Pre-Record

Two arguments:

1. **Demo reliability.** A live demo has ~5 independent failure modes (backend up, frontend up, wifi up, seed data correct, browser state clean). Each is ~95% reliable individually. Multiplied: 0.95^5 ≈ 77%. **One in four live demos has *some* failure.** Most failures are recoverable mid-demo with composure; some are not. The recording is the safety net for the un-recoverable case.

2. **Audience time respect.** When something does go wrong live, the audience watches the candidate troubleshoot for 60 seconds and loses interest. Switching to a recording maintains tempo and the audience's attention. The candidate can explain "the backend isn't cooperating today, here's the walk-through I recorded yesterday" and continue narrating over the video. The demo lands; the live failure is a footnote.

The cohort should *not* think of the recording as "what we'll show if everything is broken." It's "what we have available so a small thing breaking doesn't compound." A 30-second pause to switch to the recording, then continue, is far better than a 5-minute self-rescue.

## The Recording Spec

A backup recording for PEP capstone:

- **Length:** 5 minutes. Shorter than the live demo by design — the recording is denser, no live-demo pauses, no improvisation.
- **Resolution:** 1080p minimum. Lower quality reads as "this person isn't taking presentation seriously."
- **Audio:** Optional. A silent recording the candidate narrates over is fine and often better — narration during a live presentation feels personal, recorded narration feels like a YouTube tutorial. Pick one and commit.
- **Content:** Same 4-segment flow as Topic 6 (candidate flow → results → trainer flow → code highlight), but compressed.
- **No transitions, no music.** It's a backup, not a YouTube edit. Keep it raw.
- **Cursor visible, clicks visible.** The audience needs to follow what's happening. Most screen recorders default to showing the cursor.

## Tooling

Three reasonable options, ranked by ease:

1. **OBS Studio (Open Broadcaster Software).** Free, cross-platform, the standard. Set up a "Display Capture" source for the browser, set output to 1080p MP4, hit Start Recording. Output goes to `~/Videos` by default.
2. **Built-in OS screen recorder.**
   - Windows: `Win+G` opens Game Bar, Record button.
   - macOS: `Cmd+Shift+5` opens screen recording, select area.
   - Lower fidelity than OBS but zero setup.
3. **Loom.** Browser-based, hosts the video, generates a share link. Free tier limits length but 5 minutes fits.

PEP recommends OBS for one reason: the resulting file is a local MP4 the cohort owns. Loom is hosted (depends on Loom being up during the demo). The OS recorders work fine but vary in quality.

Whichever the cohort picks, test it *today*, not at 3am the night before capstone.

## The Recording Workflow

1. **Clean the environment.** Quit Slack, Discord, calendar pop-ups, anything that could overlay the screen. Use a clean browser profile with no bookmarks bar, no extensions visible.
2. **Verify seed data.** The recording is only useful if it shows realistic data. Confirm the candidate user has 3-4 attempts; the trainer dashboard shows 10+ tests and 25+ attempts.
3. **Start the recording.** Hit record, wait 2 seconds (in case of leading frames), then start.
4. **Walk through the 4 segments at a measured pace.** Slower than the live demo — the recording has no live-demo nervousness compression. Aim for 1:15 per segment.
5. **Stop the recording.** Save as `pep-demo-backup-YYYY-MM-DD.mp4`.
6. **Watch it.** If it's wrong, re-record. The cohort should be willing to re-record 3-4 times to get one acceptable take.
7. **Store it in two places.** Local file *and* cloud (Google Drive, Dropbox). If the laptop dies the morning of the demo, the cloud copy is the failover-for-the-failover.

A useful invariant: if everything is broken, the cohort can open the cloud URL on a phone and play the recording in the worst case. Two failovers deep, the demo still happens.

## Storage And Naming

A predictable file location reduces fumbling. Suggested layout:

```
~/pep-capstone/
├── demo-script.md              # Topic 6's bullet plan
├── demo-backup-2026-05-20.mp4  # Today's recording
├── demo-backup-2026-05-19.mp4  # Yesterday's draft (keep, in case today's is corrupt)
└── README.md                   # "How to play this in an emergency"
```

The README is 3 lines: where the file is, what plays it (VLC works on everything), what to say while it loads. The cohort writes that README *before* needing it.

## Playing The Recording During The Demo

If the live demo fails and the recording goes up, the candidate should:

1. **Acknowledge briefly.** "The backend's misbehaving today — let me play the walk-through I recorded." Don't dwell.
2. **Open the file.** Pre-staged on the desktop. Double-click. VLC or QuickTime opens. Hit fullscreen.
3. **Narrate over the video.** This is the most important part. The audience values *your* explanation alongside the video. Don't just play it silently — talk through what's happening like a director's commentary.
4. **Pause at meaningful moments.** "Here's where the chart renders, this is the recharts component from D17..." Press space; talk; press space again.

The recording is the *substrate* for continued narration, not a replacement for the candidate. The audience still wants to hear from the cohort member, just over the recording instead of over the live app.

## When To Switch To The Recording

A useful rule: **if the demo is dead for >20 seconds, switch.** Live troubleshooting beyond 20 seconds reads as "this candidate is improvising and stressed." Quick switch reads as "this candidate planned for failure and adapted." The 20-second threshold is fast enough to maintain tempo and slow enough that minor hiccups (a slow page load, a typo in a field) don't trigger an unnecessary switch.

Conditions to switch immediately:

- Backend container is dead and won't restart in 5 seconds.
- Frontend has a syntax-error overlay.
- Wifi has dropped (rare on a Docker-localhost setup, but possible if the demo uses any remote resource).
- Demo machine is locked or showing a system update.

Conditions to *not* switch and just push through:

- A page is slow to load. Talk while it loads.
- One filter doesn't return the expected results. Move past it; the chart still renders.
- A typo in the URL bar. Fix it.

The cohort should rehearse the *decision* to switch as part of demo practice. A trainer cue ("the backend just died, what do you do?") helps the cohort practice the verbal transition.

## What To Record Beyond The Demo

The cohort might also record:

- **A 30-second project introduction.** "This is a quiz-taking application I built over four weeks..." Useful as a portfolio asset.
- **A code walk-through of one feature.** Two minutes on the scoring engine, the role-gated endpoint, or the server-component pattern. Useful for technical interviews after PEP.

These are stretch goals, not required for capstone day. The 5-minute demo backup is the minimum deliverable.

## A Note On Style

The recording should not look slick. Slick reads as "this candidate spent more time on production than on engineering." A clean cursor moving through the app at a measured pace, with no transitions and no music, is exactly right. Loom-style polish ("hey everyone, in this video we'll be looking at..." with title cards) is the wrong tone for an engineering capstone.

The cohort is demonstrating an *engineering project*, not producing a marketing video. The recording should look like an engineer showing their work to another engineer.

## The Pre-Demo Checklist (Day 20 Morning)

Add the recording to the morning-of checklist:

- [ ] Backup recording plays from `~/pep-capstone/demo-backup-*.mp4`.
- [ ] Cloud copy of recording is accessible (open URL, verify download).
- [ ] VLC (or default player) opens the file in fullscreen on double-click.
- [ ] Volume works.
- [ ] Recording is the latest version (not yesterday's draft).

Five checkmarks, two minutes total. Done before the cohort opens the live app.

## Anti-Patterns

- **Recording the morning of the demo.** No time to fix problems with the recording. Record by EoD today.
- **Recording with the laptop on battery and screen-dim engaged.** The video looks darker than the live demo. Stay plugged in, brightness consistent.
- **Recording at small resolution to "save space."** The audience can't read the UI. 1080p minimum.
- **Editing the recording.** Cuts, splices, music — wrong tone. One take, raw, ship it.
- **Hiding the cursor in the recording.** The audience can't follow what was clicked. Cursor visible.
- **Forgetting to seed data before recording.** "I have no attempts" makes the recording useless. Verify data first.
- **One copy of the recording.** Local-only means a dead laptop is a dead demo. Two copies, one cloud.
- **Treating the recording as a "if everything's broken" parachute only.** It's also useful for "the wifi is acting up" mid-segment — switch for 60 seconds, switch back when wifi recovers. The recording is a tool with multiple uses.

## Connecting Back To Topic 6

Topic 6 framed risk mitigation in the abstract: rank the top three risks, plan the exact action for each. This topic provides *one specific action* — switching to the recording — that's the right answer for several of the most common risks. The cohort should treat "switch to the recording" as the default fallback for any failure that doesn't resolve in 20 seconds.

The discipline of "we recorded a backup" is something senior engineers do at conferences, in interviews, in pitch meetings. The cohort acquiring it on capstone day is real career skill, not just curriculum theatre. Murphy's law applies forever; the recording habit applies forever.

## Key Takeaways

- Pre-record a 5-minute backup walk-through of the demo, before EoD today. Use OBS or the OS screen recorder; 1080p, raw, no editing.
- The recording is a safety net for any live failure that takes >20 seconds to resolve. Quick switch maintains tempo; long live troubleshooting loses the audience.
- Narrate *over* the recording when playing it. The audience still wants the candidate's voice; the recording is just the substrate.
- Store the recording in two places: local file + cloud copy. A dead laptop without the cloud copy is a dead demo.
- Pre-demo checklist: verify the recording plays, the cloud URL is accessible, the player works, the volume works. Two minutes, every time.
- Don't edit. Don't add music. Don't hide the cursor. The recording should look like an engineer showing their work to another engineer.

---
*Prerequisites: [06-demo-preparation-selection-sequencing-risk-mitigation.md](06-demo-preparation-selection-sequencing-risk-mitigation.md).*
