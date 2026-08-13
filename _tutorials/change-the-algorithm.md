---
title: Change the algorithm — what the harness actually costs you
description: Build a merge-sort participant with the same skill, and find out why it doesn't beat insertion sort here.
order: 3
---

# Change the algorithm — what the harness actually costs you

[The first tutorial]({{ '/tutorials/first-participant/' | relative_url }}) built a participant and
checked it was reliable. This one changes the algorithm — from an adjacent-pair sweep to merge
sort — using the exact same skill, the exact same contract, no new machinery. The point isn't to
learn merge sort. It's to see, with real numbers, **what this specific harness — one shared array,
`compare`/`swap` only, one round per invocation, no memory between rounds — actually costs a
textbook `O(n log n)` algorithm.** The answer is not what Big-O notation would lead you to expect,
and that gap is the whole lesson.

This was validated the same way the first tutorial was: run start to finish in a clean container,
nothing pre-installed, real commands, real output.

## Before you begin

Same as the [first tutorial]({{ '/tutorials/first-participant/' | relative_url }}): Claude Code,
inside a clone of this repo, and a few minutes of real API cost. This run took 46 turns and about
$4 — noticeably more than the first tutorial's bubble-pass strategy, because describing merge sort
precisely enough for a stateless, memory-free handler turns out to be a real design problem, not a
restatement of a familiar algorithm. That difficulty is itself part of the lesson below.

## Step 1 — describe merge sort, and immediately hit the contract's real constraint

```bash
git clone https://github.com/scimbe/CADS-DEMO-sort.git   # or cd into your existing clone
cd CADS-DEMO-sort
claude
```

```
/sort-arena-harness

participant id: merge-sort-tutorial
display label: Merge sort (Claude, docs tutorial)
strategy: implement merge sort. Recursively split the array into two halves, conceptually sort
each half, then merge the two sorted halves back together by repeatedly looking at the front of
each half and moving whichever is smaller into the next output position, until both halves are
exhausted. Since the contract only gives me compare and swap on the one shared array (no separate
output buffer), implement the merge step in place: repeatedly compare the two candidate positions
and use a short chain of adjacent swaps to move the smaller one into place without disturbing the
already-merged prefix.
```

Notice what that strategy description already had to concede: real merge sort merges into a
**separate** output array. This contract has no second buffer — `swap` only ever exchanges two
positions in the one array everyone (including the visualization) is watching. Every
"merge sort, adapted to this contract" attempt has to solve that before it solves anything else.

## Step 2 — the real adaptation: recursion cannot survive a stateless invocation

The strategy above is still recursive in spirit — "sort each half, then merge." But a handler is
invoked **fresh every round**, per [the move protocol]({{ '/reference/move-protocol/' | relative_url }}):
no call stack, no local variables persisting between rounds, only whatever it can reconstruct from
the round input it's just been handed. Recursion needs a call stack. This doesn't have one.

The real fix — the actual generated spec's core idea — was switching to the **bottom-up** form of
merge sort: the same merges (runs of 1 combine into 2, 2 into 4, 4 into 8, …), performed in a
different order that happens to be fully recoverable from the array's current values alone, with
nothing remembered:

1. **`W`** — the largest power of two smaller than the array length for which every aligned
   `W`-sized block is already internally sorted. That tells you which merge *pass* you're on.
2. **`B`** — the leftmost aligned `2W`-block that isn't sorted yet. That tells you which merge is
   currently in progress.
3. **`d`** — the leftmost descent (the first `array[k] > array[k+1]`) inside `B`. Emit
   `swap(d-1, d)`.

The non-obvious part — worth pausing on, because it's the actual insight, not a restatement of the
steps above — is *why* step 3 works: `d` is standing in for the merge's usual right-hand read
pointer, recovered from values instead of stored as a variable. The merge invariant guarantees the
already-merged prefix is `<=` everything not yet consumed, so the merged prefix plus the remaining
left run together form one sorted run — meaning the block has **exactly one** descent, and it sits
precisely where the classic algorithm's pointer would be. While an element is still travelling
left mid-merge, that same descent position *is* the element's current location, so the identical
rule keeps working, round after round, without anything being remembered.

One direct consequence, worth naming because it's a real robustness property: this handler
**never reads `history` at all**. It's a pure function of the current `array`. A handler that
instead tries to reconstruct "which pass am I on" from the history log is vulnerable to that log
being incomplete — faults, skipped rounds, and rejected `done` claims all append nothing to
`history`, which is capped at 20 entries regardless. Reading only `array` sidesteps that class of
bug entirely.

## Step 3 — the numbers, and why they're not what "merge sort" promises

Here's the real dry-run report, verbatim, because the gap between what you'd expect and what
actually happened is the point of this whole tutorial:

```
Dry runs (all faults=0, all finished correctly sorted):

| Array                  | rounds | compares | swaps |
|-------------------------|--------|----------|-------|
| [18,60,61,29,26,25]     | 10     | 0        | 9     |
| reversed, n=12          | 67     | 0        | 66    |
| nearly sorted, n=10     | 2      | 0        | 1     |
| many duplicates, n=10   | 16     | 0        | 15    |
| already sorted / n=1    | 1      | 0        | 0     |
```

And, head-to-head against `handlers/reference-sorter.sh` — the shipped, non-LLM, real insertion
sort baseline — on the identical array:

```
| Participant                                    | rounds | comparisons | swaps | faults |
|--------------------------------------------------|--------|-------------|-------|--------|
| merge-sort-tutorial                              | 10     | 0           | 9     | 0      |
| handlers/reference-sorter.sh (insertion sort)    | 10     | 0           | 9     | 0      |
```

**Identical.** Not close — the same number, every time, on every array tested. Here's why, and it's
a real, provable bound, not an artifact of this one run:

- **Comparisons are free here.** The round input hands the handler the *entire* array already —
  there's nothing to discover by comparing that the handler couldn't already see. So a correctly
  designed handler spends zero rounds on `compare`; every round is a `swap` that actually makes
  progress. `O(n log n)` is a bound on *comparisons* — and comparisons cost this handler nothing
  regardless of which algorithm chose them.
- **`swap` only ever exchanges two adjacent positions in this handler's design**, and an adjacent
  swap removes at most one inversion. So **any** strategy built entirely out of adjacent swaps —
  bubble-shaped, insertion-shaped, or an in-place bottom-up merge like this one — costs *exactly*
  `inversions + 1` rounds to finish, full stop. That's not a property of any one algorithm; it's a
  property of the move contract's cheapest unit of change.

Merge sort's textbook advantage — fewer comparisons than insertion sort in the worst case — simply
doesn't show up in a scoring system where comparisons don't cost rounds and only adjacent swaps are
used. **This is the actual lesson of this tutorial**: the harness — what gets scored, what a move
is allowed to touch, what's visible for free — determines which of an algorithm's real advantages
survive contact with it, and which ones evaporate. A different contract (say, one that scored
comparisons, or allowed a non-adjacent `swap` to cost the same as an adjacent one) would produce a
completely different verdict about the exact same two algorithms. Nothing here is a flaw in either
implementation; it's what happens when you hold the rules fixed and only change the algorithm.

If you want a participant that actually *finishes* faster than the insertion-sort baseline in this
arena, the lever the contract leaves you is non-adjacent swaps — moving a value directly to its
final position in one round instead of walking it there one adjacent step at a time. That's a
different, harder spec to write (and a natural next experiment once you've internalized why the
adjacent-only version can't win on rounds).

## Step 4 — verify it wasn't a lucky guess

Same three checks as the first tutorial, plus one this run added because "merge sort, adapted" is
easy to get subtly wrong: comparing the generated code's actual decisions, one at a time, against
an independent reference implementation of the spec's own three-step rule, over thousands of
random arrays:

```
Spec-conformance fuzz: 3,000 random arrays (n = 0…24, including heavy-duplicate arrays),
138,154 decisions compared, 0 divergences.
```

That's a materially stronger claim than "it sorted every array I tried" — it means the generated
code implements *this spec*, not merely *some* correct sort that happens to agree with it on the
arrays actually tested. The determinism and `correction`-handling checks from the first tutorial
both passed too, including the edge case worth calling out explicitly: on a one-inversion array
where the usual "pick a different descent" correction has no second descent available to fall back
to, the generated handler correctly drops to a `compare` instead of emitting something invalid.

## What this tutorial found that wasn't the point of it

One real, verified finding fell out of validating this in a clean container — not a merge sort
problem, and worth knowing if you're about to write your own participant:

- **A gitignored `generated/` directory means a fresh clone can't run every shipped participant
  out of the box.** `bubble-sort-claude`'s own generated handler isn't committed (correctly — it's
  build output), so a first-time clone needs `participants/bubble-sort-claude/generate.sh` run once
  before that particular participant works. If a participant faults on every round immediately
  after a fresh clone, this is the first thing to check, not a sign your own setup is broken.

(An earlier run behind this tutorial also hit `dryrun.py` not existing as a repo file, despite
being referenced by the skill, `participants/CLAUDE.md`, and every template README. That gap is
closed now — `dryrun.py` is a real, committed file at this repo's root, so a fresh clone has it.)

## Once it's verified: go live

Same self-service path as the first tutorial — a public waiting room, no operator file edit. See
[Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}).
