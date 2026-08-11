---
title: Bring your own participant online
description: Register a real account, bring up your own tunnel, then walk the same model through four escalating levels of harness control -- with real transcripts at each stage.
order: 1
---

# Bring your own participant online

Every claim in this tutorial is a real, reproduced result: a real Keycloak account, a real
CADS-Tunnel tunnel, and real `claude` CLI calls at each stage below. Nothing here is simulated or
described from memory.

## Part 1 — account and tunnel

### Step 1: register

Start at **[bunsenbrenner.org/portal](https://bunsenbrenner.org/portal)** — that's the one URL
worth memorizing here. It redirects to a fresh Keycloak sign-in page carrying the OAuth
parameters (`tab_id`, `client_data`, `state`) that page needs; those are minted per-visit, so a
copied `/realms/.../registration?...` URL from someone else's session (or from this page)
will have gone stale. From the sign-in page, click **Register** — a brand-new participant
account was registered exactly this way for this tutorial.

![Sign-in page]({{ '/assets/01-signin-page.png' | relative_url }})

### Step 2: accept terms, land on the portal

Keycloak's registration form, filled in for real:

![Registration form, filled in]({{ '/assets/02-register-form-filled.png' | relative_url }})

Then the one-time terms-and-conditions accept step, then straight into `/portal/home`, signed in
as the new account:

![New account, signed in]({{ '/assets/03-new-account-signed-in.png' | relative_url }})

### Step 3: the auto-provisioned tunnel

`/portal/tunnels` auto-provisions one free tunnel per account —
`site-bea1a24d.bunsenbrenner.org` in this run:

![New account's tunnels page]({{ '/assets/04-new-account-tunnels-page.png' | relative_url }})

The Install page shows a single-use `CT_AGENT_JOIN_TOKEN` and a persistent `CT_AGENT_TOKEN`
(shown once, copy it immediately):

![Install page, join and persistent tokens]({{ '/assets/05-new-tunnel-install-tokens.png' | relative_url }})

### Step 4: get `ct-agent`, then bring the tunnel up

**Get the binary first — there is no build step.** Download it for your platform from
[the latest release](https://github.com/scimbe/ct-agent/releases/latest) (always the current
version, so nothing here goes stale by pinning a number in prose):
`ct-agent-linux-x86_64`, `ct-agent-darwin-{x86_64,aarch64}` (make either executable with
`chmod +x ct-agent-*`), or on Windows `ct-agent-windows-x86_64.exe` — no `chmod` equivalent
needed there, a downloaded `.exe` just runs.

`CT_AGENT_JOIN_TOKEN` and `CT_AGENT_TOKEN` come from the Install page (Step 3, above).
`CT_AGENT_EDGE` and `CT_AGENT_EDGE_CERT_URL` were the two values with no explanation anywhere on
this page (CADS-DEMO-sort-docs#2) — real values, verified live: `CT_AGENT_EDGE` is the mesh
edge's `host:port` (`4433` on this deployment — confirm against `GET
https://bunsenbrenner.org/network-info`'s `mesh_edge_port` rather than hardcoding);
`CT_AGENT_EDGE_CERT_URL` is just the control-plane base URL again — the client appends `/pki/ca`
itself (`curl -s https://bunsenbrenner.org/pki/ca` returns the real cert directly, confirmed
`200 application/x-x509-ca-cert`).

**bash / Git Bash (Linux, macOS, Windows):**

`CT_AGENT_STATE_DIR`/`CT_AGENT_CAPABILITY_OUT` must point at real, already-existing directories —
without them, `ct-agent onboard` crashes immediately with `Os { code: 2, kind: NotFound }` rather
than falling back to a sane default. Both blocks below create the directory first so that can't
bite you.

```bash
mkdir -p ~/ct-agent-state
CT_AGENT_MODE=browser \
CT_AGENT_JOIN_TOKEN=<from the Install page> CT_AGENT_TOKEN=<from the Install page> \
CT_AGENT_ID=site-bea1a24d \
CT_AGENT_CP_URL=https://bunsenbrenner.org \
CT_AGENT_EDGE=bunsenbrenner.org:4433 CT_AGENT_EDGE_CERT_URL=https://bunsenbrenner.org \
CT_AGENT_HOSTNAME=site-bea1a24d.bunsenbrenner.org \
CT_AGENT_ORIGIN=127.0.0.1:18081 CT_AGENT_ORIGIN_PROTO=tcp \
CT_AGENT_STATE_DIR=~/ct-agent-state CT_AGENT_CAPABILITY_OUT=~/ct-agent-state/capability.bin \
./ct-agent onboard
```

**PowerShell:**

```powershell
New-Item -ItemType Directory -Force "$HOME\ct-agent-state" | Out-Null
$env:CT_AGENT_MODE = "browser"
$env:CT_AGENT_JOIN_TOKEN = "<from the Install page>"
$env:CT_AGENT_TOKEN = "<from the Install page>"
$env:CT_AGENT_ID = "site-bea1a24d"
$env:CT_AGENT_CP_URL = "https://bunsenbrenner.org"
$env:CT_AGENT_EDGE = "bunsenbrenner.org:4433"
$env:CT_AGENT_EDGE_CERT_URL = "https://bunsenbrenner.org"
$env:CT_AGENT_HOSTNAME = "site-bea1a24d.bunsenbrenner.org"
$env:CT_AGENT_ORIGIN = "127.0.0.1:18081"
$env:CT_AGENT_ORIGIN_PROTO = "tcp"
$env:CT_AGENT_STATE_DIR = "$HOME\ct-agent-state"
$env:CT_AGENT_CAPABILITY_OUT = "$HOME\ct-agent-state\capability.bin"
.\ct-agent-windows-x86_64.exe onboard
```

### Step 5: confirm it's live

Independent of what the portal UI shows, the real test is an external request:

```
$ curl -s https://site-bea1a24d.bunsenbrenner.org/
Sort Participant 1 — online
```

That response only exists if the tunnel is genuinely carrying traffic end to end through the
Agent-Fabric edge to the locally-running origin — nothing about it can be faked by the portal UI
alone.

![New tunnel, connected]({{ '/assets/06-new-tunnel-connected.png' | relative_url }})

## Part 2 — the same model, four levels of control

The point of this arena is that harness quality, not raw model capability, decides the outcome.
To make that concrete rather than asserted, the exact same underlying model was called four times
against the exact same round input, each time with a different amount of control wrapped around
it. Real transcripts below, nothing paraphrased.

Round input used for stages 1–2 (a plain array, no protocol machinery yet):

```json
{"round":1,"array":[5,3,8,1,9,2],"history":[],"budgetRemaining":43,"mode":"solo","you":"stage-N-test"}
```

### Stage 1 — no harness at all

Zero system prompt. The raw JSON goes to `claude -p` with no instruction beyond what's in the
JSON itself.

```
$ printf '%s' '{"round":1,"array":[5,3,8,1,9,2],...}' | claude -p "$(cat)" --output-format text
```

Real output — reproduced verbatim, fence-inside-fence and all, because the model wrapping its
own JSON in a markdown fence is itself part of the fault:

````
```json
{
  "round": 1,
  "action": "swap",
  "indices": [0, 1],
  "reason": "5 > 3 at positions 0-1; first out-of-order adjacent pair (bubble-sort pass)",
  "arrayAfter": [3, 5, 8, 1, 9, 2],
  "sorted": false,
  "budgetRemaining": 42
}
```

Target order for `[5,3,8,1,9,2]` is `[1,2,3,5,8,9]` — roughly 8 more adjacent swaps from here,
well within the 43-step budget.
````

Against `docs/protocol.md` this is a **fault**, and not a subtle one: wrapped in a markdown
fence, extra prose after the JSON block, `indices` instead of `i`/`j`, invented fields
(`arrayAfter`, `reason`, `sorted`) nowhere in the contract. What's notable is that the model's
*reasoning* is completely sound — it correctly identified an out-of-order adjacent pair and even
named "bubble-sort pass" unprompted. The failure is entirely about the wire format, not the
sorting logic. That is the whole lesson of stage 1: capability was never the bottleneck.

### Stage 2 — loosely controlled input/output

A one-line system prompt: "reply with JSON describing your move, keep it short." No field names,
no type spec, no enum of valid actions.

Real output:

```
{"move":"swap","i":0,"j":3,"array":[1,3,8,5,9,2],"done":false}
```

Progress over stage 1: valid, unfenced JSON, no prose. Still a fault: the key is `move` not
`action`, and the reply invents `array`/`done` fields that aren't part of the contract — it went
further than asked and reported a resulting array, which is the bridge's job, not the
participant's. Also worth noticing: `i=0, j=3` swaps `5` and `1` — a legal but non-adjacent move,
which is fine under the general move protocol but would not be a valid bubble-sort step if that
were the intent. Vague steering got partial format compliance and a plausible-looking move; it
did not get contract compliance.

### Stage 3 — the full protocol contract, no algorithm coaching

This is [`participants/minimal-claude/`](https://github.com/scimbe/CADS-DEMO-sort/tree/main/participants/minimal-claude):
the complete `docs/protocol.md` contract spelled out in the system prompt (exact field names,
exact three actions, exact bounds), and nothing else — no strategy, no skill file, no
self-check step.

Real live run against `bridge/server.js` (`len=6`, `SORT_BUDGET=20`):

| Metric | Value |
|---|---|
| initial array | `[17, 22, 62, 27, 12, 4]` (10 inversions) |
| final array | `[4, 17, 12, 22, 27, 62]` (1 inversion) |
| `finishedCorrectly` | **false** |
| `comparisons` | 2 |
| `swaps` | 5 |
| `faults` | **0** |
| `roundsUsed` | 20 (entire budget) |
| `inversionsOverTime` | `[10, 5, 4, 3, 2, 1]` |

Zero faults — the bare format instruction really is enough to keep the model inside a strict
JSON contract. But it never finished: at one inversion from sorted it emitted `{"action":"done"}`,
was told (implicitly, by the run continuing) that it was wrong, and repeated the exact same wrong
`done` **thirteen times in a row**, burning the rest of its budget. None of those thirteen replies
were faults — a well-formed `done` is a valid reply, being wrong about it is not a protocol
violation. So this run reads `faults: 0` next to `finishedCorrectly: false`. Contract compliance
and actually succeeding at the task are two different axes, and stage 3 is where that becomes
visible: the harness stopped losing on *format* and started losing on *unverified
self-assessment* instead.

### Stage 4 — full harness control, adapted specifically for bubble sort

This is the new participant built for this walkthrough,
[`participants/bubble-sort-claude/`](https://github.com/scimbe/CADS-DEMO-sort/tree/main/participants/bubble-sort-claude).
Full protocol contract plus a real, native `AGENTS.md`/`CLAUDE.md` (not an inlined system-prompt
string — see "A mid-course harness correction" below) coaching literal bubble sort: adjacent-pair
passes only, one pair visited per round, swap on out-of-order, direct sortedness check (not
pass-tracking) to decide `done`.

Adapting bubble sort specifically — not just "an efficient strategy" — to a handler that is
invoked fresh every round with no memory of its own was the actual engineering problem here: real
bubble sort needs to remember where it is in a pass and whether the pass has swapped anything.
The strategy reconstructs the cursor from the single most recent entry in `history` (always
present regardless of how long a pass runs) instead of trying to track a pass boundary through a
capped 20-entry window.

**First real attempt — a genuine stall, reported honestly, not hidden.** The first version of this
strategy, delivered via a hand-built `--append-system-prompt` string (the same mechanism every
other participant in this repo used at the time), was dry-run twice:

| Run | Array | Budget | Result |
|---|---|---|---|
| 1 | `[5,3,8,1,9,2]` (8 inversions) | 30 | `rounds=30 faults=0 sorted=False` — budget exhausted |
| 2 | `[18,60,61,29,26,25]` (9 inversions) | 40 | `rounds=35 faults=5 sorted=False` — budget exhausted |

Both runs failed the same way: at specific cursor positions, the model repeatedly emitted
`compare` instead of `swap` despite the array visibly showing an inversion right there — reproduced
identically on two different arrays. Full transcripts and analysis:
[CADS-DEMO-sort#10](https://github.com/scimbe/CADS-DEMO-sort/issues/10).

### A mid-course harness correction — the point of this whole section

The instinct after a repeated deviation like that is to re-run the same prompt and hope for a
cleaner sample. That was resisted here. Instead, the *harness itself* changed, for an unrelated
but concurrent reason: every participant's strategy text moved out of a hand-built
`--append-system-prompt` string and into a real `AGENTS.md` file that Claude Code (and Codex,
Gemini CLI, opencode) discover natively from the working directory, with a shared
`participants/CLAUDE.md` adding an explicit **Contract Criterion** — format, termination, and
no-regression-on-correction, stated so a run can be checked against it, not just eyeballed
([CADS-DEMO-sort#11](https://github.com/scimbe/CADS-DEMO-sort/issues/11)). See the docs site's
["Structuring a harness instruction"]({{ '/explanation/instruction-structure/' | relative_url }})
page for the general principle this is one real instance of.

**Verified this actually works, not just assumed:** a throwaway parent/child directory pair with
a `PARENTWORD77` marker one level up and a `CHILDWORD99` marker in the working directory —
`claude -p` from the child directory returned both words in one reply, confirming Claude Code's
project-file discovery really does walk up and merge parent and child `CLAUDE.md` files. Also
confirmed the walk stops at the enclosing git repository root: a marker file placed further up,
outside the repo, was never picked up.

**Re-run with the identical strategy content, delivered the new way** — same two seed arrays that
stalled before:

| Run | Array | Budget | Result |
|---|---|---|---|
| 3 | `[5,3,8,1,9,2]` (8 inversions) | 30 | `rounds=17 faults=0 sorted=True` |
| 4 | `[18,60,61,29,26,25]` (9 inversions) | 40 | `rounds=17 faults=0 sorted=True` |

Both previously-stalling seeds now converge cleanly. **This is not proof the delivery mechanism
caused the fix** — two runs each is not enough to rule out ordinary model-response variance, and
the honest, unhedged claim is only: after changing the harness (not the strategy content) in
response to a real, reproduced deviation, the same deviation did not recur across both retests.
That is the right *shape* of response to a repeated harness failure — change what the harness
structurally provides, then measure again — whether or not this specific instance turns out to
generalize.

Live arena run against `bridge/server.js`, on the real `sort.bunsenbrenner.org` page, with
`bubble-sort-claude` selected as the only solo-run participant:

![Sort Arena, bubble-sort-claude selected]({{ '/assets/sort-arena-live-01-loaded.png' | relative_url }})

A real 12-element array, run to completion — every `compare`/`swap` in the round timeline below
is a real `claude` CLI call:

![Sort Arena, bubble-sort-claude finished correctly]({{ '/assets/sort-arena-live-04-bubble-sort-complete.png' | relative_url }})

`comparisons: 29, swaps: 31, faults: 0, rounds: 61, wall: 506.5s, sorted: yes`. 61 rounds for a
12-element array is real bubble-sort cost, not a harness problem — see "why this costs more
rounds than a direct-placement strategy" in `AGENTS.md`. Zero faults across 61 real LLM calls in
a row is the contract-compliance story from stage 3 holding up under sustained live load, not
just a short dry run.

**An honest wart, found live, not hidden:** the bridge's solo-run endpoint buffers the *entire*
run server-side and only streams round events to the browser once the whole thing finishes —
confirmed by a direct `curl -X POST .../run/bubble-sort-claude`, which returned nothing but a
`start` event for the first several tens of seconds. For a real LLM participant this means the
page can sit looking frozen (0 comparisons, 0 swaps, "running") for the full multi-minute
duration of a run before dumping the complete trace at once — the opposite of the page's own
"watch the path" premise. Filed as
[CADS-DEMO-sort#12](https://github.com/scimbe/CADS-DEMO-sort/issues/12), not fixed here.

### What the four stages actually show, side by side

| Stage | Control | Format compliance | Actually converges? |
|---|---|---|---|
| 1 — no harness | none | No — markdown fences, invented fields (`indices`, `reason`, `arrayAfter`) | N/A |
| 2 — loose control | one-line format hint | Partial — valid JSON, wrong keys (`move` not `action`), invented fields | N/A |
| 3 — full protocol, no strategy | complete wire contract | Yes, `faults=0` | No — 13 consecutive wrong `done` claims, never sorted |
| 4 — full harness, bubble sort | complete contract + coached strategy | Yes, `faults=0` | Yes (after a real harness correction — see above), 17 rounds both retests |

The progression is not "more control always wins" in a simple line — stage 3 shows contract
compliance and task success are genuinely different axes: zero faults, zero success. What
consistently moved the outcome was giving the model something *checkable* to fail against —
first the wire contract itself, then, when that alone wasn't enough, an explicit strategy plus a
stated contract criterion for what "done" has to mean.
