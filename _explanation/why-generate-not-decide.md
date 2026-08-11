---
title: Why generate code, not live decisions
description: The real evidence that led Sort Arena away from asking a model to decide each move live, toward asking it to write the sorting program once.
order: 2
---

# Why generate code, not live decisions

Sort Arena's harness asks a model to **write a sorting program once**, then runs that program for
every round. It did not start that way. Early participants called the model live, once per round,
and asked it to decide the next move on the spot. This page is the evidence for why that changed.

## The same model, four levels of control

To separate "the model isn't capable enough" from "the harness isn't giving it enough to work
with," the exact same underlying model was called four times against the same round input, each
time with a different amount of structure wrapped around it.

Round input used for stages 1–2 (a plain array, no protocol machinery yet):

```json
{"round":1,"array":[5,3,8,1,9,2],"history":[],"budgetRemaining":43,"mode":"solo","you":"stage-N-test"}
```

**Stage 1 — no harness at all.** Zero system prompt, raw JSON in. Real output, reproduced
verbatim:

````
```json
{
  "round": 1,
  "action": "swap",
  "indices": [0, 1],
  "reason": "5 > 3 at positions 0-1; first out-of-order adjacent pair (bubble-sort pass)",
  "arrayAfter": [3, 5, 8, 1, 9, 2],
  "sorted": false,
  "budgetRemaining": 42
}
```
````

A fault: markdown-fenced, extra prose, `indices` instead of `i`/`j`, invented fields. But the
*reasoning* was completely sound — it correctly found the out-of-order pair and even named
"bubble-sort pass" unprompted. Capability was never the bottleneck; wire format was.

**Stage 2 — a one-line format hint.** "Reply with JSON describing your move, keep it short." Real
output:

```
{"move":"swap","i":0,"j":3,"array":[1,3,8,5,9,2],"done":false}
```

Progress: valid, unfenced JSON. Still faulty: `move` not `action`, invented `array`/`done` fields.
`i=0, j=3` is also a non-adjacent swap — legal under the general contract, but not a valid
bubble-sort step. Vague steering bought partial format compliance, not contract compliance.

**Stage 3 — the full protocol contract, no strategy coaching.** The exact wire contract spelled
out (field names, three actions, bounds), nothing about *how* to sort. Real live run
(`len=6`, `SORT_BUDGET=20`):

| Metric | Value |
|---|---|
| initial array | `[17, 22, 62, 27, 12, 4]` (10 inversions) |
| final array | `[4, 17, 12, 22, 27, 62]` (1 inversion) |
| `finishedCorrectly` | **false** |
| `faults` | **0** |
| `roundsUsed` | 20 (entire budget) |

Zero faults — the model stayed inside the contract perfectly. But at one inversion from sorted it
emitted `{"action":"done"}`, was wrong, and repeated the *identical wrong claim thirteen times in
a row*, burning the rest of its budget. None of those replies were faults — a well-formed `done`
is valid; being wrong about it isn't a protocol violation. Contract compliance and actually
succeeding at the task turned out to be two different axes.

**Stage 4 — full contract plus a coached strategy.** Adding a real strategy spec
(`participants/bubble-sort-claude/AGENTS.md`, adjacent-pair passes, direct sortedness check) fixed
the specific stall from stage 3. But it surfaced a new, more interesting failure first.

## A genuine stall, and the fix that actually worked

The first version of the bubble-sort strategy, delivered as a hand-built `--append-system-prompt`
string, stalled twice on real dry runs:

| Run | Array | Budget | Result |
|---|---|---|---|
| 1 | `[5,3,8,1,9,2]` (8 inversions) | 30 | `rounds=30 faults=0 sorted=False` — budget exhausted |
| 2 | `[18,60,61,29,26,25]` (9 inversions) | 40 | `rounds=35 faults=5 sorted=False` — budget exhausted |

Both failed the same way: at specific cursor positions, the model repeatedly emitted `compare`
instead of `swap` despite the array visibly showing an inversion right there — reproduced
identically on two different arrays
([full transcripts, CADS-DEMO-sort#10](https://github.com/scimbe/CADS-DEMO-sort/issues/10)).

The instinct after a repeated deviation like that is to re-run and hope for a cleaner sample. That
was resisted. Instead the *harness itself* changed: every participant's strategy text moved out of
a hand-built prompt string and into a real `AGENTS.md` file that coding CLIs discover natively from
the working directory, with a shared `participants/CLAUDE.md` adding an explicit, checkable
**contract criterion** for format and termination (see
["Structuring a harness instruction"]({{ '/explanation/instruction-structure/' | relative_url }})
for the general principle). Re-run with the *identical strategy content*, delivered the new way,
against the same two seeds that stalled before:

| Run | Array | Budget | Result |
|---|---|---|---|
| 3 | `[5,3,8,1,9,2]` (8 inversions) | 30 | `rounds=17 faults=0 sorted=True` |
| 4 | `[18,60,61,29,26,25]` (9 inversions) | 40 | `rounds=17 faults=0 sorted=True` |

Both previously-stalling seeds converged cleanly. Two runs each is not enough to prove the delivery
mechanism *caused* the fix — but it is the right *shape* of response to a repeated harness failure:
change what the harness structurally provides, then measure again.

## What live decisions actually cost, at scale

Once the coached strategy worked, it was run live on `sort.bunsenbrenner.org` against a real
12-element array, to completion:

`comparisons: 29, swaps: 31, faults: 0, rounds: 61, wall: 506.5s, sorted: yes`

Zero faults across 61 real model calls in a row is the stage-3 contract-compliance story holding
up under sustained live load. But it took **506.5 seconds** for a
12-element array, because every single one of those 61 rounds was a real, ~8-second model call
made live, in the middle of the run. That cost is structural, not a bug: it scales with
`roundsUsed`, and it cannot be prompted away (a faster-instructed call was measured *slower*, not
faster, in this environment).

## The conclusion this led to

Stages 1–3 show that giving a model something *checkable* to fail against — first the wire
contract, then an explicit strategy with a stated termination criterion — is what actually moves
outcomes, not "try harder" prompting. Stage 4 shows a coached live-decision harness can reach zero
faults and full correctness. But it still pays a live model call, every round, forever, and that
cost cannot be designed around from inside the live-decision model.

The fix is architectural, not a tuning knob: ask the model to do the same disciplined,
checkable work it was already good at — writing something that satisfies a stated contract — but
do it **once**, as a real program, instead of live, every round. Verification then checks the
*program*, the same way stage 3/4 checked live replies, except now it only has to happen once, and
a failure is caught before the arena ever sees it instead of live in front of everyone. That's the
harness Sort Arena runs today — see the [Sort Arena harness skill]({{ '/tutorials/first-participant/' | relative_url }})
for the guided path through it.
