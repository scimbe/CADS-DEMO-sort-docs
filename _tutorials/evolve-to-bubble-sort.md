---
title: Evolve the harness — from "it sorts" to "it IS bubble sort, provably"
description: Use checked properties, a goal line, and a verification hook to steer your harness toward a named algorithm, reproducibly.
order: 2
---

# Evolve the harness — from "it sorts" to "it IS bubble sort, provably"

[The first tutorial]({{ '/tutorials/first-participant/' | relative_url }}) ends with a handler
that reliably sorts. This one asks for something stricter: a harness that produces a **specific,
named algorithm on demand** — and can *prove* it did, mechanically, every time. The exemplar
target is bubble sort, because its identity is fully capturable in two checkable properties.
The transferable lesson is bigger than sorting: it's how you turn "the model usually writes
roughly what I meant" into "the spec guarantees the property, and a check enforces it" — the
difference between prompting and engineering a harness.

Everything below runs locally against `dryrun.py` — no join, no tunnel, no API surprises beyond
the generation calls themselves. You need a participant from the first tutorial (any strategy —
in fact, a *non*-bubble strategy makes the exercise more instructive).

## Step 1 — State the target as checked properties, not vibes

"It's bubble sort" decomposes into two properties a referee can check without reading the code:

1. **Adjacency**: every move — compare or swap — touches neighbours only (`j == i + 1`). That is
   *the* defining constraint; any strategy that ever jumps a value straight into place is
   something else, whatever its README says.
2. **Swap-optimality among adjacent strategies**: total swaps equals the start array's
   **inversion count** (pairs that are out of order). Each adjacent swap can fix exactly one
   inversion, and a bubble sort that never swaps an already-ordered pair achieves exactly that
   bound. More swaps means wasted or undone work — a cursor that loses its place, a spec that
   lets the model "re-check" by re-swapping.

`dryrun.py` checks both natively. This is your **goal line**:

```bash
python3 dryrun.py participants/<your-id>/handler.sh --seed 42 --quiet \
  --require-adjacent --require-optimal-swaps
```

Exit 0 with `property checks passed: adjacent, optimal-swaps` is the target. Run it now against
your existing participant. If your first tutorial produced anything direct-placement-flavored,
you'll see the honest starting point instead:

```
property violation: round 2: swap 2,7 touches a NON-adjacent pair (|j-i| = 5, must be 1)
property violation: swaps (5) != initial inversions (17) -- an adjacent-swap strategy that
  never swaps an ordered pair uses exactly one swap per inversion; the surplus is wasted or
  undone work
```

Those two lines are real output (from a selection-sort-style handler against `--seed 42`). Note
what they give you that "it didn't work" never could: each violation names the round, the move,
and the *rule* it broke — which is exactly the information you need for Step 3.

## Step 2 — Wire the goal line in as a hook, so it runs mechanically

The arena's own discipline is that verification is never optional: `handler.sh --selftest` and
the bridge's scoring run whether you remember them or not. Give your local loop the same
property. Two ways, pick one:

**A `verify.sh` next to your handler** (works with every agent CLI):

```bash
cat > participants/<your-id>/verify.sh <<'SH'
#!/usr/bin/env bash
set -e
cd "$(dirname "$0")/../.."
for seed in 42 7; do
  python3 dryrun.py "participants/$(basename "$(dirname "$0")")/handler.sh" \
    --seed "$seed" --quiet --require-adjacent --require-optimal-swaps
done
echo "VERIFIED: bubble-sort properties hold on both seeds"
SH
chmod +x participants/<your-id>/verify.sh
```

**Or a Claude Code hook**, so the check fires automatically after every regeneration — add to
`.claude/settings.json` in your clone:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Bash(*generate.sh*)",
      "hooks": [{ "type": "command", "command": "participants/<your-id>/verify.sh" }]
    }]
  }
}
```

Now a regenerated handler that regresses a property **fails loudly at the moment of
regeneration** — the hook is your referee, and you've just built the same structure the arena
itself uses: generation is cheap and fallible, verification is mechanical and non-negotiable.

## Step 3 — Iterate the spec, not the code

Run the goal line; read the violations; tighten `participants/<your-id>/AGENTS.md`; regenerate;
repeat. The mapping from violation to missing spec line is the actual exercise:

| Violation you see | What your spec failed to pin down |
|---|---|
| a non-adjacent swap | you never forbade direct placement — add "only ever touch positions `k` and `k+1`; never emit a move with `j != i+1`" |
| surplus swaps | the cursor loses its place between rounds — add an explicit **cursor reconstruction rule** (the handler is stateless; where does `k` come from each round? `history`'s last entry is the only memory you have) |
| swaps of ordered pairs | you never said what to do at a pair that's already in order — add "emit `compare`, then advance; a visited-but-unswapped pair is not wasted work, it's what bubble sort does" |
| a `done` that's wrong or late | your termination rule is derived from memory instead of evidence — add "check the *live array* for being fully non-decreasing; never track 'did this pass swap' through a capped history" |

Two rules for this loop, both learned the hard way in the first tutorial and non-negotiable here:

- **Never patch `generated/handler.py` by hand.** The handler is a *build artifact*; the spec is
  the source. A hand-patched artifact passes today and regresses on the next regeneration — the
  hook from Step 2 will catch that, but only if you let the spec stay the source of truth.
- **One violation, one spec change, one regeneration.** Batch-fixing hides which sentence fixed
  which property, and knowing that mapping *is* the lesson you're here for.

When you're stuck, `participants/bubble-sort-claude/AGENTS.md` in the main repo is the worked
reference — a spec that produces all the properties above. Consult it the way you'd check a
solution page: after your own attempt, to compare *which constraint you were missing*, not as a
copy source.

## Step 4 — Define done, and check reproducibility explicitly

You are done when the goal line passes **on two different seeds, twice each** (that's what the
`verify.sh` above encodes). The double run per seed matters: it proves the generated code is
genuinely deterministic, not a live judgment that went well once — the same distinction the
first tutorial's dry-run-twice check made, now promoted to a standing gate.

At that point, compare where you started and where you are:

- Before: a spec that produced *a* working sorter.
- After: a spec that produces **bubble sort specifically**, with a mechanical proof, from any
  regeneration, and you can point at the sentence that guarantees each property.

That spec-to-property mapping is what "a better harness" means, and it transfers: swap the two
`--require-*` flags for whatever checkable properties your next real-world harness target has —
the loop (state properties → wire a hook → tighten the spec → verify twice) is the same.

## Where to go next

- Take the evolved participant live: [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }})
  — sign in, get auto-approved, run your handler over the tunnel, and watch `swaps` equal the
  live run's inversion count on the real leaderboard (the bridge reports `inversionsOverTime`;
  your final swap total matching the start inversions is the same optimality property, measured
  end-to-end over a real Agent-Fabric channel).
- [Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) inverts the
  exercise: what does this same harness *cost* an algorithm (merge sort) whose natural shape
  fights the contract? Between the two you'll have seen both directions: steering generation
  toward a target, and discovering a target's real cost structure.

Prefer to try all of this without touching the live deployment? [Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }}) brings up the same bridge on your machine in about a minute — every command in this tutorial works against it unchanged.
