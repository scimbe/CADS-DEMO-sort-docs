---
title: Bring your own participant online
description: Turn a plain-language sorting idea into a reliable competing service, and learn what "reliable" actually requires from the harness around it.
order: 1
---

# Bring your own participant online

The goal of this tutorial is not "get a model to sort a list." It's to understand **what has to
change in the harness around a model to make the service it produces reliable** — not just
correct once, but correct the same way every time it's called. That's what "reliable" means here,
and it's the whole point of the exercise.

## Step 1 — say what you want, in your own words

Open this repository with your coding CLI (Claude Code, or another that reads `AGENTS.md`) and
run the **`sort-arena-harness`** skill. It will ask you a few things — a short id for yourself, a
display name, and, most importantly, your strategy in plain language: what should your sorter do,
and why do you think it's a good idea? You don't need an algorithm name. "Fix whichever pair is
nearest the front" is a completely fine answer.

You don't write any code at this step, and you don't hand-edit a prompt. You describe the idea;
the skill turns it into a real specification and, from that, into a real program.

## Step 2 — expect the first attempt to be checked, not trusted

The skill generates a program from your spec, then actually verifies it: does it speak the wire
contract at all, and does it sort a real array correctly — twice, on the same input, to confirm
it's genuinely the same reliable code both times, not a lucky guess. You are not expected to read
the generated code yourself. The checks are what "it works" means here.

Sometimes the first attempt fails a check. That is not a bad outcome — it's the most useful moment
in the whole exercise, and it's covered next.

## What a real failure looks like, and what actually fixes it

Here is a real one, from building the `bubble-sort-claude` participant this project ships with.
The strategy was simple: visit adjacent pairs left to right, swap if out of order, repeat. The
first version of the spec was dry-run twice and failed both times:

| Run | Array | Result |
|---|---|---|
| 1 | `[5,3,8,1,9,2]` | budget exhausted, never sorted |
| 2 | `[18,60,61,29,26,25]` | budget exhausted, never sorted |

Both failed the identical way: at specific positions, the generated behavior repeatedly compared
instead of swapping, even though the array plainly showed an out-of-order pair right there. The
tempting fix is "run it again and hope for a cleaner result." That fix was **not** used, because
it doesn't actually address anything — an unreliable spec produces unreliable code however many
times you regenerate it.

The real fix was to look at *why* the model kept getting that one decision wrong. The spec said
"remember where you are in a pass" — but the handler is invoked fresh every single round with no
memory. The instruction was accurate in spirit but didn't say *how* to actually reconstruct
"where you are" from what a stateless call can actually see. Once the spec was rewritten to say
that explicitly — reconstruct the cursor from the single most recent entry in the round history,
check the real array directly rather than trying to remember whether a pass had a swap in it — the
identical two arrays that failed before both converged cleanly, every time, on retest.

That's the actual lesson: **a generated service is only as reliable as the spec's answer to "what
does the model actually know at the moment it has to decide this?"** A vague spec produces code
that's right most of the time and silently wrong at the edges. A spec that states its assumptions
explicitly enough to be checked produces code that's right the same way every time. See
["Why generate code, not live decisions"]({{ '/explanation/why-generate-not-decide/' | relative_url }})
for the full evidence trail this example is drawn from, including what the same model did with
*no* harness at all, and what changed at each step in between.

## What you'll have when this is done

A short, real report: pass or fail on the contract check, pass or fail on the dry run, and — if
either failed — what about your spec the failure points at. When both pass, you have a program
that runs in milliseconds and behaves identically on every call, not a live decision you have to
hope goes well each round. That reliability is the actual deliverable of this tutorial, not the
sorting itself.

## Once your service is reliable: go live

Turning your generated handler into something `sort.bunsenbrenner.org` actually calls is a
separate, mechanical, one-time step: mint an account, bring up a tunnel, register your handler.
None of that teaches you anything about harness reliability, so it's kept out of this tutorial —
see [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}) for the full
walkthrough, real screenshots included.
