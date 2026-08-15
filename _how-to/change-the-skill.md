---
title: Change the skill, not just the strategy
description: Add a gate the skill did not have. The worked example is a real defect found by measurement, where the contract already warned and nothing checked.
order: 3
---

# Change the skill, not just the strategy

Every other page here has you steering the harness from the passenger seat: you write a
specification, the skill drives. This one is about editing the driver.

It matters because a skill is where a harness stops being advice and becomes enforcement. The
worked example below is a real defect, found by measuring this arena, in which the contract already
said the right thing and it changed nothing — because nothing checked.

## Where the skill lives

```bash
.claude/skills/sort-arena-harness/SKILL.md
```

152 lines of Markdown with two-field frontmatter:

```yaml
---
name: sort-arena-harness
description: Turns a plain-language sorting strategy into real, verified Sort Arena
  competitor code. Use when onboarding a new Sort Arena participant, or when changing
  an existing participant's strategy or target challenge.
---
```

The body is a procedure in four steps: write `AGENTS.md` from what the user said, run `generate.sh`,
verify three ways, and **on failure tighten the specification rather than re-roll the generation**.

That last step is the whole design. Everything below is a variation on it.

## The three levers, cheapest first

| Lever | What it changes | Cost |
|---|---|---|
| Add a **gate** | what counts as done | one script, minutes |
| Add a **spec line** | what the model is asked for | one sentence |
| Edit the **`description`** | *when* the skill engages at all | one line, easy to get wrong |

Most people reach for the second. The first is usually the one that pays, because a gate keeps
working after everyone has stopped paying attention.

The third is the subtlest: `description` is the trigger. It's matched against what the user is
doing, so widening it makes the skill fire on tasks it wasn't built for, and narrowing it makes it
silently never fire. A skill that doesn't run is indistinguishable from one that doesn't exist.

## The worked example

Measured across 11 generations from two different specifications: **5 of them emitted `compare`
moves that nobody asked for.** Same model, same specification text, different draw.

| | Rounds | Comparisons | Swaps |
|---|---|---|---|
| Variant A | **113** | 0 | 112 |
| Variant B | **288** | 175 | 112 |

Identical swap counts — identical *work* — and a factor of 2.5 in the number the arena ranks by.
The round accounting is exact: `rounds = comparisons + swaps + 1`.

You can see the same thing in the live arena, on a participant that has been running there since
long before this was measured:

![The hosted arena after a completed bubble-sort run: the scorecard reads comparisons 29, swaps 31, rounds 61, faults 0, and the move log shows consecutive compare moves]({{ '/assets/sort-arena-live-04-bubble-sort-complete.png' | relative_url }})

`29 + 31 + 1 = 61`. The move log makes it concrete — rounds 51 through 59 are nine consecutive
`compare` moves, and none of them changed the array. Nearly half this participant's rounds went on
reading values that were already in front of it.

Note what the arena does *not* do about that: the run is marked **finished correctly**, faults 0.
Nothing here is broken. It's a participant paying twice for a result it could have had for half,
and no check anywhere says so. That's the gap a gate fills.

A `compare` move buys nothing here, because the round input already hands the handler the entire
array. And this isn't an obscure corner: the contract states it outright, in
`participants/CLAUDE.md`, in the text that is passed to the model at generation time:

> `compare` reveals nothing you can't already read from `array` yourself, and still costs a round.

It's stated, it's passed to the model, and it happens anyway in roughly half of runs. **A property
that lives only in prose is not a property of the system.** That's the transferable lesson, and the
fix is to move it: prose → gate.

## Building the gate

`dryrun.py` already prints what you need. Turn it into a verdict:

```bash
#!/usr/bin/env bash
# Gate: fail if the handler spends rounds on compare moves.
set -u
out="$(python3 dryrun.py "$1" --seed 42 --len 20 --budget 600 --quiet 2>&1)"
n="$(printf '%s\n' "$out" | sed -n 's/.*comparisons=\([0-9]*\).*/\1/p' | head -1)"
if [ "${n:-0}" -gt 0 ]; then
  echo "GATE FAIL: $n compare moves. The round input already contains the whole array,"
  echo "           so each one costs a round and buys nothing. Add to AGENTS.md:"
  echo '           "Never emit compare moves; read the array directly and emit only swaps."'
  exit 1
fi
echo "GATE OK: no wasted compare moves"
```

Run it against both kinds of handler:

```
$ ./gate.sh participants/mysorter/handler.sh
GATE FAIL: 84 compare moves. The round input already contains the whole array,
           so each one costs a round and buys nothing. Add to AGENTS.md:
           "Never emit compare moves; read the array directly and emit only swaps."
exit=1

$ ./gate.sh participants/baseline/handler.sh
GATE OK: no wasted compare moves
exit=0
```

Three properties make that a gate rather than a print statement, and all three are worth copying:

- **It exits non-zero.** A human-readable warning gets skimmed; an exit code stops a pipeline.
- **It names the observation**, not just the verdict — `84 compare moves`, so you know how far off
  you are.
- **It says what to change, and where.** `AGENTS.md`, not the generated code. A gate that reports a
  violation without naming the source file trains people to patch the artifact.

## Wiring it into the skill

Add it to the verification step, next to the checks already there:

```markdown
3. Verify what came back, four ways: `handler.sh --selftest`, a local dry run run **twice**
   on the same array, `dryrun.py`'s `correction check`, and **`./gate.sh` — no compare
   moves.** The round input already contains the whole array, so a compare move costs a
   round and reveals nothing.
```

And make sure step 4 covers it — the failure branch is the part that has to hold:

```markdown
4. If any check fails, tighten `AGENTS.md` and regenerate. Do not edit `generated/handler.py`;
   it is a build artifact and the next generation overwrites it. One change per iteration,
   so you know which sentence enforced which property.
```

Now regenerate. The gate fails, you add the sentence it named, you regenerate again, and the gate
passes — and it will keep passing for every participant who uses this skill from now on, including
the ones who never read this page.

## What you just did

You didn't improve a prompt. You changed what the system is willing to accept, and the change
survives you. That is the difference between a harness and a suggestion, and it's the reason the
harness — not the model — is the variable worth working on.

Two caveats, so the technique doesn't get misapplied:

**A gate encodes a decision, so make sure it's the right one.** "No compare moves" is correct for
*this* contract, where the array is fully visible. Under a contract that hid the array, comparisons
would be the only way to learn anything and this gate would be actively harmful. Gates are
contract-relative; write down which contract yours assumes.

**Add them one at a time.** Two new gates in one iteration and a failure tells you nothing about
which property the specification is missing.

## Related

- [The harness is the variable]({{ '/explanation/the-harness-is-the-variable/' | relative_url }}) —
  why this is the lever that matters, with the measurements.
- [Evolve to bubble sort]({{ '/tutorials/evolve-to-bubble-sort/' | relative_url }}) — the same loop
  using the two property checks `dryrun.py` already ships.
