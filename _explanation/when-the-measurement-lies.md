---
title: When the measurement lies
description: Seven wrong claims published on this site in one day, each caused by an unstated assumption in the measuring tool rather than in the system.
order: 4
---

# When the measurement lies

Every other page here argues for checking things: gates, property flags, exit codes. This one is the
other half, and it is the uncomfortable half. **A check is only as good as the instrument behind it,
and instruments carry assumptions nobody wrote down.**

The seven cases below are not hypothetical. Each was published on this site or reported to the
project as fact, and each was wrong. They happened within a single day, to someone actively looking
for this kind of error. That is the point: knowing about the failure mode does not make you immune
to it.

## 1. One fast window became a general figure

**Claimed:** generation takes about 31 seconds, measured over six runs.

**Actually:** the median over 28 runs is 130 seconds, with a spread from 25 s to 472 s.

The six runs were real and the arithmetic was right. They happened to fall in a fast window. Six
samples from a distribution with a 19-fold spread say almost nothing about its centre, and the
number was quoted for days as though it did.

**The assumption:** that a handful of consistent samples means a stable quantity. It usually means
you haven't sampled long enough to see the tail.

## 2. Block-wise arms invented a cause

**Claimed:** a more complex specification takes longer to generate — 201 s against 53 s.

**Actually:** run back to back, the complex spec took 156 s and the simple one 161 s. The difference
was latency drift between the two blocks of runs, not complexity.

The two arms were measured one after the other, which is the natural way to run them and the wrong
one. Anything that drifts over the measurement window — load, routing, time of day — is silently
attributed to whatever changed between the blocks.

**The fix is cheap:** interleave. Alternate the arms instead of batching them. This one correction
has since prevented two more wrong conclusions on this project.

## 3. Parsing a rendering instead of the source

**Claimed:** the join page's serve block omits two required variables, so `ct-agent` refuses to
start. A table was built showing which page emitted what.

**Actually:** the page emitted both. The block had three assignments on one line, and the regex used
to read it — `^\s*(CT_[A-Z_]+)=(\S+)` — took one match per line and silently dropped the rest. The
startup failures were real, but the cause was loss in transit, not absence in the source.

The table built on top of it was an artifact of the same regex, and it looked convincing precisely
because it was self-consistent.

**The assumption:** one assignment per line. Never stated, never checked, and the source file was
right there.

## 4. Asking about the wrong variable

**Claimed:** from array length 21 upward, the default round budget of 200 is exhausted and the run
reports `sorted=False`.

**Actually:** measured across eight seeds at lengths 18 through 24, zero failures. At length 21 with
a random array it sorts in 123 rounds.

Length is not what drives it. The cost is `inversions + 1`, and a random array has roughly `n²/4`
inversions while a reversed one has `n(n-1)/2`. At length 21 that is about 105 versus 210 — only the
second exceeds the budget. The original claim was the worst case stated as the general rule.

**The assumption:** that the variable in the question is the variable in the mechanism.

## 5. Measuring the wrapper, blaming the language

**Claimed:** a Java handler costs 211 ms per round against Python's 84 ms — JVM startup, unavoidable
under a process-per-round contract.

**Actually:** interleaved on JDK 25, Java runs 38–40 ms per round and Python 47–48 ms. Java is the
faster one.

The 211 ms was real. The wrapper being measured checked for `java` and `javac` **by executing
them**, on every round — a deliberate choice, because probing with `command -v` finds a broken stub
on Windows. So every round paid for three JVM starts instead of one. Adding the probe back
reproduces it exactly: 142 ms with, 39 ms without, same machine, same minute.

**The assumption:** that the thing being timed was the language. What was actually timed was the
whole invocation, and a check that looks free in a shell script is a process spawn.

## 6. A pattern narrower than the thing it was matching

**Claimed:** all my test agents were stopped. Reported three separate times over an afternoon.

**Actually:** one had been running the whole time, attempting a channel join roughly six times an
hour against a dead identity. Someone watching the other end of the connection found it and told me.

The command was `pkill -f "cta/ct-agent"`. It matched the agent at `/tmp/cta/ct-agent` and never
matched the older one at `/tmp/cads-probe/bin/ct-agent-0417`. The pattern was written for the
process I was thinking about, then used as though it covered the class.

**The assumption:** that a filter written for one instance generalises to the category. It is the
same shape as case 3 — a pattern nobody checked for completeness — but failing in the direction of
*missing* things rather than mangling them.

## 7. A checker that invented its own failures

**Claimed:** after a CSS fix, one page still scrolled sideways on a phone — 542 px in a 390 px
window.

**Actually:** it did not scroll at all. `window.scrollTo(9999, 0)` moved the page zero pixels.

The script had been flagging any element whose right edge fell outside the viewport. But content
*inside* a horizontally scrollable container is supposed to do exactly that — that is what the
container is for. So once the fix worked, the checker kept reporting the very thing that proved it
had worked. Two more guesses were made at a bug that no longer existed before the definitive test
— can the page actually be scrolled? — settled it in one line.

**The assumption:** that "extends past the viewport" and "makes the page scroll" are the same
question. They differ precisely where the fix operates.

## What they have in common

None of these was a mistake in the system under test. Every one was an unstated assumption inside
the measurement:

| Case | The assumption |
|---|---|
| 1 | a few consistent samples mean a stable quantity |
| 2 | nothing drifts between one batch of runs and the next |
| 3 | one assignment per line |
| 4 | the variable in the question drives the mechanism |
| 5 | the timed thing is the named thing |
| 6 | a filter written for one instance covers the category |
| 7 | "extends past the viewport" is the same question as "scrolls" |

Six and seven fail in opposite directions, which is worth noticing: one instrument **missed** real
cases, the other **invented** them. Both felt equally reliable while wrong.

And they share a tell: **each produced a clean, self-consistent story.** The six fast runs agreed
with each other. The complexity hypothesis explained the numbers. The table of missing variables was
internally coherent. That coherence is not evidence — a broken instrument reports consistently,
which is exactly what makes it convincing.

## The habits that caught them

Not a checklist to complete, but the moves that actually did the work here:

- **A control arm, interleaved.** Case 2 was caught this way, and case 5. If two conditions are
  being compared, alternate them; if only one condition exists, invent the other.
- **Read the source, not the rendering.** Case 3 would have taken thirty seconds to disprove by
  opening the file the parser was reading.
- **State the mechanism before the metric.** Case 4 dissolves the moment you ask what actually
  costs a round, rather than what correlates with failing.
- **Ask what else changed.** Case 5 turned on noticing that the two handlers differed in their
  wrappers, not just their languages.
- **Ask the question the fix operates on.** Case 7 survived two more guesses because the check
  asked "does anything stick out?" instead of "does the page scroll?". One line of the right
  question ended it.
- **Let someone outside look.** Case 6 was not caught here at all. Someone watching the other end
  of the connection saw a process that had been reported stopped three times. No amount of care on
  this side would have found it, because the instrument and the belief shared the same blind spot.

None of this is about being careful. All seven were made while being careful. It is about the
instrument being part of the system, and therefore something the harness has to check too — which
is the same argument this site makes everywhere else, turned back on the person making it.

## Why this page exists

Because a training that only shows its successes teaches the wrong thing. If you build gates and
harnesses on the strength of measurements, the measurement is load-bearing, and a page listing seven
failures in one day is more useful for that than a page implying they don't happen.

Every corrected figure above is now the one this site publishes. The retractions are in the commit
history and in the project's issue thread, named rather than quietly overwritten — which is the
only version of this that is worth anything.
