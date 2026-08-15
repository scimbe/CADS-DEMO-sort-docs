---
title: Partition mode — when every piece is sorted and the whole still isn't
description: A live, reproducible run against the real arena, where three participants finish their segments perfectly and the harness still reports the array unsorted.
order: 6
---

# Partition mode — when every piece is sorted and the whole still isn't

The first four tutorials were all about **handler-level** harness design: what a participant's own
code does with the array it's handed. This one is about a **mode-level** harness decision instead
— something [The move protocol]({{ '/reference/move-protocol/' | relative_url }}) defines once,
for everyone, that no participant's own strategy can opt out of. It produces a result that looks
like a bug the first time you see it and isn't one: every participant finishing its own piece
perfectly, and the arena still correctly reporting the whole thing unsorted.

## What partition mode actually does

Race mode (used for every head-to-head table in the earlier tutorials) gives each participant its
own **full, independent copy** of the same starting array. Partition mode does something different:
it splits **one** array by position into contiguous, non-overlapping segments — one per
participant — and each one sorts only its own slice. From inside a handler, this is
indistinguishable from an ordinary `solo` run against a shorter array; the only visible difference
is two extra fields on every round event, `segmentStart`/`segmentLength`, that let a client map a
segment-local `i`/`j` back to the whole array's real position.

## A real run against the live arena, verbatim

Three of this repo's shipped participants are currently reachable and working (a self-service
remote participant, `windows-selection-e2e`, is not, tracked separately on
[`ct-agent#15`](https://github.com/scimbe/ct-agent/issues/15) — this run deliberately only uses
participants confirmed live, not a hand-picked example):

```bash
curl -s -N -X POST "https://sort.bunsenbrenner.org/partition?ids=reference-sorter,bubble-sort-claude,algorithm-coached-claude&len=18"
```

An 18-element array splits three ways into 6/6/6. Real output, trimmed to the parts that matter
(every intermediate round streamed too — this is the `start` and `final` events only):

```json
{"stage":"start","initialArray":[65,42,98,98,61,81,89,76,27,61,34,56,57,2,21,60,44,68],
 "segments":[{"you":"reference-sorter","start":0,"length":6},
             {"you":"bubble-sort-claude","start":6,"length":6},
             {"you":"algorithm-coached-claude","start":12,"length":6}]}

{"stage":"final",
 "finalArray":[42,61,65,81,98,98, 27,34,56,61,76,89, 2,21,44,57,60,68],
 "wholeArraySorted":false,
 "perParticipant":[
   {"you":"reference-sorter",        "finishedCorrectly":true,"roundsUsed":7, "comparisons":0,"swaps":6, "faults":0},
   {"you":"bubble-sort-claude",      "finishedCorrectly":true,"roundsUsed":14,"comparisons":2,"swaps":11,"faults":0},
   {"you":"algorithm-coached-claude","finishedCorrectly":true,"roundsUsed":5, "comparisons":0,"swaps":4, "faults":0}]}
```

Look at `finalArray` split at the same 6/6/6 boundaries: `[42,61,65,81,98,98]`,
`[27,34,56,61,76,89]`, `[2,21,44,57,60,68]`. **Every single segment is internally sorted** — check
each one, they're all non-decreasing. All three participants reported `finishedCorrectly: true`
with zero faults. And the array as a whole — `42, 61, 65, ..., 98, 27, 34, ...` — very obviously
is not sorted: `98` is immediately followed by `27`.

This is exactly the outcome [The move protocol]({{ '/reference/move-protocol/' | relative_url }})
says to expect, not a bug this run happened to trigger: *"splitting by position is not the same as
splitting by value range, and concatenating locally-sorted slices only yields a globally sorted
array when each slice already happened to hold the right value range."* What's new here is seeing
it actually happen against the live arena with real participants, rather than taking the docs'
word for it.

## Why this is a harness lesson, not a participant bug

Nothing about this is any participant's fault, and nothing in a smarter handler could have
prevented it. `reference-sorter`, `bubble-sort-claude`, and `algorithm-coached-claude` were each
handed a 6-element slice with no visibility into the other two segments' values — from inside a
single handler invocation, sorting `[57, 2, 21, 60, 44, 68]` into `[2, 21, 44, 57, 60, 68]` is a
completely correct, completely finished job. **The gap lives entirely in what the mode itself
promises**: partition mode splits by array *position*, and a globally sorted result requires
splitting by value *range* instead (imagine handing segment 1 every value that belongs in
positions 0-5 of the final sorted array, segment 2 the next sixth, and so on) — which requires
information no single segment's handler has, by the mode's own design. Real parallel/external sort
algorithms solve exactly this with a merge phase after the segments finish; partition mode
deliberately doesn't implement one, because the point of this mode is watching segments sort
concurrently on screen, not shipping a working parallel sort.

Compare this to every earlier tutorial's failures: [Non-adjacent
swaps]({{ '/tutorials/non-adjacent-swaps/' | relative_url }})'s infinite loop and [Coaching a
strategy]({{ '/tutorials/coaching-a-strategy/' | relative_url }})'s historical fault were both
things a better handler could fix. `wholeArraySorted: false` here is not that kind of gap — it's
the harness telling you, correctly, what its own scoring actually measures per segment versus what
a human glancing at the picture might assume it measures overall. Knowing the difference before
you look at a partition-mode run is the actual point of this tutorial.

## The per-participant numbers still tell the usual story

Equal segment lengths (6 each) make the three `perParticipant` rows directly comparable, and they
land exactly where the earlier tutorials predict: `algorithm-coached-claude`'s direct-placement
strategy finishes fastest (5 rounds, 0 comparisons — [Coaching a
strategy]({{ '/tutorials/coaching-a-strategy/' | relative_url }})'s at-most-`n-1`-swaps bound in
miniature), `reference-sorter`'s insertion sort is respectable (7 rounds), and
`bubble-sort-claude`'s adjacent-only sweep is the slowest of the three (14 rounds, and the only one
to spend any rounds on `compare`). Partition mode doesn't change how a strategy's own efficiency
gets measured — only what "the array" that efficiency is measured against actually means.

## Try it yourself

Any set of currently-live participant ids works — check [`GET
/participants`](https://sort.bunsenbrenner.org/participants) for the current roster, then:

```bash
curl -s -N -X POST "https://sort.bunsenbrenner.org/partition?ids=<id-a>,<id-b>,<id-c>&len=<N>"
```

Watch for the same shape: `finishedCorrectly: true` across the board, `wholeArraySorted: false`
anyway, unless the values happened to already fall into position-ordered ranges — which, for a
random array, they usually won't.

No live arena needed, either: [Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }})
brings up the same bridge on your own machine in about a minute, and the exact `curl` above works
against `http://127.0.0.1:8789` unchanged — handy when the hosted deployment is down or you just
don't want to depend on it.
