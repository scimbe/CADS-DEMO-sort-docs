---
title: Sort Arena — Docs
description: Documentation for CADS-DEMO-sort — same task, same contract, different harness.
---

# Sort Arena

**CADS-DEMO-sort** ([GitHub](https://github.com/scimbe/CADS-DEMO-sort)) is a CADS-Tunnel reference
pipeline where nothing in the repo itself sorts anything. Every participant is a separate process,
reached over the same real move contract, and the whole point is watching how differently the
*exact same underlying model* behaves depending on what harness (system prompt, skill, algorithm
coaching) is wrapped around it — not the prompt text.

The live arena is at **[sort.bunsenbrenner.org](https://sort.bunsenbrenner.org/)** — a real,
running visualization, not a static mockup.

## Start here

- [Bring your own participant online]({{ '/tutorials/first-participant/' | relative_url }}) — the
  `sort-arena-harness` skill loop, literal commands included: describe a strategy in plain
  language, get real generated code back, verify it before it goes live.
- [Change the algorithm]({{ '/tutorials/change-the-algorithm/' | relative_url }}) — the same skill,
  a genuinely different algorithm (merge sort), and a real, measured answer to what this specific
  harness actually costs an `O(n log n)` algorithm versus a simpler one.

## Sections

- **[Tutorials]({{ '/tutorials/' | relative_url }})** — learn by doing, start to finish.
- **[How-to guides]({{ '/how-to/' | relative_url }})** — accomplish a specific task.
- **[Reference]({{ '/reference/' | relative_url }})** — look up an exact fact (the move contract, field shapes, bounds).
- **[Explanation]({{ '/explanation/' | relative_url }})** — understand why the arena is built this way.
