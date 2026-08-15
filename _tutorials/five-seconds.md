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
REPO="$(git rev-parse --show-toplevel)"
mkdir -p ../mysorter/generated && cd ../mysorter
cp "$REPO"/templates/generated-python/{generate.sh,handler.sh,reference-handler.py} .
cp reference-handler.py generated/handler.py
export SORT_PROTOCOL_MD="$REPO/participants/CLAUDE.md"
```

Your participant lives in **its own directory, beside the clone — not inside it.** Only two things
ever come from the repo: the move contract (`SORT_PROTOCOL_MD`) and `dryrun.py`. Your work stays
yours, and `git status` in the clone stays empty.

No `chmod` is needed — `cp` carries the executable bit over.

## 0:05 — It runs

Two checks, then it's live in the arena on your own machine.

```bash
./handler.sh --selftest
python3 "$REPO/dryrun.py" ./handler.sh --seed 42 --len 8 --quiet
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
cd "$REPO"
export SORT_PARTICIPANTS_JSON='[{"you":"mysorter","label":"My sorter","cmd":"../mysorter/handler.sh"}]'
node bridge/server.js &
python3 -m http.server 8000 &
```

Both servers run from the clone, which is why the `cd` is there; `cmd` is resolved relative to it,
so `../mysorter/handler.sh` points back at your directory. Return with `cd ../mysorter` afterwards.

Then open **`http://127.0.0.1:8000/index.html?bridge=http://127.0.0.1:8789`** and hit *Solo run*.

![A completed solo run: sorted bars, the round timeline, the move log, and the scorecard]({{ '/assets/participant-01-first-run.png' | relative_url }})

The `?bridge=` parameter is required because the page (port 8000) and the API (port 8789) are on
different origins. Without it the page assumes same-origin. Full detail in
[Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }}).

## 0:10 — Now make it yours

Nothing above involved a model. That changes now. Describe your strategy in your own words — no
algorithm name required — in `AGENTS.md`, next to the scripts you just copied:

```markdown
# mysorter

## Strategy
Scan the array left to right comparing each adjacent pair; if a pair is out of
order, swap it immediately; once a full pass produces no swaps, it is sorted.
```

Then generate and verify:

```bash
./generate.sh
./handler.sh --selftest
python3 "$REPO/dryrun.py" ./handler.sh --seed 42 --len 8 --quiet
python3 "$REPO/dryrun.py" ./handler.sh --correction-check
```

**How long this takes, measured rather than promised:** across **28 generations** the median was
**130 seconds**, with a spread from 25 s to 472 s. Two runs out of 28 took longer than five minutes.
Two specifications of quite different complexity were interleaved and came out the same (156 s vs
161 s back to back), so the spread is latency, not your wording — a fast run is luck, not a faster
prompt. Budget a few minutes and don't conclude anything from one slow run.

**A failed generation cannot cost you anything.** The draft is written to a staging file and only
moved into place once it compiles, so a bad draw leaves your previous handler exactly where it was —
`generate.sh` says so explicitly when it gives up. (This page briefly told you to keep a backup
first. That was correct at the time and isn't any more.)

**Five things worth knowing before you hit them:**

- `correction check: NOTE` is **not** a failure. It's expected for a deterministic handler that
  never emits an invalid move, so it never receives a real correction to react to.
- Determinism means: same seed twice, identical numbers apart from `wallClockMs`. Run it twice.
- **`sorted=False` is usually the budget, not your handler** — and the rule is about *inversions*,
  not length. An adjacent-swap sorter spends `inversions + 1` rounds, so the 200 default runs out
  only once the array has ~200 inversions. A random array needs about `n²/4` of them, so lengths up
  to 24 are normally fine; a *reversed* array needs `n(n-1)/2`, and at `--len 21` that is 210 —
  measured, it stops at `rounds=200 sorted=False`. Pass `--budget 600` when you feed adversarial
  arrays. (An earlier version of this page said "from `--len 21` upward you need `--budget 600`.
  That is true only for the worst case; at `--len 21` with a random array it sorts in 123 rounds.)
- **`--len` above 24 does not work at all.** `MAX_ARRAY_LEN` is 24, and exceeding it exits 1 with a
  Python traceback rather than a message. Reported.
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
python3 "$REPO/dryrun.py" ./handler.sh --seed 42 --quiet \
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
| Clone to running participant | 1–2 s | nothing — deterministic |
| Generating your own strategy | 25–472 s, median 130 s | one model call |
| Joining the hosted arena and answering a round | 16 s | nothing — deterministic |
| Forcing and proving a property | 1 spec line | you |

Measured end to end on one continuous run — empty directory to a participant answering rounds from
the hosted arena — that came to **43 seconds**, of which 25 s was the model call. With the median
generation instead it is about 2.5 minutes.

The first row is the one that matters, and it's the reason the scaffold ships a working handler: the
part you can rely on is deterministic, and the part that varies by a factor of ten only ever upgrades
something that already runs.

The model is the constant. The harness is the variable, and a skill is the tool you touch it with:
it writes the spec, generates code from it, and checks the result against criteria you fixed in
advance. Doing that once on a sorting algorithm means doing it on a task whose correctness is fully
checkable — which is exactly why sorting is the example here, and not because sorting is
interesting.

---

*Measured on fresh clones: clone plus scaffold 1–2 s to a contract-passing baseline. Generation:
28 runs, median 130 s, range 25–472 s, 27 of 28 produced a usable handler on the first or second
attempt. Full path to answering rounds in the hosted arena: 43 s in one continuous run. Screenshots
are from a locally running arena and from the hosted one.*
