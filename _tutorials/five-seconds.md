---
title: Five seconds to your first sorter
description: Clone, copy, run — a working participant before any model call. Then make it yours, and see why the harness, not the model, is what actually differs.
order: 0
---

# Five seconds to your first sorter

Every other tutorial here teaches something specific. This one gets you running, and then shows you
the single idea the whole project exists to demonstrate: **the model is the constant, the harness is
the variable.**

Timings below are measured, not estimated. Commands are exactly what was typed.

## 0:00 — Clone

No dependencies, no build step, no account.

```bash
git clone https://github.com/scimbe/CADS-DEMO-sort.git
cd CADS-DEMO-sort
```

## 0:01 — Create a participant

The scaffold ships a working, contract-verified sorter. You copy it, and you have something runnable
**before any model has been asked anything**.

```bash
mkdir -p participants/mysorter/generated
cp templates/generated-python/{generate.sh,handler.sh,reference-handler.py} participants/mysorter/
cp participants/mysorter/reference-handler.py participants/mysorter/generated/handler.py
chmod +x participants/mysorter/*.sh
```

## 0:05 — It runs

Two checks, then it's live in the arena on your own machine.

```bash
./participants/mysorter/handler.sh --selftest
python3 dryrun.py participants/mysorter/handler.sh --seed 42 --len 8 --quiet
```

```
SELFTEST OK: mysorter's generated handler emits a valid move for a real round input
start: [81, 14, 3, 94, 35, 31, 28, 17]
rounds=18 comparisons=0 swaps=17 faults=0 wrongDone=0 sorted=True inversions=17 wallClockMs=857
final: [3, 14, 17, 28, 31, 35, 81, 94]
```

That's the whole output, verbatim — `--quiet` drops the per-round log, not the start and final arrays.
`swaps=17` equals `inversions=17`, which is the first thing worth noticing: this handler spends
exactly the minimum an adjacent-swap strategy can spend. Only `wallClockMs` varies between runs.

Now watch it work. Two servers — the bridge answers the API, a static server serves the page:

```bash
export SORT_PARTICIPANTS_JSON='[{"you":"mysorter","label":"My sorter","cmd":"./participants/mysorter/handler.sh"}]'
node bridge/server.js &
python3 -m http.server 8000 &
```

Then open **`http://127.0.0.1:8000/index.html?bridge=http://127.0.0.1:8789`** and hit *Solo run*.

![A completed solo run: sorted bars, the round timeline, the move log, and the scorecard]({{ '/assets/participant-01-first-run.png' | relative_url }})

The `?bridge=` parameter is required because the page (port 8000) and the API (port 8789) are on
different origins. Without it the page assumes same-origin. Full detail in
[Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }}).

## 0:10 — Now make it yours

Nothing above involved a model. That changes now. Describe your strategy in your own words — no
algorithm name required — in `participants/mysorter/AGENTS.md`:

```markdown
# mysorter

## Strategy
Scan the array left to right comparing each adjacent pair; if a pair is out of
order, swap it immediately; once a full pass produces no swaps, it is sorted.
```

Then generate and verify:

```bash
./participants/mysorter/generate.sh
./participants/mysorter/handler.sh --selftest
python3 dryrun.py participants/mysorter/handler.sh --seed 42 --len 8 --quiet
python3 dryrun.py participants/mysorter/handler.sh --correction-check
```

**How long this takes, measured rather than promised:** across 11 generations the median was
**127 seconds**, with a spread from 38 s to 409 s. Two specifications of quite different complexity
were interleaved and came out the same (156 s vs 161 s back to back), so the spread is latency, not
your wording — a fast run is luck, not a faster prompt. Budget a few minutes and don't conclude
anything from one slow run.

**Before you run it, protect what already works.** A failed generation currently *removes*
`generated/handler.py` instead of leaving the previous one in place, so a bad draw can take your
working participant with it:

```bash
cp participants/mysorter/generated/handler.py participants/mysorter/generated/handler.py.bak
```

This is a real defect, not a habit worth teaching — it's reported with a tested fix, and this
paragraph disappears from the page once that lands.

**Five things worth knowing before you hit them:**

- `correction check: NOTE` is **not** a failure. It's expected for a deterministic handler that
  never emits an invalid move, so it never receives a real correction to react to.
- Determinism means: same seed twice, identical numbers apart from `wallClockMs`. Run it twice.
- From `--len 21` upward an adjacent-swap sorter needs `--budget 600`. At the 200 default it reports
  `sorted=False` — that's the budget, not your handler.
- Never hand-edit `generated/handler.py`. It's a build artifact; `AGENTS.md` is the source. The next
  regeneration overwrites your edit.
- You do not need to read the generated Python. The three checks are what "it works" means here.

## What just happened

You didn't write a sorting algorithm. You wrote a *specification*, and a model turned it into code.
The model is the same one every other participant uses. What differs is only what you built around
it: how precisely the spec pins the behaviour, what checks run against it, what the contract allows.

That's the harness. And its effect is measurable rather than arguable:

![Race across three participants: comb sort 20 rounds, two adjacent strategies at 40 rounds and 39 swaps each]({{ '/assets/participant-02-race.png' | relative_url }})

Three participants, same array, same model. Comb sort finishes in 20 rounds against two runs of 40.
And the two adjacent-only strategies land on **exactly 39 swaps each** — the start array's inversion
count.

That is not coincidence, it's a bound: an adjacent swap removes at most one inversion, so a strategy
restricted to adjacent swaps always costs exactly `inversions` swaps, whatever algorithm picked the
moves. The only lever that saves rounds is jumping distance.

Which leads to the point of the whole arena: **the contract decides which of an algorithm's
strengths can show up at all.** Merge sort's advantage is in comparison count — and comparisons cost
nothing here, because the round input hands you the entire array already. So merge sort is not
faster than insertion sort in this arena. Not because it's worse, but because the harness doesn't
measure the dimension it wins in. [Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }})
walks that result out in full.

## Steering the harness on purpose

So far "it sorts" was the goal. The sharper version: a harness that guarantees a *specific* property
and proves it mechanically. Two checks separate "this is bubble sort" from "this sorts somehow":

```bash
python3 dryrun.py participants/mysorter/handler.sh --seed 42 --quiet \
  --require-adjacent --require-optimal-swaps
```

![The goal line before and after one added spec line: four named violations and exit 1, then both properties passing with exit 0]({{ '/assets/participant-03-goalline.png' | relative_url }})

The loop is always the same, and it is the actual lesson: **read the violation → add one line to the
spec → regenerate.** Not fix the generated code — that's an artifact. And only one change per round,
or you won't know which sentence enforced which property.

The exit code is the referee: 1 on violation, 0 on pass. That's what lets you wire the check into a
hook that runs automatically after every regeneration.
[Evolve the harness]({{ '/tutorials/evolve-to-bubble-sort/' | relative_url }}) is the full exercise.

## Going live

The local setup and the hosted arena are independent — nothing here registers you anywhere. When you
want your sorter reachable at `sort.bunsenbrenner.org`, that's a separate, self-service step: sign
in, pick an id and a label, run the command the page hands you. See
[Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }}).

## What you now know

| Step | Time | Depends on |
|---|---|---|
| Clone to running participant | 5 s | nothing — deterministic |
| Generating your own strategy | 38–409 s, median 127 s | one model call |
| Forcing and proving a property | 1 spec line | you |

The first row is the one that matters, and it's the reason the scaffold ships a working handler: the
part you can rely on is deterministic, and the part that varies by a factor of ten only ever upgrades
something that already runs.

The model is the constant. The harness is the variable, and a skill is the tool you touch it with:
it writes the spec, generates code from it, and checks the result against criteria you fixed in
advance. Doing that once on a sorting algorithm means doing it on a task whose correctness is fully
checkable — which is exactly why sorting is the example here, and not because sorting is
interesting.

---

*Measured on a fresh clone: clone 1 s, scaffold <1 s, first success 5 s (fastest observed 2 s).
Generation: 11 runs, median 127 s, range 38–409 s, 11/11 produced a contract-passing handler.
Screenshots are from a locally running arena.*
