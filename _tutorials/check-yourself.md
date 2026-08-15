---
title: Check yourself
description: Six predictions with a command that settles each one. If a page taught you that unchecked properties aren't properties, it owes you a way to check what you took away.
order: 7
---

# Check yourself

This site argues that a property nobody checks is not a property of the system. A training that
can't tell you whether you understood it has the same problem — so here are six claims to predict,
each with a command that settles it.

**Predict before you run.** A prediction you didn't commit to can always feel like the one you had
in mind. Write the number down.

Every expected output below was produced by running the command, not reasoned out.

## Setup

Anywhere outside the clone, with the baseline handler:

```bash
REPO="$(git rev-parse --show-toplevel)"   # run this inside your clone first
mkdir -p ../check/generated && cd ../check
cp "$REPO"/templates/generated-python/{generate.sh,handler.sh,reference-handler.py} .
cp reference-handler.py generated/handler.py
```

---

## 1. How many swaps?

The handler sorts by swapping adjacent pairs. Before running it on three different arrays: **what
determines the swap count, and can you predict it without knowing the algorithm?**

```bash
for s in 7 42 99; do python3 "$REPO/dryrun.py" ./handler.sh --seed $s --len 12 --quiet | grep rounds=; done
```

<details><summary>Answer</summary>

```
rounds=32 comparisons=0 swaps=31 … inversions=31
rounds=28 comparisons=0 swaps=27 … inversions=27
rounds=35 comparisons=0 swaps=34 … inversions=34
```

`swaps == inversions`, every time. You can predict it from the *array* without knowing which
adjacent-swap algorithm is running, because an adjacent swap removes at most one inversion — so any
adjacent-only strategy pays exactly the inversion count. **The strategy has no say in this number.
The contract does.**

</details>

## 2. Where do the rounds come from?

Same output. **Predict `rounds` from the other two columns.**

<details><summary>Answer</summary>

`rounds = comparisons + swaps + 1`. Every move costs one round; the final `done` costs one more.
32 = 0 + 31 + 1, and so on.

That identity is why a `compare` move is expensive here even though it changes nothing: it buys a
round's worth of nothing. Which sets up question 4.

</details>

## 3. What does a failed generation cost you?

You have a working handler. A generation now fails outright — both attempts produce garbage.
**Predict the state of `generated/handler.py` afterwards.**

```bash
md5 -q generated/handler.py        # or md5sum on Linux
# ... a generation that fails twice ...
md5 -q generated/handler.py
./handler.sh --selftest
```

<details><summary>Answer</summary>

**Byte-identical, and the selftest still passes.** The draft is written to a staging file and only
moved into place once it compiles, so a failed draw cannot reach your working handler.

This was not always true — until `2805fb6` a doubly-failed generation deleted it. Worth knowing
because it changes how much you have to fear pressing the button: the answer is now "nothing".

</details>

## 4. What does the word "compare" cost?

Two specifications, same meaning, same model:

> **A** — Scan the array left to right **comparing** each adjacent pair; if a pair is out of order,
> swap it immediately.
>
> **B** — Scan the array from left to right. Whenever an element is larger than the one directly to
> its right, swap that pair immediately.

**Predict the `comparisons` column for each.**

<details><summary>Answer</summary>

A produced `175` compare moves in seven of eight generations. B produced `0`, every time — with no
instruction anywhere telling it not to.

`compare` is a move type in the contract. A specification that names it gets code that emits it.
**Your spec's vocabulary is part of your harness**, and here it costs a factor of 2.5 in the metric
the arena ranks by. [Change the skill]({{ '/how-to/change-the-skill/' | relative_url }}) has the
three-arm measurement.

</details>

## 5. Will merge sort win?

You write a merge sort participant and race it against the adjacent-swap baseline.
**Predict which finishes in fewer rounds.**

<details><summary>Answer</summary>

**Neither reliably — merge sort has no advantage here.** Its edge over insertion sort is in
comparison count, and comparisons are free: the round input hands you the whole array, so you can
read every element without spending anything the arena measures.

The contract has no dimension where merge sort's strength shows up. A harness that measures the
wrong thing makes a better approach look equal or worse —
[Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) walks it out.

</details>

## 6. When does the default budget run out?

**Predict: at which array length does the default `--budget 200` start reporting `sorted=False`?**

```bash
python3 "$REPO/dryrun.py" ./handler.sh --seed 42 --len 21 --quiet | grep rounds=
python3 "$REPO/dryrun.py" ./handler.sh --array "$(seq 21 -1 1 | paste -sd, -)" --quiet | grep rounds=
```

<details><summary>Answer</summary>

**The question is a trap: length is the wrong variable.**

```
random,   len 21:   rounds=123 … sorted=True   inversions=122
reversed, len 21:   rounds=200 … sorted=False  inversions=210
```

Same length, opposite outcome. From question 1, the cost is `inversions + 1` — and a random array
has about `n²/4` inversions while a reversed one has `n(n-1)/2`. At length 21 that's ~105 versus
210, and only the second exceeds 200.

An earlier version of this site stated the worst case as a general rule ("from `--len 21` upward you
need `--budget 600`"). It measured wrong because it asked about length instead of the thing that
actually drives the number.

Above `--len 24` nothing runs at all: `MAX_ARRAY_LEN` is 24, and exceeding it exits 1 with a Python
traceback rather than a message.

</details>

---

## If you got 1, 2 and 4

Then you have the thing this site is for. All three are cases where **the outcome was decided by
the harness rather than by the strategy or the model**: a bound the contract imposes, an identity
the round loop enforces, and a word in a file becoming a move on the wire.

The model was the same in every one of them. That's the whole argument, and now you have checked it
rather than been told it.
