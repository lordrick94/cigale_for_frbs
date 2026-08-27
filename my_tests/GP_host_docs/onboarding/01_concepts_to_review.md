# Concepts to review before working on HostSub_GP

HostSub_GP (Liu & Miller 2025, [arXiv:2508.15278](https://arxiv.org/abs/2508.15278))
removes host-galaxy light from a long-slit transient spectrum using a 2D
Gaussian Process (GP) model trained jointly on the spectrum and archival
imaging. The math/stats content is the steepest part of the learning curve;
the spectroscopy content you likely already have from CHIME/FRB host-galaxy
work. This doc is ordered roughly by how central the concept is to the code,
not by difficulty.

## 1. Gaussian Process regression (core — spend the most time here)

This is the load-bearing method. Everything else in the pipeline exists to
build inputs for, or consume outputs from, a GP fit.

- **What a GP is**: a distribution over functions, specified by a mean
  function `m(x)` and a covariance/kernel function `k(x, x')`. Conditioning
  on observed data gives a posterior mean (the prediction) and posterior
  variance (the uncertainty) at any new input.
- **Marginal likelihood and hyperparameter fitting**: GP hyperparameters
  (kernel amplitude, length scales, mean offset) are usually fit by
  maximizing the log marginal likelihood, not by cross-validation. In this
  code that's `GP._optimize`/`_neg_log_prob` (`src/hostsub_gp/gp.py`),
  minimized with `jaxopt.ScipyBoundedMinimize` (L-BFGS-B).
- **Why GPs here specifically**: the galaxy's light varies smoothly in both
  space (across the slit) and wavelength, but not in a way you'd want to
  hand-pick a parametric form for. A GP lets you say "smooth, with these
  physically motivated length scales" without fitting an explicit galaxy
  model (Sersic profile, etc.).
- **Reading**: Rasmussen & Williams, *Gaussian Processes for Machine
  Learning* — chapters 2 (regression), 4 (kernels), 5 (hyperparameter
  learning) are directly applicable and free online. The `tinygp` docs
  (the JAX GP library this repo uses) are a faster, code-first alternative.

## 2. Kernel design and covariance structure

Once GP regression itself is familiar, the interesting engineering in this
repo is almost entirely in kernel construction (`gp.py:_build_gp`):

- **Matérn and squared-exponential (RBF) kernels**: standard smoothness
  priors. Matérn kernels are less "infinitely smooth" than RBF and are
  generally preferred for physical data with realistic small-scale
  structure — that's why they dominate here (`Matern32`, `Matern52`).
- **Separable / product kernels**: `k((s,λ),(s',λ')) = k_spatial(s,s') ×
  k_spectral(λ,λ')` — treats spatial and spectral covariance as
  independent, which is both a modeling assumption and a computational
  trick (it lets you build a 2D kernel out of cheap 1D kernels). See
  `OneDKernel` in `gp.py`.
- **Quasi-separable / structured kernels for 1D**: `tinygp.kernels.quasisep`
  gives O(N) instead of O(N³) scaling for 1D GPs — worth knowing this
  exists as a performance technique, used for the purely-spectral
  (`kernel_type="1D"`) host-shape GP.
- **Anisotropic kernels via linear transforms**: `transforms.Linear`
  rescales/rotates the input space before applying an isotropic kernel —
  how the 2D residual GP gets independent spatial vs. spectral length
  scales from what's nominally a single Matérn32.
- **Non-stationary / custom kernels**: the `EmissionLineKernel` (`gp.py`)
  is the one genuinely novel piece of kernel engineering in this codebase —
  it locally suppresses covariance between points that straddle a known
  emission line, so the smooth GP doesn't blur sharp line flux into
  neighboring wavelengths. Understanding *why* a stationary kernel would
  fail here (a single global length scale can't be both "smooth continuum"
  and "sharp line" at once) is the key insight — this is a case where
  domain knowledge (a line list) directly determines covariance structure,
  not just data.

## 3. GP mean functions as priors / informative Bayesian priors

A GP's mean function is often set to zero by default, but here it's set to
the (rescaled) archival-imaging-derived host profile — i.e., the GP models
*residuals from a photometric prior*, not the raw galaxy flux
(`get_host_prior`, `spec_model.py`). This is standard practice when you have
an informative prior available (external imaging) and want the GP to only
have to explain what the prior gets wrong — smaller residual variance,
better-constrained fits, less data needed. Worth internalizing the general
pattern: `posterior_target = prior_model + GP(residual)`.

Hyperparameter *bounds* are also set from physical quantities (seeing FWHM,
spectral resolution) rather than left unconstrained — a soft-informative-
prior approach via optimization bounds rather than an explicit Bayesian
prior distribution. Useful to recognize this as a lightweight alternative
to full Bayesian hierarchical modeling (no MCMC, no posterior over
hyperparameters — just constrained MAP/maximum-likelihood point estimates).

## 4. Robust statistics

- **MAD (median absolute deviation)** as a robust proxy for standard
  deviation, used throughout for outlier/cosmic-ray rejection
  (`mad_std`, `sigma_clip` in `spec_wrapper.py`). Useful because spectral
  data has cosmic rays and bad pixels that would blow up an ordinary
  standard deviation.
- **Sigma clipping** as an iterative outlier-rejection scheme — standard,
  but worth knowing the specific MAD-based variant used here so you
  recognize it when reading `coadd2d`'s cross-frame CR rejection.

## 5. PSF / seeing matching

`_match_seeing`/`update_seeing` (`spec_model.py`) fit a wavelength-dependent
difference in seeing (FWHM) between the spectrum's PSF and the archival
images' PSF, using a Kolmogorov-turbulence motivated scaling
(FWHM ∝ λ^(−1/5)), and resolve it by Gaussian-convolving whichever of the
two is sharper. If you haven't worked with PSF-matching before: the general
technique (fit a convolution kernel width that minimizes χ² between two
images/spectra of the same object) shows up constantly in image
differencing (e.g. difference imaging for transient detection) — worth
recognizing as a reusable pattern beyond this repo.

## 6. Emission-line detection and cross-correlation redshifts

`_find_host_emission` (`spec_model.py`) flags emission lines via a
3σ-above-continuum flux test plus a spatial-profile χ² test, then
identifies which lines they are via cross-correlation against a reference
line list (`data/Emission_line_list.csv` — [OII], Balmer series, [OIII],
[NII], [SII]) to estimate a redshift. If cross-correlation redshift
estimation is unfamiliar: the idea is to shift a template line list over
a grid of trial redshifts and find the shift that best matches detected
line wavelengths, rather than fitting one line at a time.

## 7. Synthetic photometry (validation tooling, not production code)

`playground/MUSE/muse_synphot.py` integrates a MUSE IFU cube against
PanSTARRS filter throughput curves (`∫ cube(λ) × T(λ) dλ`) to derive a
synthetic broadband image directly from spectroscopic data. This is a
standard technique (synthetic photometry from spectra) used here purely to
build a MUSE-based validation testbed with known ground truth — it is not
part of the PypeIt-facing production pipeline. Skip unless you're working
on the validation code specifically.

## 8. Astrometric registration

Archival cutouts are reprojected onto slit-aligned pixel coordinates via
`reproject.reproject_adaptive` (WCS-to-WCS resampling), with Astrometry.net
plate-solving as a fallback (`_utils/_astronometry.py`). Standard
astrometric-registration concepts (WCS, reprojection/regridding) — likely
already familiar from FRB host-imaging work, flagged here mainly so you
know where in the code it happens.

## 9. JAX (engineering prerequisite, not science, but required to read the code)

The whole GP/optimization stack is built on JAX, not NumPy/SciPy/sklearn.
Before reading `gp.py`, `interp.py`, or the batching logic in
`spec_model.py`, it's worth 30–60 minutes with JAX's core idioms if you
haven't used it:

- Arrays are immutable — in-place-looking updates are actually
  `x.at[idx].set(...)` (functional update, returns a new array).
- `jax.jit` (compilation), `jax.vmap` (vectorizing a scalar/1-input
  function over a batch dimension), `jax.grad`/autodiff (what makes
  gradient-based optimizers like L-BFGS-B fast here — no numerical
  finite-differencing needed).
- `jaxopt.ScipyBoundedMinimize` — a SciPy-compatible bounded optimizer
  wrapper that uses JAX-computed gradients under the hood.

Without this, `gp.py` and the batching code in `spec_model.py` will read as
unfamiliar NumPy with strange syntax rather than as a coherent pattern.

## Suggested reading order

1. JAX basics (§9) — needed just to parse the code syntactically.
2. GP regression fundamentals (§1) — the conceptual core.
3. Kernel design (§2) — where this repo's actual novelty lives.
4. Mean functions as priors (§3) — how the photometric prior and GP fit
   together.
5. Everything else (§4–§8) as you encounter it in the code — these are all
   standard, well-known techniques and don't need dedicated study time
   up front.

See [pipeline_overview/](pipeline_overview/README.md) for how these pieces
compose into the actual pipeline, and
[pypeit_integration_planning/](pypeit_integration_planning/README.md) for
what integrating this with PypeIt would involve.
