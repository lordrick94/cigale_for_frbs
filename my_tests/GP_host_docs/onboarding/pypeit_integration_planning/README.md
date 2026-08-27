# Planning: integrating HostSub_GP with PypeIt

Index:

1. [01_current_state.md](01_current_state.md) — what already works today
   (the post-hoc file-based integration), and its limits.
2. [02_integration_options.md](02_integration_options.md) — the two
   plausible integration strategies, with concrete file:line anchors into
   PypeIt for where each would hook in.
3. [03_risks_open_questions.md](03_risks_open_questions.md) — real bugs
   found during the code read, version-skew issues, IFU-vs-longslit scope
   limits, and questions worth resolving before writing code.
4. [04_roadmap.md](04_roadmap.md) — a phased plan, ordered so each phase
   produces something usable on its own.

Companion docs: [../01_concepts_to_review.md](../01_concepts_to_review.md)
(background) and [../pipeline_overview/README.md](../pipeline_overview/README.md)
(what the code does today, with diagrams).
