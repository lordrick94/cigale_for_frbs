# Risks, bugs, and open questions

Found while reading the code for this planning exercise. Confirmed items
are real bugs observed directly in the code (against the PypeIt checkout at
`/home/lordrick/Projects/F4_Repos/PypeIt`); everything else is a design
question worth resolving before writing new integration code.

## Confirmed bugs (fix regardless of which integration option is chosen)

1. **Kernel-type typo blocks the single-filter photometric-prior path.**
   `host_model.py` (`model_host_profile_prior`, single-band branch) calls
   `GP(kernel_type="HostProfie", ...)` — missing the "l". `gp.py`'s
   `_build_gp`/`_build_kernel` only recognizes `"HostProfile"`, `"1D"`,
   `"2D"` and raises `ValueError` otherwise. This means: whenever only one
   archival filter loads successfully (e.g. a field only covered by one
   PS1/Legacy-Survey band, or a download failure for the others), the
   photometric-prior fit crashes. **Trivial one-character fix.**

2. **`fit_profile` import path doesn't match the current PypeIt version.**
   `spec_proc.py` does `from pypeit.core import extract` then calls
   `extract.fit_profile(...)`. In the checked-out PypeIt,
   `fit_profile` lives in `pypeit.core.spatialprofile`, not
   `pypeit.core.extract` (confirmed: `pypeit/core/extract.py` has no such
   attribute; `pypeit/core/skysub.py` itself imports
   `from pypeit.core import spatialprofile` and calls
   `spatialprofile.fit_profile`). This breaks the
   `extraction="optimal"` path of `SpecData.write_pypeit_spec1d` against
   the local PypeIt checkout — README claims v1.17 compatibility
   ("should work with v1.14+"), so this looks like real version skew, not
   a hypothetical. **Needs a fix + a decision on which PypeIt version(s)
   to actually support and test against going forward** (this bears
   directly on Option B in
   [02_integration_options.md](02_integration_options.md): a PypeIt
   `Extract` subclass is exposed to exactly this kind of drift, continuously,
   not just at install time).

## Documentation/code mismatches (lower priority, but worth a pass)

- `docs/reference/spectrographs.rst` lists only Keck/LRIS, MMT/Binospec,
  NOT/ALFOSC as supported; `spec_proc.py` also implements VLT/FORS2 header
  parsing. Either the docs are stale or FORS2 support is unfinished/
  unvalidated — worth checking which, since it changes how much you trust
  the FORS2 code path if you extend it.
- `docs/reference/parameters.rst` is essentially an empty stub despite
  being referenced from the docs index.
- `docs/cookbook.rst` documents two known-fragile behaviors that a deeper
  integration should ideally design away rather than continue working
  around: (1) with `no_local_sky=True`, "PypeIt may have trouble finding
  and tracing the transient, especially if the trace is significantly
  curved"; (2) `pypeit_coadd_2dspec` sometimes rejects `*_hostsub.fits`
  files for frame alignment, needing `[coadd2d] offsets = 0,0,0` as a
  workaround.

## Design/scope questions worth resolving before implementation

1. **Is Option B (in-line integration) actually worth the maintenance
   cost right now?** It couples HostSub_GP to PypeIt's internal `Extract`
   API, which just demonstrated it can drift under you (bug #2 above).
   Given HostSub_GP is a young, actively-changing research codebase, is it
   premature to take on that coupling before the standalone tool has
   stabilized? (Recommendation in
   [02_integration_options.md](02_integration_options.md) is to treat
   Option A as the near-term priority for exactly this reason.)
2. **GPU support is explicitly "to be supported soon" per the docs, not
   implemented.** If GP fitting is/becomes a runtime bottleneck at scale
   (many exposures, many sources), that's a prerequisite to check before
   committing to a specific integration timeline — no point optimizing
   pipeline plumbing if the GP fit itself needs a hardware-acceleration
   pass first.
3. **`extract_sci`'s extraction method is flagged by the original authors
   as provisional** (`spec_model.py`: `# TODO: adopt the extraction
   method of pypeit`) — the current `marginalize`-based sum/mean
   extraction versus PypeIt's optimal/boxcar extraction
   (`_extract_pypeit_optimal` already exists as an alternate path). Worth
   deciding whether Option B should standardize on PypeIt's own
   optimal-extraction machinery from the start (cleaner integration) or
   preserve HostSub_GP's current extraction as the default.
4. **Should the IFU/SlicerIFU/Fiber pypelines be an explicit
   non-goal, or a stated future phase?** Worth writing down one way or the
   other so scope doesn't creep silently — see
   [02_integration_options.md](02_integration_options.md) for why it's a
   substantially larger project than the long-slit case.
5. **Instrument-specific workarounds with no in-code justification**, e.g.
   `spec_proc.py`'s hard-coded MMT/Binospec "faulty rows"
   list (`[113, 210, 719, 1999, ...]`) patched by row-averaging. Worth
   tracking down the origin (detector defect map? empirical finding?) and
   documenting it, especially before trusting it in an automated in-line
   pipeline step where there's less opportunity for a human to notice
   something looks off.
6. **Ad hoc config value coercion in `_default_pypeit_local_sky`**
   (`str(cfg["enabled"]).lower() == "true" if isinstance(...)` pattern) —
   symptomatic of the `hostsub.txt` input file's values arriving as
   strings sometimes and native types other times. If Option B moves
   configuration into PypeIt's own `PypeItPar` system, this class of
   bug should disappear naturally (PypeIt's parameter system already
   handles type coercion); worth treating as one more small argument in
   favor of B over continuing to extend the standalone `.txt` config
   format.
