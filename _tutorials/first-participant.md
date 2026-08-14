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

- **An agent CLI.** This tutorial's screenshots and commands use
  [Claude Code](https://claude.com/claude-code), because Claude Code auto-discovers this repo's
  `sort-arena-harness` **skill** (`.claude/skills/sort-arena-harness/SKILL.md`) and turns it into
  the `/sort-arena-harness` slash command used below. The skill itself is plain markdown
  instructions driving ordinary scripts (`generate.sh`, `handler.sh --selftest`, `dryrun.py`) —
  nothing in it requires Claude. Driving Codex, Gemini CLI, or opencode instead? Two options:
  point your agent at the file — *"Read `.claude/skills/sort-arena-harness/SKILL.md` and follow
  it"* — and you get the same guided flow (opencode can even load Claude-style skills directly);
  or skip to [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }})'s
  "Manual fastest start" and write `AGENTS.md` by hand from a template. All three routes end in
  the same place.
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
reversed, heavy duplicates, n=2, and — **with `--budget 600`, since a coached bubble sort's real
worst case at this length is well past `dryrun.py`'s 200 default** (the GUI's own Round Budget
field defaults to 600 for exactly this reason) — n=21/22/24 (301/278/309 rounds). Running any of
these three at the 200 default reports `sorted=False`, budget exhausted, not a broken handler.
```

Passing on the first try is not the common case worth designing this tutorial around — a real
failure teaches you more, and is covered next. But when it does pass first try, this is what that
looks like: real numbers, real shapes tested, no hand-waving.

## What a real failure looks like, and what actually fixes it

Here is a real one, from this project's own history: building the `bubble-sort-claude`
participant this repo ships with. The strategy was simple: visit adjacent pairs left to right,
swap if out of order, repeat. The first version of the spec was dry-run twice and failed both
times:

| Run | Array | Result |
|---|---|---|
| 1 | `[5,3,8,1,9,2]` | budget exhausted, never sorted |
| 2 | `[18,60,61,29,26,25]` | budget exhausted, never sorted |

Both failed the identical way: at specific positions, the behavior the spec produced repeatedly
compared instead of swapping, even though the array plainly showed an out-of-order pair right
there. The full round-by-round record is preserved in
[CADS-DEMO-sort#10](https://github.com/scimbe/CADS-DEMO-sort/issues/10) — that issue, not this
page, is the primary artifact for this story. The tempting fix is "run it again and hope for a
cleaner result." That fix was **not** used, because it doesn't actually address anything — an
unreliable spec produces unreliable behavior however many times you re-sample it.

The real fix was to look at *why* the model kept getting that one decision wrong. The spec said
"remember where you are in a pass" — but the handler is invoked fresh every single round with no
memory. The instruction was accurate in spirit but didn't say *how* to actually reconstruct
"where you are" from what a stateless call can actually see. Once the spec was rewritten to say
that explicitly — reconstruct the cursor from the single most recent entry in the round history,
check the real array directly rather than trying to remember whether a pass had a swap in it — the
identical two arrays that failed before both converged cleanly, every time, on retest. The
rewritten spec is the one that ships today as
[`participants/bubble-sort-claude/AGENTS.md`](https://github.com/scimbe/CADS-DEMO-sort/blob/main/participants/bubble-sort-claude/AGENTS.md)
— its "Where you are" and "Whether you're done" bullets are the direct fix for this exact
failure, so you can read the "after" side of the diff even though the "before" no longer exists
(next paragraph).

**Don't expect to reproduce this failure yourself.** An earlier version of this page presented it
as if you could; [CADS-DEMO-sort#22](https://github.com/scimbe/CADS-DEMO-sort/issues/22) (item 7)
tried, hard, and established three things a reader deserves to know up front:

- **The broken spec no longer exists as a file.** It was a hand-built prompt string from before
  this participant's spec lived in a committed `AGENTS.md`; it was never checked in, so there is
  no artifact in the repo to re-run. The phrasing quoted above is reconstructed from the failure
  record in CADS-DEMO-sort#10, not copied from a surviving file.
- **It predates the generate-once harness.** The failing runs called the model live, once per
  round. The harness this tutorial teaches was built largely *because* of failures like this one
  — the evidence page linked below walks that history in order.
- **Today's models tend to patch this exact gap on their own.** CADS-DEMO-sort#22 deliberately
  re-created the vague spec — the same "remember where you are in a pass" framing — and the
  generated handler passed every check first try, because the model silently added a direct
  sortedness check the spec never asked for. A vague spec that happens to generate working code
  is still a vague spec: nothing guarantees the next model, or the next regeneration, fills the
  same gap the same way — which is precisely the reliability point of this tutorial.

So treat this section as a documented case study of the diagnose-tighten-retest loop, not as an
exercise to replay. The loop is what transfers: when your own generation fails, the failure
points at a question your spec left unanswered.

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

## Step 5 — watch it sort

Everything above is local verification: `--selftest`, a dry run, determinism, the correction path.
None of it puts your handler on screen — and the visualization is the actual payoff the homepage
advertises ("a real, running visualization, not a static mockup"). `index.html`'s default-selected
**Solo run** tab does exactly this: pick your participant, an array length, a round budget, run it,
and watch a live bar visualization, a round timeline colored by compare/swap/fault/done, and a
scorecard fill in as your handler actually sorts (CADS-DEMO-sort#22 found this step missing from
every tutorial in this repo, despite being the single most convincing thing in the project).

You don't need the hosted arena or a join request to see this — [Run the arena
locally]({{ '/how-to/run-the-arena-locally/' | relative_url }}) gets your own generated handler
into that same GUI with zero dependencies, no network, and no operator, using nothing but the
`node`/`python3` you already have.

## Once your service is reliable: two directions from here

**Go deeper into the harness** — the natural next step: [Evolve the harness — from "it sorts" to
"it IS bubble sort, provably"]({{ '/tutorials/evolve-to-bubble-sort/' | relative_url }}) takes the
participant you just built and steers it toward a *named, mechanically checkable* algorithm using
`dryrun.py`'s property checks (`--require-adjacent`, `--require-optimal-swaps`) as a goal line and
a verification hook as the referee. That's where the spec-tightening loop you just practiced
becomes a real engineering workflow.

**Or go live now**: turning your generated handler into something `sort.bunsenbrenner.org`
actually calls is self-service — sign in with your Keycloak account (the login is the
legitimization), submit, get approved automatically, run the serve command it hands you. See
[Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}) for the full
walkthrough.
