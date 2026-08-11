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
  full path from a brand-new account to a real, live participant sorting on the arena, walked in
  four escalating stages: no harness, loosely controlled output, the real protocol, full harness
  control. Every screenshot is from the actual live deployment.

## Sections

- **[Tutorials]({{ '/tutorials/' | relative_url }})** — learn by doing, start to finish.
- **[How-to guides]({{ '/how-to/' | relative_url }})** — accomplish a specific task.
- **[Reference]({{ '/reference/' | relative_url }})** — look up an exact fact (the move contract, field shapes, bounds).
- **[Explanation]({{ '/explanation/' | relative_url }})** — understand why the arena is built this way.
