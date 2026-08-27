# Onboarding: HostSub_GP + PypeIt integration

Generated in response to the TODOs in [../explanation.md](../explanation.md).
Three documents/folders, matching the three requested deliverables:

1. [01_concepts_to_review.md](01_concepts_to_review.md) — science/stats/ML
   concepts to review before working on this codebase (Gaussian Process
   regression, kernel design, robust statistics, PSF matching, JAX).
2. [pipeline_overview/](pipeline_overview/README.md) — what HostSub_GP
   does end to end, with diagrams (pipeline flow, GP model composition,
   module dependencies, spatial-region layout).
3. [pypeit_integration_planning/](pypeit_integration_planning/README.md) —
   planning docs for integrating this with PypeIt: current state, two
   integration options with concrete PypeIt file:line anchors, known bugs
   and open questions, and a phased roadmap.

Read in that order if you're starting from zero; jump straight to (3) if
you already know the codebase and just want the integration plan.
