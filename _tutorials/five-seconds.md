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

### Where you are, and how to get back

Two directories now matter, and every command below assumes you are in the second one:

```
CADS-DEMO-sort/     the clone — you only ever read from it
mysorter/           your participant — everything you run lives here
```

`REPO` and `SORT_PROTOCOL_MD` are shell variables. **They vanish when you close the terminal**, and
nothing reminds you. Opening a new one, or coming back tomorrow, start with this:

```bash
cd /path/to/mysorter                                   # your participant directory
export REPO="$(cd ../CADS-DEMO-sort && pwd)"           # adjust if your clone sits elsewhere
export SORT_PROTOCOL_MD="$REPO/participants/CLAUDE.md"
```

A quick check that you are in the right place with the right variables set:

```bash
ls handler.sh generate.sh && ls "$SORT_PROTOCOL_MD" && echo "REPO=$REPO"
```

All three have to answer. If `ls handler.sh` fails you are in the wrong directory; if the contract
path fails, `REPO` points somewhere else than your clone. (`AGENTS.md` is deliberately not in that
check — you only write it further down, and a reader re-entering before then would be alarmed by
its absence.) Everything in this tutorial fails in confusing ways from the wrong directory, so it is
worth the three seconds.

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
export SORT_BRIDGE_LISTEN=127.0.0.1:8789
node bridge/server.js &
python3 -m http.server 8000 --bind 127.0.0.1 &
```

Both binds are explicit on purpose. Neither server needs anything beyond your own machine for this
tutorial, and their defaults (`0.0.0.0`) would otherwise put both on your network — reachable by
anyone else on the same Wi-Fi, not just you.

Both servers run from the clone, which is why the `cd` is there; `cmd` is resolved relative to it,
so `../mysorter/handler.sh` points back at your directory. Return with `cd ../mysorter` afterwards.

**Open a new terminal for that `cd ../mysorter` and everything after it — the coding-agent CLI in
particular.** The `&` above backgrounds both servers in *this* terminal, not in a separate session.
Ctrl-C here still only targets the foreground job, but closing this terminal, or a shell that treats
a stuck `generate.sh`/`claude` call and its backgrounded siblings as one job on interrupt, takes the
arena down with it — and the next command in this tutorial re-opens the same page expecting it to
still answer. A second terminal removes the question entirely: this one just runs the two servers
and you leave it alone.

**Confirmed the other direction too: Ctrl-C in this terminal does not stop them either.** They are
backgrounded, so a Ctrl-C here has nothing in the foreground to hit — the servers are still running,
and re-running the two start commands fails with `Address already in use` /
`Error: listen EADDRINUSE`. That is not a broken restart, it's the previous pair still holding both
ports. Stop them by port, then start again:

```bash
lsof -ti:8789 -ti:8000 | xargs kill    # macOS/Linux; still not free after this, check for
                                        # a process from a different terminal session:
pkill -f "bridge/server.js"
pkill -f "http.server 8000"
```

On Windows (PowerShell): `Get-NetTCPConnection -LocalPort 8789,8000 | Select OwningProcess`, then
`Stop-Process -Id <that PID>` for each.

Then open **`http://127.0.0.1:8000/index.html?bridge=http://127.0.0.1:8789`** and hit *Solo run*.

![A completed solo run: sorted bars, the round timeline, the move log, and the scorecard]({{ '/assets/participant-01-first-run.png' | relative_url }})

The `?bridge=` parameter is required because the page (port 8000) and the API (port 8789) are on
different origins. Without it the page assumes same-origin. Full detail in
[Run the arena locally]({{ '/how-to/run-the-arena-locally/' | relative_url }}).

## 0:10 — Now make it yours

Nothing above involved a model. That changes now.

**Do this in the new terminal from the step above.** It is a fresh shell — `REPO` and
`SORT_PROTOCOL_MD` are not set in it yet, even though you set them earlier in the *other* terminal.
Run the re-entry block from "Where you are, and how to get back" (0:01, above) now, in this
terminal, before anything else:

```bash
cd /path/to/mysorter                                   # your participant directory
export REPO="$(cd ../CADS-DEMO-sort && pwd)"           # adjust if your clone sits elsewhere
export SORT_PROTOCOL_MD="$REPO/participants/CLAUDE.md"
```

**Skip this and `./generate.sh` fails with `cat: .../CLAUDE.md: No such file or directory`** — that
exact message, because unset `SORT_PROTOCOL_MD` makes `generate.sh` fall back to a path
(`mysorter/../CLAUDE.md`) that only exists if your participant sits inside the clone, which since
0:01 it deliberately doesn't. If you hit that error, this is why — export the three lines above and
retry.

Now create the file at exactly `mysorter/AGENTS.md` — next to `generate.sh`, not inside `generated/`:

```bash
cat > AGENTS.md <<'EOF'
# mysorter

## Strategy
Scan the array left to right comparing each adjacent pair; if a pair is out of
order, swap it immediately; once a full pass produces no swaps, it is sorted.
EOF
```

Describe your own strategy in your own words — no algorithm name required. The block above is only
the example; typing or pasting it verbatim gets you the same adjacent-swap handler you already have.

### What actually generates the code

`generate.sh` does not contain a model. It **shells out to a coding-agent CLI that you have
installed and signed in**, hands it the move contract plus your `AGENTS.md`, and writes whatever
comes back to `generated/handler.py`.

```bash
LLM="${CT_LLM_CMD:-claude}"        # this is the line in generate.sh that picks the tool
```

So before running it you need **one** of these on your PATH and authenticated:

| CLI | Use it by |
|---|---|
| **Claude Code** (`claude`) | the default — nothing to set |
| **opencode** | `export CT_LLM_CMD=opencode` |
| **Codex** (`codex`) | `export CT_LLM_CMD=codex` |
| **Gemini** (`gemini`) | `export CT_LLM_CMD=gemini` |

The code is generated **wherever that CLI sends its request** — for the cloud CLIs, on the vendor's
servers; your machine only receives the text and writes the file. Nothing about the arena runs a
model: the contract is spoken by the generated Python, and that runs locally.

If you have no CLI installed, everything up to here still worked — you already have a sorting
participant. You just can't replace it with your own strategy yet.

**And the CLI does not have to talk to a vendor.** `generate.sh` only knows `CT_LLM_CMD`; where that
tool sends its request is the tool's business. Pointing the whole pipeline at a model on your own
GPU is two environment variables — and it is the hardest and most convincing version of this
exercise, because the one thing you swap is the part everyone assumes is decisive. See
[Run it against your own model]({{ '/how-to/run-against-your-own-model/' | relative_url }}).

Then generate and verify:

```bash
# you are in mysorter/, with REPO and SORT_PROTOCOL_MD set — see "Where you are" above
./generate.sh
./handler.sh --selftest
python3 "$REPO/dryrun.py" ./handler.sh --seed 42 --len 8 --quiet
python3 "$REPO/dryrun.py" ./handler.sh --correction-check
```

**How long this takes, measured rather than promised:** across **29 attempts, 28 of which produced
a usable handler**, the median was **130 seconds**, with a spread from 25 s to 472 s. Two of the 28
took longer than five minutes.
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

## 0:30 — Going live in the hosted arena

Everything so far ran on your machine. Three steps put the same handler on
`sort.bunsenbrenner.org`, where anyone can race it. **They took 16 seconds measured**, and none of
them touches your code — the handler you already verified is the one that goes online.

### Step 0 — you need an account, and you can make one yourself

`join.html` sits behind the deployment's sign-in, so before anything else you need a login. There is
no invitation step: the realm has **open self-registration**.

1. Open [sort.bunsenbrenner.org/join.html](https://sort.bunsenbrenner.org/join.html). You are
   redirected to the sign-in page.
2. Click **Register** at the bottom (*"New user? Register"*).
3. Fill in e-mail, password, first and last name, and accept the terms.
4. You are signed in immediately — **no confirmation mail is sent, and none is needed.**

That last point is worth knowing rather than discovering: the account works the moment you submit.
Verified by creating one today.

### Step 1 — `join.html` gives you a grant

Back on **[join.html](https://sort.bunsenbrenner.org/join.html)**, your signed-in account *is* the
authorisation — there is no waiting room.

The page generates a channel identity **in your browser**: a holder keypair and a noise keypair.
The private halves never leave it; only the public halves plus a signature are submitted. Fill in a
participant id and a display label, submit, and the grant comes back on the first status poll.

**Measured: 4.6 s** from submit to a grant in hand.

Two things to get right, because both cost a fresh start if you don't:

- **Copy the whole serve block immediately, and finish on this machine.** The grant is delivered
  once. Your private keys live in this browser's local storage and nowhere else, so re-opening the
  page elsewhere mints a *different* identity that the grant does not match.
- **If you lose it, submit again under a new participant id.** Your browser keeps the same identity;
  only the id and grant are fresh. That is the documented recovery, and it works — it is also the
  reason a portal route with re-fetchable grants is being built.

### Step 2 — get the binary, wire it to your handler, start it

**Download `ct-agent`.** The [releases page](https://github.com/scimbe/ct-agent/releases/latest)
ships one file per platform — no installer, no build step. Pick by OS and architecture:

| Your machine | Asset |
|---|---|
| macOS, Apple Silicon | `ct-agent-darwin-aarch64` |
| macOS, Intel | `ct-agent-darwin-x86_64` |
| Linux, 64-bit | `ct-agent-linux-x86_64` (or `-aarch64`) |
| Windows | `ct-agent-windows-x86_64.exe` |

```bash
# example: macOS on Apple Silicon
curl -L -o ct-agent https://github.com/scimbe/ct-agent/releases/latest/download/ct-agent-darwin-aarch64
chmod +x ct-agent
./ct-agent --version        # expect 0.5.3 or newer
```

There is **no FreeBSD asset**; see [Join as a participant]({{ '/how-to/join-as-a-participant/' | relative_url }})
for what to do there.

**Wire it to your handler.** The join page's serve block already does the wiring — it's not a
`.env` file to load separately, it's a literal shell transcript: a dozen `export` lines followed by
the actual `ct-agent channel` command that starts serving, all in one piece. One of those exports,
`CT_AGENT_SERVICE_HANDLER_CMD`, is already set to your handler by the page — that line is the whole
link between the tunnel and your sorter: it tells `ct-agent` **which program to hand each round
input to**. The arena calls the tunnel, the tunnel calls your `handler.sh`, and your handler's one
line of JSON goes back the same way. Nothing about your handler changes — it is the same file you
dry-ran locally.

Copy the whole block from the join page and either paste it straight into your terminal, in
`/path/to/mysorter`, or save it as a script first if you want to keep or review it:

```bash
cd /path/to/mysorter            # your participant directory again
# paste the block into serve.sh, then:
chmod 600 serve.sh              # it contains your private keys
bash serve.sh                   # runs every export, then ct-agent channel — no source, no separate step
```

The first log line is the one that matters:

```
ct-agent channel: plane-brokered Accept (relay …:4436) — persistent serve: concurrent sessions
ct-agent channel: direct rung …:4435 succeeded
ct-agent channel: channel park expired with no partner within the edge park window (#21) -- re-parking
```

`plane-brokered Accept` means you paired correctly. The re-parking line every 30 seconds is **normal
idle behaviour**, not a fault — it means you are connected and waiting for the arena to call you.

**Measured: 2 s** from start to `Accept`.

### Step 3 — it is usable on the arena page

Open **[sort.bunsenbrenner.org](https://sort.bunsenbrenner.org/)**. Your label is in the participant
dropdown; pick it, choose an array length, and hit *Solo run*.

![The hosted arena running a participant called 'intern sorter (adjacent)': finished correctly, comparisons 0, swaps 51, faults 0, rounds 52, and a move log ending in 'done — array is sorted']({{ '/assets/hosted-own-participant.png' | relative_url }})

**Measured: 5 s** for the first answered round. A real run of this handler:

```
finishedCorrectly  True
comparisons  0     swaps  74
faults       0     transportFaults  0
roundsUsed   75    wallClockMs  6353
```

`0 + 74 + 1 = 75` — the same round accounting as your local dry run, now over a real Agent-Fabric
channel. **83 ms per round**, against roughly 50 ms locally; the difference is the network, not your
handler. `transportFaults` is counted separately from `faults` and never scored against you.

Leave `ct-agent` running for as long as you want to stay live. Full detail, including the portal
route and what to do when something does not come up, is in
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
29 attempts, 28 usable, median 130 s, range 25–472 s — the one failure produced prose instead of
code and was caught by the compile check. Full path to answering rounds in the hosted arena: 43 s in one continuous run. Screenshots
are from a locally running arena and from the hosted one.*
