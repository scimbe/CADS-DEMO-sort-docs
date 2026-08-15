---
title: Sort Arena — Docs
description: Documentation for CADS-DEMO-sort — same task, same contract, different harness.
---

# Sort Arena

**CADS-DEMO-sort** ([GitHub](https://github.com/scimbe/CADS-DEMO-sort)) is a CADS-Tunnel reference
pipeline where nothing in the repo itself sorts anything. Every participant is a separate process,
reached over the same real move contract, and the whole point is watching how differently the
*exact same underlying model* behaves depending on what harness (system prompt, skill, algorithm
coaching) is wrapped around it — not the prompt text.

The live arena is at **[sort.bunsenbrenner.org](https://sort.bunsenbrenner.org/)** — a real,
running visualization, not a static mockup.

## Start here

**New here? [Five seconds to your first sorter]({{ '/tutorials/five-seconds/' | relative_url }}).**
Clone, copy four commands, and a contract-verified participant is sorting — before any model call,
so nothing can fail on you. Then one spec sentence makes it your own strategy, and the same page
shows you why the harness, not the model, is what actually differs. Everything below goes deeper.

- [Bring your own participant online]({{ '/tutorials/first-participant/' | relative_url }}) — the
  `sort-arena-harness` skill loop, literal commands included: describe a strategy in plain
  language, get real generated code back, verify it before it goes live.
- [Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) — the same skill,
  a genuinely different algorithm (merge sort), and a real, measured answer to what this specific
  harness actually costs an `O(n log n)` algorithm versus a simpler one.
- [Non-adjacent swaps]({{ '/tutorials/non-adjacent-swaps/' | relative_url }}) — skip the skill
  entirely and write a comb-sort handler by hand from the move contract alone; catches a real
  infinite loop along the way.
- [Coaching a strategy]({{ '/tutorials/coaching-a-strategy/' | relative_url }}) — a real bug from
  before this repo's own harness migration, and why it disappeared once a live per-round decision
  became fixed, generated code instead.
- [Partition mode]({{ '/tutorials/partition-mode/' | relative_url }}) — a live run where three
  participants finish their segments perfectly and the arena still, correctly, reports the whole
  array unsorted.
- [Evolve to bubble sort]({{ '/tutorials/evolve-to-bubble-sort/' | relative_url }}) — change the
  *harness itself*: pin down what "is a bubble sort" means as checkable properties, then iterate a
  plain-language spec until the generated code provably has them.

Want all of it on your own machine first? [Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }})
brings up the same bridge locally in about a minute — every tutorial works against it unchanged.

## If you're here to learn the harness, not to sort

Read these five in order. They're the argument, and each one is grounded in measurements from this
arena rather than assertion. The fourth lets you check whether it landed; the fifth is what
happens when the measurements behind all of it are wrong:

1. [Five seconds to your first sorter]({{ '/tutorials/five-seconds/' | relative_url }}) — get
   something running, then see the same model produce measurably different participants.
2. [The harness is the variable]({{ '/explanation/the-harness-is-the-variable/' | relative_url }}) —
   the four layers a participant can differ in, and why the contract decides which of an algorithm's
   strengths can show up at all.
3. [Change the skill, not just the strategy]({{ '/how-to/change-the-skill/' | relative_url }}) —
   edit the driver. The worked example is a real defect where the contract already said the right
   thing and nothing checked, so it happened in 20 of 27 generations anyway.
4. [Check yourself]({{ '/tutorials/check-yourself/' | relative_url }}) — six predictions, each with
   a command that settles it. One of them catches a claim this site itself got wrong.
5. [When the measurement lies]({{ '/explanation/when-the-measurement-lies/' | relative_url }}) —
   seven wrong claims published here in one day, and what every one of them had in common. Two of
   them fail in opposite directions: one instrument missed real cases, the other invented them. If
   you build gates on measurements, the measurement is load-bearing too.

## Sections

- **[Tutorials]({{ '/tutorials/' | relative_url }})** — learn by doing, start to finish.
- **[How-to guides]({{ '/how-to/' | relative_url }})** — accomplish a specific task.
- **[Reference]({{ '/reference/' | relative_url }})** — look up an exact fact (the move contract, field shapes, bounds).
- **[Explanation]({{ '/explanation/' | relative_url }})** — understand why the arena is built this way.
