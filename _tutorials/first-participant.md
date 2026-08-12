---
title: Bring your own participant online
description: Turn a plain-language sorting idea into a reliable competing service, step by step, including exactly what to type.
order: 1
---

# Bring your own participant online

The goal of this tutorial is not "get a model to sort a list." It's to understand **what has to
change in the harness around a model to make the service it produces reliable** — not just
correct once, but correct the same way every time it's called. That's what "reliable" means here,
and it's the whole point of the exercise.

This walkthrough was validated by actually running it, start to finish, on a clean machine with
nothing pre-installed — a fresh Ubuntu container, a fresh clone of this repo, nothing assumed.
Every command below is the literal command that was typed, and the sample output is real, not
paraphrased.

## Before you begin

- **[Claude Code](https://claude.com/claude-code)**, installed and signed in. This tutorial is
  specifically about the `sort-arena-harness` **skill**, which is a Claude Code feature — it is
  not available if you're driving Codex, Gemini CLI, or opencode. Using one of those instead? Skip
  to [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }})'s "Manual
  fastest start" — you'll write `AGENTS.md` by hand from a template rather than being guided
  through it, but you end up in the same place.
- **`git`**, to clone this repo.
- A few minutes and a small amount of real API cost. A full run — spec, code generation, three
  verification checks — took about 5 minutes and roughly $2 of model usage in the run this
  tutorial is based on (Claude Opus pricing; a cheaper model will cost less and may take longer).

## Step 1 — clone the repo and start Claude Code inside it

```bash
git clone https://github.com/scimbe/CADS-DEMO-sort.git
cd CADS-DEMO-sort
claude
```

This matters more than it looks: Claude Code only discovers this repo's `sort-arena-harness`
skill (`.claude/skills/sort-arena-harness/SKILL.md`) when it's running **inside** a checkout of
this repo. Starting `claude` from your home directory, or any other folder, won't find it.

## Step 2 — invoke the skill

At the prompt, type:

```
/sort-arena-harness
```

This is a real slash command — Claude Code resolves it against this repo's own
`.claude/skills/` directory the moment you type it, no separate install step. It replies by
asking for the three things it needs, one at a time if you don't already have them ready:

- **A participant id** — a short slug, e.g. `insertion-fan`.
- **A display label** — the name shown on the leaderboard.
- **Your strategy, in your own words** — no algorithm name required. In the validation run behind
  this tutorial, the answer given was:

  > scan the array left to right comparing each adjacent pair; if a pair is out of order, swap it
  > immediately; once a full left-to-right pass produces no swaps at all, the array is sorted and
  > I'm done

  If you already know your id/label/strategy, you can save a round-trip by putting all three in
  your first message instead of waiting to be asked — either way works.

You do not write any code at this step, and you do not hand-edit a prompt. You describe the idea;
the skill turns it into a real specification (`participants/<your-id>/AGENTS.md`) and, from that,
into a real program (`participants/<your-id>/generated/handler.py`).

## Step 3 — expect the first attempt to be checked, not trusted

The skill generates a program from your spec, then actually verifies it: does it speak the wire
contract at all (`handler.sh --selftest`), does it sort a real array correctly, and does it do so
**twice** on the same input — confirming it's genuinely the same reliable code both times, not a
lucky guess. You are not expected to read the generated code yourself; the checks are what "it
works" means here.

Here is the real report from the validation run, verbatim, because seeing the actual shape of a
passing report is more useful than a description of one:

```
Done — `docs-tutorial-test` is built and verified.

## Pass/fail report

**All three contract criteria passed, on the first generation.** No spec-tighten-and-regenerate
cycle was needed here.

| Check | Result |
|---|---|
| `handler.sh --selftest` | **OK** |
| Criterion 1 — format, zero faults | **PASS** (`faults=0`, `wrongDone=0` on every run) |
| Criterion 2 — terminated correctly in budget | **PASS** (`sorted=True` on every run) |
| Criterion 3 — no regression on `correction` | **PASS** (`swap 0,1` without → `compare 0,1` with) |
| Determinism (same array, run twice) | byte-identical output apart from `wallClockMs` |

Main dry run, `--seed 42 --len 8`, array `[82, 15, 4, 95, 36, 32, 29, 18]` → sorted correctly in
**43 rounds** (25 comparisons, 17 swaps), ~3.2s for the whole run. Also clean on: already-sorted,
reversed, heavy duplicates, n=2, and n=21/22/24 (301/278/309 rounds).
```

Passing on the first try is not the common case worth designing this tutorial around — a real
failure teaches you more, and is covered next. But when it does pass first try, this is what that
looks like: real numbers, real shapes tested, no hand-waving.

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

A directory at `participants/<your-id>/` — spec, generated code, a handler wrapper, and a README
recording the verification report — plus a short, real report: pass or fail on the contract
check, pass or fail on the dry run, and, if either failed, what about your spec the failure points
at. When both pass, you have a program that runs in milliseconds and behaves identically on every
call, not a live decision you have to hope goes well each round. That reliability is the actual
deliverable of this tutorial, not the sorting itself.

**One thing worth checking before you trust a generated README's own claims about how to join the
arena:** verify it points at [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }})
and not at hand-editing a bridge config file — the self-service waiting room described there is
the only real path today; anything else in a generated README is stale.

## Once your service is reliable: go live

Turning your generated handler into something `sort.bunsenbrenner.org` actually calls is a
separate, self-service step — no CLI, no operator file edit, just a public waiting room. See
[Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}) for the full
walkthrough.
