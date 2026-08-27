# Integration options

Two realistic strategies, not mutually exclusive — Option A is closer to
"finish and harden what exists," Option B is closer to "make HostSub_GP a
real PypeIt pipeline stage." Recommendation: do A first (it's mostly bug
fixing and workflow polish), treat B as a longer-term project gated on
whether the manual multi-command workflow actually becomes a real pain
point for users.

## Option A — Harden the post-hoc workflow (low risk, no PypeIt core changes)

Keep the current architecture (run PypeIt → run `hostsub_setup` → run
`hostsub` → optionally re-run `coadd2d`), but close the gaps identified in
[01_current_state.md](01_current_state.md) and
[03_risks_open_questions.md](03_risks_open_questions.md):

- Fix the two confirmed bugs (kernel-type typo, `fit_profile` import path)
  so the full feature set actually works against the current PypeIt
  version.
- Consider a single wrapper CLI/Makefile target that chains
  `run_pypeit` → `hostsub_setup` → `hostsub` → `pypeit_coadd_2dspec`, so
  users don't have to remember the exact sequence and flags (especially
  `no_local_sky=True` and `[coadd2d] offsets = 0,0,0`).
- Pin/document the exact PypeIt version range this is tested against
  (README currently claims v1.17, "should work with v1.14+" — worth
  verifying against CI rather than asserting).

This is almost pure engineering hygiene — no new architectural decisions,
and it's the right thing to do regardless of whether Option B ever
happens.

## Option B — In-line PypeIt pipeline step

Insert host-galaxy GP subtraction as an actual stage of PypeIt's own
reduction, so it runs inside `run_pypeit` rather than as a separate
command.

### Where it would hook in

PypeIt's per-detector reduction sequence (traced via
`pypeit.exposure.reduce_exposure` → `pypeit_steps.py`):

```
process_exposure          → calibrated PypeItImage per detector
findobj_on_exposure        → pypeit_steps.findobj_on_det
                              + finalize_sky_det
                              (produces: final_sky_dict, all_specobjs_find,
                               calib_slits)
extract_exposure            → pypeit_steps.extract_det
                              (calls Extract.get_instance(...).run(),
                               builds final Spec2DObj)
save_exposure                → writes spec1d/spec2d FITS
```

(`pypeit/pypeit.py:212` `reduce_all` → `pypeit/exposure.py:351` — file:line
refs are against the PypeIt checkout at
`/home/lordrick/Projects/F4_Repos/PypeIt`, re-verify against current HEAD
before implementing.)

The natural insertion point is inside `pypeit_steps.extract_det`, between
`finalize_sky_det` (global sky already computed) and the call to
`Extract.get_instance(...).run()`. At that point you already have:
`sciImg` (the calibrated 2D image, a `PypeItImage`), `slits`
(`SlitTraceSet`), `final_sky` (global sky model), and `sobjs_obj`
(object-finding results) — exactly the inputs `SpecData.from_pypeit`
currently reconstructs by *re-reading* an already-written `spec2d` file.
Hooking in here means HostSub_GP would operate on in-memory PypeIt objects
instead of round-tripping through FITS.

### Two ways to actually wire it in, in increasing order of invasiveness

1. **Pre-subtract, then hand off to PypeIt's existing extraction.** Fit the
   GP host model against `sciImg`/`final_sky`/`slits`, subtract it from
   `sciImg.image` in the host region, then let PypeIt's existing
   `MultiSlitExtract.local_skysub_extract` (which wraps
   `pypeit.core.skysub.local_skysub_extract`) run unmodified on the
   corrected image. Smallest behavioral surface — PypeIt's own local sky
   fit and extraction still happen, just against host-subtracted data
   instead of raw data. This is the most direct analog to today's
   `no_local_sky=True` + external-subtraction workflow, just moved earlier
   in the pipeline and without the manual disable flag.
2. **New `Extract` subclass that replaces local-sky-and-extraction
   entirely.** A `HostSubExtract` (or similar) implementing the same
   `local_skysub_extract(...)` contract as `MultiSlitExtract` — return
   `(skymodel, bkg_redux_skymodel, objmodel, ivarmodel, outmask, sobjs)` —
   but internally doing the full GP fit + extraction from
   `spec_model.py`/`gp.py` instead of PypeIt's B-spline joint sky/object
   fit. This follows PypeIt's existing pypeline-dispatch convention
   (`Extract.get_instance()` picks a subclass by pypeline name), so it
   would need either a new selection mechanism (a `par['reduce']['skysub']`
   flag, e.g. `use_gp_hostsub = True`, checked in `Extract.get_instance()`
   alongside the existing pypeline check) or a genuinely new "pypeline"-like
   dispatch key.

Option B.1 is meaningfully less work and less risky than B.2 — it reuses
PypeIt's existing extraction unchanged and only touches where GP
subtraction happens in the timeline. B.2 gives more architectural
integration (config-driven, no external CLI step at all) but means
maintaining a PypeIt `Extract` subclass in lockstep with PypeIt's own API,
which is exactly the kind of thing that broke once already (see the
`fit_profile` import-path drift in
[03_risks_open_questions.md](03_risks_open_questions.md)) — a real ongoing
maintenance cost.

### Prerequisite either way: does the GP model need object-finding done first?

`SpecModel._build_spat_filter` needs a trace (the `mask` region) to define
where the host/sky regions are. In the current post-hoc workflow this
comes from a `SpecObj` already produced by PypeIt's object finding. In an
in-line integration, this dependency is still there — `sobjs_obj` from
`findobj_on_exposure` must run *before* the host-subtraction step,
regardless of which of B.1/B.2 is chosen. This is not a blocker, just a
sequencing constraint to keep in mind (host subtraction cannot move earlier
than object finding in the pipeline).

## Scope boundary: long-slit (MultiSlit) only

Both options assume PypeIt's **MultiSlit** pypeline. PypeIt also has
Echelle, SlicerIFU, and Fiber pypelines with materially different slit/trace
geometry (`SlicerIFUFindObjects`, `FiberExtract`, etc.). HostSub_GP's core
geometric assumption — a single 1D spatial trace split into
mask/host/sky regions — does not map onto IFU/fiber data, where "the slit"
is itself a 2D spatial slice. The `playground/MUSE/` code validates the
longslit algorithm using MUSE IFU cubes as a synthetic testbed (mocking a
narrow long-slit by averaging a strip of cube pixels); it does not exercise
or extend PypeIt's actual IFU reduction path. Extending to real IFU/fiber
data reduced through PypeIt's `SlicerIFU`/`Fiber` pypelines would be a
substantially larger project (different spatial-region definition, likely
a genuinely 2D-in-space GP rather than 1D-spatial-slice) — worth explicitly
scoping *out* of any near-term plan unless there's a specific IFU use case
driving it.
