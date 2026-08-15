---
title: The harness is the variable
description: Why the same model produces measurably different participants, what the harness is made of, and what a skill actually manipulates.
order: 1
---

# The harness is the variable

Every participant in this arena can call the same model. The arena still separates them clearly and
reproducibly. That gap is the entire subject, and this page is the argument for why it exists —
with the measurements that back it.

## The claim, and what it is not

The claim is **not** that prompt wording is powerful. It's the opposite: in this pipeline the model
is asked once, at build time, and never consulted again during a run. Everything that decides how a
participant behaves in the arena sits *around* the model call.

Call that surrounding structure the **harness**. Here it has four layers, and each one is a place
where a participant can differ from another without the model changing at all:

| Layer | Where it lives | What it decides |
|---|---|---|
| The contract | `participants/CLAUDE.md` | what a move may even be |
| The specification | your `AGENTS.md` | what strategy is being asked for |
| The generation step | `generate.sh` | how the spec becomes code, and what "usable" means |
| The verification gates | `dryrun.py` flags | which properties are provably true before it goes live |

A skill is what touches all four in one loop. That's the practical definition worth carrying away:
**a skill is not a better prompt, it's an automated grip on the harness.**

## The evidence

Three participants, one array, the same underlying model:

| Participant | Strategy | Rounds | Swaps |
|---|---|---|---|
| comb sort | shrinking gap | **20** | 25 |
| adjacent A | insertion-like | 40 | **39** |
| adjacent B | bubble-like | 40 | **39** |

Two things in that table are worth more than the headline.

First, comb sort halves the rounds. Nothing about the model differs — only the strategy the harness
pinned down.

Second, and more instructive: **the two adjacent strategies land on exactly the same swap count.**
Not close — identical, and identical to the inversion count of the starting array. That's not luck,
it's a bound. An adjacent swap removes at most one inversion, so any strategy restricted to adjacent
moves pays exactly `inversions` swaps no matter how cleverly it chooses them. Two different
algorithms, generated from two different specifications, are indistinguishable on that metric because
the contract made them indistinguishable.

## The contract decides what can even be seen

Which sets up the result that surprises people, and the reason
[Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) exists: **merge sort
is not faster here.**

Merge sort's advantage over insertion sort is in comparison count. But the round input hands the
handler the *entire array*, so comparisons are free — you can read every element without spending
anything the arena measures. Merge sort's edge is real; this contract just doesn't have a dimension
where it shows up. The scoreboard counts rounds and swaps, so the only lever that moves it is how far
a single move can jump.

That generalizes past sorting, and it's the uncomfortable half of the lesson: **a harness that
measures the wrong dimension makes a better approach look equal or worse.** Before optimizing
anything against a metric, check that the metric can see the quality you're trying to add.

## The specification's vocabulary is part of the harness

The four layers above make the harness sound like structure — files, scripts, checks. One measured
result says it reaches further down than that, into word choice.

Two specifications produced participants that spent half their rounds on `compare` moves buying
nothing. Both specifications used the word *comparing*, and that turned out to be what the
generation step keyed on. Rewriting
one of them with the same meaning and no contract vocabulary — "whenever an element is larger than
the one directly to its right" — dropped compare moves to zero, without any instruction telling the
generated code to avoid them.

Nobody wrote a rule connecting the two. The specification said `comparing`, the contract has a
`compare` move, and the generation step joined them up. That is the harness acting exactly as
designed, on an input nobody thought of as an input.

**This is not the "prompt wording is powerful" claim this page opened by rejecting** — and the
difference is the whole point. Nothing here got better because the phrasing was more persuasive. A
word in one layer collided with a move type in another, and a third layer wired them together. The
effect is mechanical, reproducible, and visible in the scoreboard; you can predict it from the
contract without knowing anything about the model. That's a harness property. "Say it more nicely"
is not.

The practical form of this is in
[Change the skill]({{ '/how-to/change-the-skill/' | relative_url }}), with the three-arm measurement
that separates the wording effect from an explicit prohibition — and the later correction showing
that what actually matters is framing rather than the word itself. The conceptual form belongs here:
**everything the generation step reads is harness**, including the words you thought were just
description.

## Why the model is asked once, not every round

The obvious design would be to call the model each round and let it pick the move. The arena did work
that way, and [why generate, not decide]({{ '/explanation/why-generate-not-decide/' | relative_url }})
covers the switch in full. Two consequences matter for this page:

**Determinism becomes checkable.** Fixed code plus a fixed seed gives byte-identical results apart
from wall-clock. You can put a bound in a test — `swaps == inversions` — and have it mean something.
With a live decision per round, the same seed can produce a different run, and every check degrades
into a sample.

**Failure moves to build time.** A model that misunderstands the strategy now fails during
`generate.sh`, where a retry costs seconds and nobody is watching, instead of mid-run in front of an
audience. This is why the scaffold ships a working handler: the participant's *first* success is
deterministic, and the model call only ever upgrades something that already works.

## What a skill actually manipulates

Concretely, `sort-arena-harness` does four things, and none of them is "phrase the request better":

1. **Writes the specification** — turns a plain-language strategy into `AGENTS.md`, the file that is
   the actual source of the participant.
2. **Runs the generation** — `generate.sh`, which validates the result with `py_compile` and retries
   on a failure rather than handing you a broken artifact.
3. **Runs the gates** — the selftest, a seeded dry run, and the property checks.
4. **Feeds violations back into the specification** — not into the generated code.

Step 4 is where most of the learning is, and where most people go wrong on the first attempt. When
`--require-adjacent` reports a violation, the tempting fix is to edit `generated/handler.py`. That
edit works exactly until the next regeneration overwrites it. The generated file is a build artifact;
`AGENTS.md` is the source. Fixing the artifact is fixing a symptom, and the harness will re-create the
cause.

The discipline that makes the loop converge:

- **One change per iteration.** Add two sentences and you won't know which one did the work.
- **Read the violation, not the code.** The check names what it saw; the spec is where that belongs.
- **The exit code is the referee.** `0` passes, `1` violates. Not a paragraph you interpret — a value
  a hook can branch on, which is what makes the loop automatable rather than manual.

## Why sorting

Sorting is not the interesting part. It was chosen because its correctness is *fully checkable*: an
array is sorted or it isn't, `swaps == inversions` holds or it doesn't, a move is adjacent or it
isn't. That makes it possible to state a property in advance, mechanically test it, and get an
unambiguous verdict.

That is the transferable move. Most real tasks don't hand you a total correctness oracle — but most
have some property you can pin down and check automatically, and the harness discipline is worth
exactly as much as the properties you're willing to make checkable. A participant here is a small,
honest model of that: a specification, a generation step, and a gate that says no.

## What this arena does not show

Stated plainly, because a training example that oversells itself teaches the wrong reflex:

- **It doesn't rank models.** Every measurement here holds the model fixed on purpose. Nothing on
  this site is evidence that one model sorts better than another.
- **It doesn't show harness effects on open-ended work.** Sorting has a correctness oracle. Tasks
  without one are harder in a way this example deliberately avoids.
- **Round counts are contract-relative.** The 20-versus-40 result is a fact about *this* contract.
  Change what a move may be and the ordering can change with it — which is, after all, the point.
