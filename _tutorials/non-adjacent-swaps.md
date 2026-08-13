---
title: Non-adjacent swaps — writing a handler by hand, no skill
description: Build a comb-sort participant directly from the move contract (no CLI, no code generation), and find a real infinite loop before it ever goes near the arena.
order: 4
---

# Non-adjacent swaps — writing a handler by hand

The first two tutorials both used the `sort-arena-harness` skill: describe a strategy, get
generated code back, verify it. This one takes the other documented path — [Join as a
participant]({{ '/how-to/join-as-a-participant/' | relative_url }})'s "Manual fastest start" — and
writes a handler directly against [the move protocol]({{ '/reference/move-protocol/' | relative_url }}),
no coding CLI, no API cost, just a text editor. It also picks up the exact question [Change the
algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) closed on: that tutorial proved
any handler built entirely out of *adjacent* swaps costs `inversions + 1` rounds, full stop, no
matter which textbook algorithm chose them. The lever it named but didn't use was **non-adjacent**
swaps — moving a value more than one slot per round. This tutorial uses that lever, and finds a
real bug in the process worth knowing about before you reach for it yourself.

The finished, verified handler is a real file in the repo —
[`handlers/comb-sort.sh`](https://github.com/scimbe/CADS-DEMO-sort/blob/main/handlers/comb-sort.sh)
— not just the fragments quoted below. Clone it and run it yourself rather than retyping anything
here; that's the whole point of a "manual fastest start" tutorial.

The algorithm is comb sort: compare pairs a fixed **gap** apart instead of only adjacent pairs,
swap out-of-order ones, then shrink the gap (here by a factor of 1.3, the usual choice) each pass
you complete without a violation, down to gap 1 — at which point a clean pass means the array is
actually sorted, same terminating condition ordinary bubble sort uses.

## Before you begin

Nothing beyond what [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }})
already lists: `python3` (or `python` on Windows) and a bash-compatible shell to run `dryrun.py`
and the handler it drives. No Claude Code, no API key, no cost — the whole point of the manual path
is that it doesn't need any of that.

## Step 1 — the handler, and the state problem every non-adjacent design has to solve

A handler is invoked fresh every round with no memory of the last one — [Bring your own
participant online]({{ '/tutorials/first-participant/' | relative_url }})'s failure story is about
exactly this. An adjacent bubble-pass handler can get away with reconstructing "where am I in this
pass" from `array` alone. A gap-based handler can't: **the current gap itself is a number nothing
in `array` encodes.** It has to come from somewhere else, every single round.

The obvious source is `history`: look at the most recent move, and read the gap back out of its
own `i`/`j`. First attempt:

```python
last_swap = next((e for e in reversed(history) if e["action"] == "swap"), None)
gap = abs(last_swap["j"] - last_swap["i"]) if last_swap else max(1, int(n / 1.3))
```

This looks reasonable, dry-ran clean on one seed, and was wrong.

## Step 2 — a real infinite loop, caught before it ever reached the arena

Running the handler above through `dryrun.py` — the same script now committed at this repo's own
root, `python3 dryrun.py ./handlers/comb-sort.sh --seed 1 --len 12` — five random 12-element arrays
produced this:

```
seed 1: rounds=200 comparisons=199 swaps=1  faults=0 sorted=False  (budget exhausted)
seed 2: rounds=200 comparisons=195 swaps=5  faults=0 sorted=False  (budget exhausted)
seed 3: rounds=200 comparisons=193 swaps=7  faults=0 sorted=False  (budget exhausted)
seed 4: rounds=15  comparisons=0   swaps=14 faults=0 sorted=True
seed 5: rounds=200 comparisons=194 swaps=6  faults=0 sorted=False  (budget exhausted)
```

Four of five arrays never finished. Not faulted — `faults=0` throughout, every reply was valid
JSON, well-formed, in range. It burned its entire 200-round budget almost entirely on `compare`
and simply never got anywhere. Seed 4 passing on the first try, alone, is what made this dangerous:
it's exactly the kind of result that looks like success if you only try one array.

The cause, once you look at what a `compare` round actually does here: when the current gap turns
out clean (no violation to swap), the only legal move left is `compare` — and the first version
hardcoded that probe to `{"i": 0, "j": 1}`, throwing away which gap it had just moved on to. The
*next* round's recovery code above only ever looks at the last **swap** — and if no swap has
happened yet anywhere in the run, `last_swap` is still `None`, so it recomputes the exact same
starting gap it began with. Clean gap, hardcoded probe, `None` recovery: the handler re-derives the
identical starting gap forever, checks it, finds it clean (it already know it was), and emits the
identical `compare` again. Nothing in that cycle ever changes. It isn't a slow convergence — it's a
genuine infinite loop that a fixed round budget merely disguises as "ran out of budget."

The fix has nothing to do with comb sort itself — it's a harness-design fix. **Every move a handler
sends is the only persistence mechanism this contract gives you**, not just the moves that change
`array`. So the `compare` probe has to carry the same information a `swap` would:

```python
last = history[-1] if history else None                       # last entry, ANY action
gap = abs(last["j"] - last["i"]) if last is not None else max(1, int(n / 1.3))
...
next_gap = max(1, int(gap / 1.3))
print(json.dumps({"action": "compare", "i": 0, "j": next_gap}))  # encodes next_gap, not a guess
```

Two changes, both required: read the *last entry of either type*, and make the `compare` probe's
own `i`/`j` equal to the gap being tried rather than a placeholder. With both in place, gap
information survives every round regardless of which action produced it, and there's no state left
that can silently get lost.

## Step 3 — the numbers, re-run after the fix

Same five seeds, same `dryrun.py`, after the fix:

```
seed 1: rounds=17 comparisons=5 swaps=11 faults=0 sorted=True
seed 2: rounds=13 comparisons=5 swaps=7  faults=0 sorted=True
seed 3: rounds=19 comparisons=5 swaps=13 faults=0 sorted=True
seed 4: rounds=20 comparisons=5 swaps=14 faults=0 sorted=True
seed 5: rounds=17 comparisons=5 swaps=11 faults=0 sorted=True
```

Every one of ten further random seeds (six shown, four more run the same way), a reversed
24-element array, a 24-element random array, a five-element all-duplicates array, and the
singleton/pair edge cases all finished with `faults=0` `sorted=True`. Determinism held (same array,
run twice, byte-identical apart from `wallClockMs`). The `correction` probe came back `NOTE`, the
same "never actually sends an invalid move, so it never receives a real correction" case [Change
the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) already explains.

Head-to-head against `handlers/reference-sorter.sh` (real insertion sort), five random 16-element
seeds, budget disabled:

```
| Seed | comb-sort rounds | reference rounds |
|------|-------------------|-------------------|
| 1    | 32                | 70                |
| 2    | 27                | 49                |
| 3    | 29                | 66                |
| 4    | 25                | 65                |
| 5    | 29                | 78                |
```

Consistently under half the reference sorter's rounds, on every seed tried. This is the direct
payoff of the lever [Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }})
identified but didn't build: a gap-`swap` isn't limited to fixing one adjacent inversion at a time
the way every handler in that tutorial was. A single `swap(i, i+gap)` can resolve the relationship
between `array[i]` and every element strictly between the two positions as well as the pair itself,
so large-distance disorder gets cleared in one move instead of being walked there one slot per
round. That's a real, measured difference in `roundsUsed` — not a difference in which algorithm
"sorts better" in the abstract; both handlers reach a correctly sorted array every time. It's a
difference in what this specific contract counts as a round, and non-adjacent `swap` is the one
lever in it that adjacent-only designs can't reach.

## Step 4 — what verification means without a separate spec to check against

[Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }})'s strongest check
was fuzzing generated code against an independently-written reference implementation of its own
spec — proving the code matches *that spec*, not merely "some correct sort." That check doesn't
have an equivalent here: writing the handler by hand means the code and the design **are** the same
artifact: there's no separate spec document to diff the implementation against. That's a real
trade-off of skipping the skill, not an oversight — the skill's generate-then-verify-against-the-spec
loop exists precisely to catch the gap between what you meant and what got written, and the manual
path has no such second, independent description to check against. What you get instead is exactly
what this tutorial relied on throughout: `dryrun.py` run across enough real, varied arrays —
random, reversed, duplicate-heavy, and the tiny edge cases — to catch a real bug (Step 2) by running
the thing, not by reasoning about it in the abstract. That's a materially weaker guarantee than
Step 4 of the previous tutorial, and worth knowing going in if you write a handler this way.

## Once it's verified: go live

Same self-service path as the other tutorials — a public waiting room, no operator file edit, no
skill required either. See [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}).
