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

Measured across 27 usable generations from two different specifications: **20 of them emitted
`compare` moves that nobody asked for** — 5 of the first 11, then 15 of the next 16. Same model,
same specification text, different draw.

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

It's stated, it's passed to the model, and it happened anyway in **20 of 27** generations. **A
property that lives only in prose is not a property of the system.** That's the transferable lesson, and the
fix is to move it: prose → gate.

## First, the cheaper fix: one word

Before building anything, it's worth knowing *why* those compare moves appear — because the answer
is a rule you can follow for free, and it was not what anyone expected.

Both specifications that produced them contained the word **compare**:

> Scan the array left to right **comparing** each adjacent pair…

Three versions of that same strategy, semantically identical, same model, same time window:

| | Specification | Compare moves across runs |
|---|---|---|
| **A** | contains "comparing" | `175 175 175 175 0 175 175 175` |
| **B** | word removed, nothing else added | `0 0` |
| **C** | word removed, plus "never spend a move to inspect" | `0 0 0` |

**B is the arm that matters, and it was almost missed.** The first control was C — which changed two
things at once: it removed the word *and* added an explicit prohibition. That confounds the result,
so B was run to separate them: wording changed, no prohibition. Same outcome as C.

**Removing the word is enough; the prohibition adds nothing on top of it.** (Whether the word
itself is what matters, or how it is framed, is settled just below — the short answer is framing.)
Here is B, in full:

> Scan the array from left to right. Whenever an element is larger than the one directly to its
> right, swap that pair immediately. Once a full pass produces no swaps, the array is sorted and you
> are done.

Nothing about that says "don't emit compares." It just contains no word that maps onto a move type
in the contract.

So the rule, and it generalises past this arena:

> **Words in the specification that name things in the contract get taken literally.** `compare`,
> `swap`, `done` are move types here. Use them for moves you want, and different words for
> everything else — "when an element is larger than", not "compare the elements".

### The rule is about framing, not vocabulary

That shorthand — *the word is the cause* — turned out to be too simple, and the correction is more
useful than the original.

The rule ships in the skill now (`bddf750`), alongside the gate as a fourth verification step. Ten
invocations were measured against it: six with a compare-free strategy, four with one that says
"compare each adjacent pair … keep comparing" four times over. **All ten produced zero compare
moves**, at exactly 28 rounds — the optimum for the test array. Without the rule, the same
compare-free strategy produced 80 rounds and 52 compare moves.

That is the whole loop closed on itself: a defect found by measuring, a check built to catch it, a
rule to stop it earlier, both handed to the person who owns the file, and a re-measurement against
the shipped result. The rule catches it cheaply when it works; the gate catches it when the rule
doesn't.

But the specifications the skill wrote still contain the word — twelve times in the adversarial
case. It did not avoid the term. It disambiguated it, under a heading of its own:

> **Why "compare" the word is not `compare` the move**
>
> The round input already gives you the real, complete `array`. "Comparing" two elements — i.e.
> reading them and checking their order — costs nothing and is not the contract's `compare` move:
> that move exists for a live decision-maker who cannot otherwise see …

So the accurate statement is not "avoid the vocabulary". It is: **a move type mentioned neutrally,
inside a description of what the algorithm does, reads as a step to perform.** Removing the word
works. Negating it works. Explaining the difference works best — and that last one was the skill's
idea, not a rule anyone wrote.

The practical form: don't police words, state the distinction once. Then let the gate catch what
survives.

This is worth sitting with, because it is the harness thesis in miniature. Nobody wrote a rule
saying "emit a compare move per comparison." The specification said `comparing`, the contract has a
`compare` move, and the generation step connected them. **The vocabulary of your spec is part of
your harness** — and it costs rounds in the arena's own scoreboard without ever failing a check.

Which is exactly why the gate below is still worth building: a rule only holds while someone
remembers it.

## The gate now ships

When this page was written the check did not exist, so it showed a shell script that grepped
`dryrun.py`'s output. That is no longer necessary — `--require-no-wasted-compares` landed in
`0c81457`, in the shape of the two checks beside it:

```bash
python3 "$REPO/dryrun.py" ./handler.sh --seed 42 --len 12 --quiet \
  --require-adjacent --require-optimal-swaps --require-no-wasted-compares
```

Against a handler that spends rounds comparing:

```
rounds=80 comparisons=52 swaps=27 faults=0 sorted=True inversions=27
property violation: 52 compare move(s) spent -- the round input already carries the whole
array, so a compare reveals nothing the handler could not read directly and still costs a
round (rounds = comparisons + swaps + 1). Fix this in AGENTS.md, not in the generated
handler: state that the strategy reads the array directly and emits swaps only
exit=1
```

And against one that doesn't:

```
property checks passed: adjacent, no-wasted-compares, optimal-swaps
exit=0
```

The route from "I noticed something" to "the tool refuses it" was: measure it, build the check
locally to prove it catches the real case, propose it in the shape of the existing checks, and
hand it to whoever owns the file. The gate you build yourself is the argument; the merged one is
the outcome.

### Why it was worth the trouble

The strongest case for it turned up after it was written. Invoking `sort-arena-harness` with a
deliberately compare-free strategy — *"walk the array from left to right and whenever an element is
larger than the one directly to its right, swap that pair immediately"* — produced a specification
that **instructs the handler to emit compare moves**, and defends it:

> Else (already in the right order): emit `{"action": "compare", …}`. This still counts as visiting
> the pair and advancing the cursor next round; **it is not wasted, it is the honest cost of walking
> every adjacent pair** rather than jumping straight to the ones that need fixing.

It is wasted: the round input carries the whole array, so the ordering can be read without spending
a round. The result was **80 rounds where 28 suffice** — `52 + 27 + 1` — against an identically
specified competitor, for identical actual work.

Nothing in the skill's own instructions asks for this, and the contract passed to the model says the
opposite in as many words. The model wrote the justification by itself. **That is the case a rule
cannot catch and a gate can** — and it is why both belong in the loop, not one or the other.

## Wiring it into the skill

Add it to the verification step, next to the checks already there:

```markdown
3. Verify what came back, four ways: `handler.sh --selftest`, a local dry run run **twice**
   on the same array, `dryrun.py`'s `correction check`, and **`--require-no-wasted-compares`**.
   The round input already contains the whole array, so a compare move costs a round and
   reveals nothing.
```

And make sure step 4 covers it — the failure branch is the part that has to hold:

```markdown
4. If any check fails, tighten `AGENTS.md` and regenerate. Do not edit `generated/handler.py`;
   it is a build artifact and the next generation overwrites it. One change per iteration,
   so you know which sentence enforced which property.
```

And add the rule that stops it one step earlier, when the specification is written rather than
after it has been generated from:

```markdown
1. … when writing AGENTS.md, use `compare`, `swap` and `done` only for moves that should
   actually be played. For "check how two elements relate", use words that do not name a
   move type — the array is fully visible every round.
```

Now regenerate. The gate fails, you add the sentence it named, you regenerate again, and the gate
passes — and it will keep passing for every participant who uses this skill from now on, including
the ones who never read this page. The rule catches it cheaply when it works; the gate catches it
when the rule doesn't.

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
