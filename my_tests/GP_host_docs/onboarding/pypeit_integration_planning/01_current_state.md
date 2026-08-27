# Current state: HostSub_GP as a PypeIt post-processor

HostSub_GP already integrates with PypeIt today — but entirely as a
**post-hoc, file-based** tool, not as a step inside PypeIt's own reduction
pipeline. Understanding this precisely matters because it's the baseline
any deeper integration has to improve on, and because it already tells you
which parts of PypeIt's data model HostSub_GP has committed to.

## How it works today

1. Run PypeIt normally on the science + standard exposures, producing
   `spec2d`/`spec1d` FITS files in `Science/`. The cookbook recommends
   setting `[reduce][[skysub]] no_local_sky = True` so PypeIt's own local
   sky subtraction (which can mistake host-galaxy structure for cosmic
   rays or an extra local-sky component) doesn't fight with HostSub_GP.
2. `hostsub_setup pypeit -c FILE.pypeit --target NAME`
   (`src/hostsub_gp/scripts/hostsub_setup.py`) matches standard-star and
   science `spec1d` objects by slit/detector/spatial proximity and writes
   a template `hostsub.txt` input file — reusing PypeIt's own
   `pypeit.inputfiles.InputFile` parser (`inputfiles.py`).
3. `hostsub hostsub.txt` (`src/hostsub_gp/scripts/hostsub.py`) does the
   full GP host-subtraction pipeline (see
   [../pipeline_overview/README.md](../pipeline_overview/README.md)),
   reading `Spec2DObj`/`SpecObj` via PypeIt's own I/O classes.
4. Output is written as `*_hostsub.fits` — a modified `AllSpec2DObj` with
   `skymodel` updated to include the GP host model — and optionally a new
   `SpecObj`/`spec1d` file built from the HostSub extraction
   (`SpecData.update_pypeit_skymodel`, `SpecData.write_pypeit_spec1d` in
   `spec_proc.py`).
5. Downstream, `*_hostsub.fits` files can be fed back into PypeIt's own
   `pypeit_coadd_2dspec` for multi-exposure combination (with a documented
   workaround needed: `[coadd2d] offsets = 0,0,0`, since PypeIt's frame
   alignment can complain about the modified files).

## What this buys you

- **No PypeIt core code changes required.** HostSub_GP only imports
  PypeIt's public I/O and (optionally) its `core.extract`/`core.skysub`
  functions — it never subclasses or monkey-patches PypeIt internals.
- **Reuses PypeIt's file formats end-to-end.** Anything downstream that
  consumes standard PypeIt `spec2d`/`spec1d` files (QA tools,
  `pypeit_coadd_2dspec`, custom analysis scripts) can consume HostSub_GP's
  output largely unmodified.
- **Reuses PypeIt's own extraction/sky-fit code as a fallback/comparison.**
  `_extract_pypeit_optimal` and `_fit_pypeit_local_sky` in `spec_proc.py`
  call `pypeit.core.extract.extract_optimal`/`extract_boxcar` and
  `pypeit.core.skysub.skyoptimal`/`optimal_bkpts` directly, so the classic
  PypeIt extraction/sky-fit machinery is already wired up as an
  alternative to the GP-based path.

## What this doesn't buy you (the gap a deeper integration would close)

- **Runs as a separate command, after the fact.** Users must remember to
  set `no_local_sky=True`, run `hostsub_setup`, then `hostsub`, then
  potentially re-run `coadd2d` with a workaround flag — several manual
  steps outside PypeIt's normal `run_pypeit` invocation.
  PypeIt's object-finding trace, which was fit *without* knowledge of the
  host GP model, is used as-is; a tighter integration could let object
  finding and the host model inform each other iteratively.
- **PypeIt's own local sky subtraction has to be disabled rather than
  replaced.** `no_local_sky=True` is a blunt instrument — it means PypeIt
  does no local sky subtraction at all in the disabled region, rather than
  HostSub_GP's GP model formally taking over that specific pipeline stage.
- **No access to PypeIt's per-exposure processing before final spec2d
  write.** HostSub_GP operates on already-finalized `spec2d` files; it
  can't influence, e.g., the object-finding trace, the CR mask, or the
  wavelength solution — it can only work with what PypeIt already decided.
- **Only MultiSlit-pypeline (long-slit) data is supported**, since
  `SpecData.from_pypeit` hard-codes per-instrument header parsing for
  LRIS/Binospec/ALFOSC/FORS2 and `SpecModel._build_spat_filter` assumes a
  single 1D spatial trace + mask/host/sky split. PypeIt's Echelle,
  SlicerIFU, and Fiber pypelines are untouched (see
  [03_risks_open_questions.md](03_risks_open_questions.md)).

See [02_integration_options.md](02_integration_options.md) for what
closing that gap would actually involve.
