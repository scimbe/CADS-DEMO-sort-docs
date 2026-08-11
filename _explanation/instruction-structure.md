---
title: Structuring a harness instruction — GOAL / CONTEXT / CONSTRAINTS / OUTPUT
description: A default frame for writing AGENTS.md/CLAUDE.md content so instructions stay testable and repeatable through the harness, plus the deeper principle it exists to serve — fix the harness, not the prompt.
order: 3
---

# Structuring a harness instruction

Every participant's `AGENTS.md`/`CLAUDE.md` in this repo is, underneath the specific strategy, an
instruction to a model that gets invoked over and over, unattended, with no human in the loop to
notice when it drifts. That constraint — nobody is reading each reply — is why *how* an
instruction is structured matters here more than it would in a one-off chat.

## The default frame this project uses

**GOAL — what comes out at the end, stated so it can be checked.**
Not "sort well" — "the array is fully sorted ascending, confirmed by directly checking every
adjacent pair, inside the round budget." A goal you can't check is a goal you can't harness-test.

**CONTEXT — the situation the instruction executes in.**
What the model actually has available each call: a fresh invocation with no memory of its own,
the real current `array`, up to 20 recent `history` entries, nothing else. Naming this explicitly
stops an instruction from silently assuming persistence or knowledge the harness doesn't provide.

**CONSTRAINTS — what binds the harness, not the strategy.**
The parts that are non-negotiable regardless of how clever the strategy gets: the exact JSON
shape, `i != j`, no prose. Constraints are where a violation is unambiguous and a reviewer (human
or automated) can check compliance without judgment calls.

**OUTPUT — the form the result must take, stated in advance.**
Not "a good answer" — "exactly one JSON object, one of three named shapes, nothing else on
stdout." Sometimes OUTPUT is a plan before code, a diff before a merge, or (as here) one strict
move object before the array changes at all.

This repo's own `participants/CLAUDE.md` and every `participants/<name>/AGENTS.md` follow this
shape, even where the headings aren't spelled out verbatim: the shared file states GOAL/CONTEXT/
CONSTRAINTS/OUTPUT once for every participant, and each participant's own file adds only the
strategy that decides *which* move to pick within that frame.

**This is a default, not the only structure.** Anthropic's own harness-design writeups describe
other real, working shapes — e.g. a *generator/evaluator* pair that negotiates a "sprint
contract" (what "done" looks like) before any work starts, rather than a single instruction
stating GOAL/CONTEXT/CONSTRAINTS/OUTPUT up front. Use whichever shape fits how the harness
actually calls the model. GOAL/CONTEXT/CONSTRAINTS/OUTPUT is this project's starting frame because
every participant here is a single stateless call, not a multi-turn negotiation — pick what your
own harness's call shape actually needs.

## The deeper principle: fix the harness, not the prompt

A structured instruction only pays off if a deviation from it changes something durable. The
principle, stated as its own worked example:

> **Scenario**: a reviewer notices `console.log` was used instead of the project's logger.
> That is not a one-off typo — it's an error *class*: nothing in the harness stopped the model
> from reaching for the wrong tool, and the same call, run again, could make the same choice
> again on a different file.
>
> **GOAL**: add ONE checkable rule to `CLAUDE.md`'s Conventions section that structurally
> prevents this error class. State it so a reviewer can apply it unambiguously — not "be careful
> with logging," but something a grep or a reviewer's checklist can test.
>
> **Do NOT repeat the task.** Re-running the same prompt and hoping for a cleaner roll fixes
> nothing about *why* it happened. **Change the harness instead** — add the rule, then let every
> future call inherit it for free.

The core of it: a deviation that happened once is not noise to retry past — it's a signal of a
rule the harness doesn't have yet.

### This repo's own real instance of the same principle

Building `bubble-sort-claude` produced a real, lived version of exactly this pattern, not a
hypothetical. Two dry runs of the coached bubble-sort strategy, delivered via an inline
`--append-system-prompt` string, stalled: the model repeatedly emitted `compare` instead of
`swap` at specific cursor positions despite the array visibly showing an inversion there,
across two different arrays. The instinct to re-run the same prompt hoping for a better sample
was resisted — see [CADS-DEMO-sort#10](https://github.com/scimbe/CADS-DEMO-sort/issues/10) for
the full data. Instead, the *harness itself* changed: every participant's strategy text moved out
of a hand-built prompt string and into a real `AGENTS.md`/`CLAUDE.md` file that Claude Code
discovers natively from the working directory
([CADS-DEMO-sort#11](https://github.com/scimbe/CADS-DEMO-sort/issues/11)), and the shared
`participants/CLAUDE.md` gained an explicit **Contract Criterion** section stating, in checkable
terms, what "done" means for any handler (format, termination, no-regression-on-correction) —
not just "seems to work." Re-run afterward with the identical strategy content, the same seed
array that stalled twice now converged cleanly: `rounds=17, faults=0, sorted=True`. One run is not
proof the architecture change caused it — see the tutorial's stage-4 section for the honest
caveat — but it is the right *kind* of response to a repeated deviation: change what the harness
structurally provides, then measure again, rather than repeating the same call and hoping.
