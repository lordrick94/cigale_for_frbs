# Phased roadmap

Ordered so each phase is independently useful even if later phases never
happen — no phase requires committing to Option B up front.

## Phase 0 — Bug fixes (do first, regardless of everything else)

- Fix `"HostProfie"` → `"HostProfile"` typo in `host_model.py`
  (`model_host_profile_prior`).
- Fix `pypeit.core.extract.fit_profile` → `pypeit.core.spatialprofile.fit_profile`
  in `spec_proc.py`, and add a regression test (or at least a manual smoke
  test) exercising the `extraction="optimal"` path so this class of
  version-skew bug is caught earlier next time.
- Pin a tested PypeIt version (or version range) in `pyproject.toml` and
  document it accurately in `README.md` — currently claims v1.17 with
  "should work with v1.14+" but hasn't been verified against the local
  checkout.
- Reconcile `docs/reference/spectrographs.rst` with actual code support
  (add FORS2, or remove/flag it as unvalidated).

## Phase 1 — Harden the existing post-hoc workflow (Option A)

- Single wrapper command/script chaining
  `run_pypeit` → `hostsub_setup` → `hostsub` → `pypeit_coadd_2dspec`,
  including the `no_local_sky=True` and `[coadd2d] offsets = 0,0,0`
  settings, so the multi-step manual process becomes one command.
- Fill in `docs/reference/parameters.rst` (currently a stub).
- Resolve or document the open questions in
  [03_risks_open_questions.md](03_risks_open_questions.md) #5–#6
  (Binospec bad-row list provenance, config value coercion) — small but
  reduces fragility before anyone builds on top of this layer.

This phase alone should meaningfully improve real-world usability without
touching PypeIt internals at all.

## Phase 2 — Prototype Option B.1 (pre-subtract, reuse PypeIt extraction)

- Build a standalone function that takes the in-memory PypeIt objects
  available at the `pypeit_steps.extract_det` insertion point (`sciImg`,
  `slits`, `final_sky`, `sobjs_obj`) and produces a host-subtracted
  `sciImg.image`, using the existing `spec_model.py`/`gp.py` machinery —
  without yet wiring it into `pypeit_steps.py` itself. Validate this
  in-memory path produces equivalent results to the current file-based
  `SpecData.from_pypeit` path, on a known test case (e.g. one of the MUSE
  playground validation cases, or a previously-processed real exposure).
- Only once that's validated: wire it into `pypeit_steps.extract_det`
  behind an opt-in config flag, so PypeIt's default behavior is unchanged
  and this is purely additive.

## Phase 3 (optional, gated on real demand) — Option B.2

Full `Extract` subclass replacing local-sky-and-extraction. Only pursue if
Phase 2's opt-in flag sees enough real use that the remaining friction
(still using PypeIt's own local-sky/extraction after GP pre-subtraction)
is a demonstrated pain point, and if there's appetite for the ongoing
PypeIt-API-coupling maintenance cost flagged in
[03_risks_open_questions.md](03_risks_open_questions.md) #1.

## Explicitly out of scope for this roadmap

- Echelle, SlicerIFU, and Fiber pypeline support (see
  [02_integration_options.md](02_integration_options.md) — a materially
  different, larger project; revisit only if a specific IFU/fiber use case
  arises).
- GPU acceleration of the GP fit itself (tracked upstream in `tinygp`/JAX,
  not a HostSub_GP-side integration task — but worth re-checking before
  Phase 2/3 if GP fit runtime becomes the bottleneck at scale).
