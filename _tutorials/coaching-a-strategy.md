---
title: Coaching a strategy — and a bug the harness fixed for free
description: The same coached selection-sort strategy, before and after this project's own harness migration — a real bug that disappeared not because the algorithm changed, but because who executes it did.
order: 5
---

# Coaching a strategy — and a bug the harness fixed for free

The first tutorial described a strategy in your own words and let the skill turn it into a spec.
This one looks at a participant this repo already ships, `algorithm-coached-claude`, which sits at
a different point on the same spectrum: instead of a loose description, it's handed a fully
worked-out procedure — exact index-derivation rules, not just an algorithm's name. And because
this participant is old enough to predate this repo's own harness migration, it comes with
something none of the earlier tutorials had available: a **real bug, filed and explained in its
own README, from before the migration** — and the chance to check, with today's `dryrun.py`,
whether that bug survived into the generated-code version that replaced it. It didn't, and *why*
it didn't is the actual lesson.

## What the coaching document says

`participants/algorithm-coached-claude/AGENTS.md` teaches selection sort by direct placement:
find the leftmost index `p` that isn't yet in its final position, find `m`, the index of the
minimum of everything from `p` onward, and swap them — one swap makes one position final,
however far away `m` is. It also states, explicitly, a correctness argument: *"Because `i != j`
is required by the protocol, step 1's definition of `p` already guarantees `m != p`: if `array[p]`
were the minimum of its suffix, `p` would not have been chosen."*

That argument is correct. It didn't stop a real fault from happening anyway.

## The historical bug, kept honestly in the README

Before this repo's harness migration, this participant worked the way [Bring your own
participant online]({{ '/tutorials/first-participant/' | relative_url }}) describes as the
now-superseded architecture: a live model call, once per round, coached by the strategy document
inlined into its system prompt — no generated code, no determinism guarantee. Run against a real
array containing a duplicate value (`[16, 46, 23, 84, 46, 77]`, `46` twice), the model derived `p`
wrong and emitted a swap with `i == j` — exactly the case the coaching document says is
impossible if `p` is derived correctly. `validateMove` rejected it as a fault. A second run, on a
duplicate-free array of comparable difficulty, had zero faults — one data point each, not enough
to call it a duplicates effect on its own, but a real, reproducible failure, not a fluke: the
model re-derived the same wrong `p` consistently enough to cause a four-round oscillation
(`inversionsOverTime`: `9 → 4 → 7 → 4 → 7 → 4 → 1 → 0`) before breaking out.

The README keeps this writeup deliberately, unedited, as real project history — see the file
itself for the full account, including why "coaching improved the strategy; it did not make the
participant immune to re-deriving a wrong answer consistently" is the honest verdict for that
architecture.

## What's actually running today, and what a fresh test finds

That architecture is retired for **this** participant. `algorithm-coached-claude` now runs on the
same generated-code harness as `bubble-sort-claude` — the only other two-stage participant this
repo ships: `AGENTS.md` is the spec, `generate.sh` writes
`participants/algorithm-coached-claude/generated/handler.py` once, and `handler.sh` execs that
file — zero live model calls happen during a real run today. Reading the generated code shows the
same direct-placement logic the coaching document describes, but stated as plain, fixed Python.

Not every shipped participant migrated, deliberately: `chaotic-claude`, `minimal-claude`, and
`verbose-reasoner-claude` still make a live per-round `claude -p` call and have no `generate.sh` —
they're the control group this repo keeps on purpose, showing what harness variation alone (not
code generation) does to the numbers. `minimal-claude`'s own README states this explicitly: it's
"what you get with nothing added." So "retired" describes this one participant's own history, not
a repo-wide architecture change.

```python
def find_placement(array):
    n = len(array)
    for p in range(n - 1):
        m = p
        for k in range(p + 1, n):
            if array[k] < array[m]:
                m = k
        if m != p:
            return p, m
    return None
```

`m` is always the index of the **earliest** occurrence of the true minimum of `array[p:]` — not
"a" minimum, computed once, the same way, every time. That structurally forecloses the historical
bug: there's no live judgment call left that could re-derive `p` inconsistently.

Tested this directly rather than take that reasoning on faith — `dryrun.py`, run today, against
the exact kind of array that caused the original fault:

```
[46, 16, 46, 84, 23, 77]  (duplicate value, the original bug's own shape): rounds=5  faults=0  sorted=True
[5, 5, 5, 5, 5, 5, 5, 5]  (all duplicates):                                rounds=1  faults=0  sorted=True
[3, 1, 3, 1, 3, 1, 3, 1]  (alternating duplicates):                        rounds=5  faults=0  sorted=True
```

**Reproducing this needs a generation first, and your numbers may differ from these.** The
participant directory in the clone holds `AGENTS.md` and the scripts but no `generated/handler.py`
— that artifact is built, not committed, so a fresh clone cannot run these arrays until you run
`generate.sh`. And when you do, expect the *faults* to match and the *round counts* not necessarily
to: measured across 27 generations whose move mix was recorded, the same spec text produced
handlers that differ by a factor of 2.5 in rounds
([Change the skill]({{ '/how-to/change-the-skill/' | relative_url }}) has the measurement). Exact
round counts for generated code are a property of one draw; `faults=0` and `sorted=True` are the
properties of the *specification*, and those are the ones worth checking.

Zero faults, on the specific array shape that used to fault, plus two harder duplicate-heavy
shapes for good measure. Five further random 12-element arrays, a reversed 24-element array, and
the singleton/pair edge cases all came back `faults=0` `sorted=True` too, with `comparisons=0` on
every single run — confirming the coaching document's second claim (`compare` buys nothing when
you can already see the array) holds exactly as strongly as it did before the migration. The
`correction` probe came back the same `NOTE` result the last two tutorials explain: a handler this
deterministic never emits an invalid move to begin with, so it never receives a real correction to
react to.

## The actual lesson: the bug didn't move because the algorithm got smarter

It's tempting to read "the bug is gone" as "the harness migration made this participant better at
selection sort." That's not what happened. The direct-placement *algorithm* was already correct —
the coaching document's own correctness argument was right the whole time. What changed is **who
carries out the arithmetic each round**. Under the live-decision architecture, "derive `p` and
`m` from the current array" was a fresh judgment call a model made every round, and a judgment
call can be executed inconsistently even when the instructions describing it are exactly right.
Under the generated-code architecture, that exact same derivation is a fixed function, executed
identically every time it's called — there's no "mostly gets it right" left to fail on a
duplicate-heavy input, because nothing is being freshly decided anymore.

This is the same claim [Why generate code, not live
decisions]({{ '/explanation/why-generate-not-decide/' | relative_url }}) makes in the abstract,
but this tutorial gets to show it as a real before/after on one unchanged strategy, rather than
argue it in general terms: same coaching document, same algorithm, same correctness argument,
one real fault under one architecture and zero across 15+ fresh, targeted tests — including the
exact array shape that used to fail — under the other.

## The numbers, head-to-head

Five random 16-element seeds against `handlers/reference-sorter.sh` (insertion sort), budget
unlimited:

```
| Seed | coached rounds | reference rounds |
|------|-----------------|-------------------|
| 1    | 12              | 70                |
| 2    | 10              | 49                |
| 3    | 14              | 66                |
| 4    | 15              | 65                |
| 5    | 14              | 78                |
```

Roughly 5-6x fewer rounds — the largest margin of any participant covered in these tutorials, and
for a structural reason: direct placement needs **at most `n - 1` swaps, full stop**, regardless
of how scrambled the array is, because every swap finalizes one position permanently. [Change the
algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) showed that any strategy built
from adjacent swaps costs `inversions + 1` rounds no matter which algorithm chose them; direct
placement sidesteps that bound entirely by never doing an adjacent-only walk in the first place —
a stronger, more direct exploitation of the same non-adjacent-swap lever [Non-adjacent
swaps]({{ '/tutorials/non-adjacent-swaps/' | relative_url }}) used for comb sort, because every
swap here is guaranteed to finalize a position rather than merely reduce disorder by however much
the gap happens to buy.

## Once it's verified: go live

Same self-service path as every other tutorial here — a public waiting room, no operator file
edit. See [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}).

Prefer to try all of this without touching the live deployment? [Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }}) brings up the same bridge on your machine in about a minute — every command in this tutorial works against it unchanged.
