# CIGALE Wrapper – Quick Start

This guide shows how to run **CIGALE** via a small Python wrapper and helper functions used in the accompanying notebook (`cigale_tests.ipynb`). It assumes you will pull public photometry via the **FRB** repo utilities and then run CIGALE on a single galaxy.

---

## 1) Environment setup

Create and activate a fresh conda env (Python ≥3.10 recommended):

```bash
conda create -n cigale-env python=3.11 -y
conda activate cigale-env
```

Install core dependencies (Astropy, NumPy, Jupyter) and the **FRB** repo (editable dev install is convenient):

```bash
git clone https://github.com/FRBs/FRB.git
pip install -e FRB
```

Install **CIGALE** following its documentation (see: [`cigale documentation`](https://cigale.lam.fr/documentation/)):


## 2) Required data (NEDLVS photometry)

The notebook expects a local FITS table with public photometry (NEDLVS). Download it and note the local path:

- NEDLVS FITS: (dowload [`NEDLVS FITS`](https://drive.google.com/file/d/16Xn0UJLnfxc8F8WDB5Ulm7uDf4-o23Fv/view?usp=drive_link))

Then **set the environment variable** so FRB’s survey utilities can find it. You can set this in your shell *or* at the top of the notebook:

**Shell (bash):**
```bash
export NEDLVS=/path/to/nedlvs_public_photometry_file.fits
```

**Notebook (Python):**
```python
import os
os.environ["NEDLVS"] = "/path/to/nedlvs_public_photometry_file.fits"
```

---

## 3) What the notebook does

The notebook uses:
- `frb.surveys.survey_utils.search_all_surveys(...)` to gather public photometry around a sky position.
- `frb.galaxies.nebular.get_ebv(...)` and `frb.galaxies.photom.correct_photom_table(...)` to apply foreground extinction.
- `frb.surveys.catalog_utils.convert_mags_to_flux(...)` to convert magnitudes to flux densities.
- A thin `cigale` wrapper (`from cigale import run`) that writes a `pcigale` configuration, data file, and then executes CIGALE.

Inputs you provide in the notebook:
- `gal_name` (string; either resolvable name or your own label)
- `gal_z` (float; galaxy redshift)
- Sky position: by default the notebook calls `SkyCoord.from_name(gal_name)` which uses an online name resolver.  
  - If this fails or you prefer to be explicit, pass coordinates directly:
    ```python
    from astropy import units as u
    from astropy.coordinates import SkyCoord
    coord = SkyCoord(ra=<RA_deg>*u.deg, dec=<DEC_deg>*u.deg, frame="icrs")
    ```

---

## 4) Minimal run example (mirrors the notebook)

```python
# 1) Basic inputs
gal_name = "J060740.40+433447.57"   # your galaxy name or ID
gal_z = 0.2411                      # your galaxy redshift

# 2) Coordinates
from astropy.coordinates import SkyCoord
coord = SkyCoord.from_name(gal_name)   # or construct from RA/Dec directly

# 3) Get public photometry (radius can be tuned, default 1 arcsec)
from astropy import units as u
from frb.surveys import survey_utils as su
from frb.galaxies.photom import correct_photom_table
from frb.galaxies.nebular import get_ebv
from frb.surveys.catalog_utils import convert_mags_to_flux

# Helper (simplified from notebook): pulls photometry, applies E(B–V), converts mags→flux
def get_public_photometry(gal_name, gal_coord, radius=1*u.arcsec):
    phot_tbl = su.search_all_surveys(gal_coord, radius, include_radio=False)
    if len(phot_tbl) == 0:
        return {}
    phot_tbl.sort('separation')
    phot_tbl['Name'] = 'HGXXXXXXXXX'
    phot_tbl[0]['Name'] = gal_name
    ebv = get_ebv(gal_coord)['meanValue']
    correct_photom_table(phot_tbl, ebv, name=gal_name)
    flux_tbl = convert_mags_to_flux(phot_tbl)
    # … assemble a dict of band, flux, flux_err, etc., including EBV if desired
    # (See notebook for the full implementation.)
    return {"photometry_table": phot_tbl, "flux_table": flux_tbl, "EBV": ebv}

photom = get_public_photometry(gal_name, coord)

# 4) Run CIGALE via the wrapper
from cigale import run    # provided by your wrapper repo/module
def run_cigale(photom, gal_z, gal_name,
               data_file="cigale_in.fits", config_file="pcigale.ini",
               wait_for_input=False, save_sed=True, plot=True, outdir="out", **kwargs):
    # Prepares a table with required columns and calls `run(...)` (wrapper)
    # (Full implementation is in the notebook.)
    return run(photometry_table=..., zcol="redshift",
               data_file=data_file, config_file=config_file,
               wait_for_input=wait_for_input, save_sed=save_sed,
               plot=plot, outdir=outdir, **kwargs)

run_cigale(photom, gal_z, gal_name)
```

Key arguments supported by the wrapper’s `run_cigale(...)` (from the notebook):
- `data_file`: root filename for the CIGALE input table (`.fits`).
- `config_file`: root filename for the generated pcigale configuration (`.ini`).
- `wait_for_input`: if `True`, pauses so you can edit the generated config before running.
- `save_sed`: if `True`, stores best-fit SEDs.
- `plot`: if `True`, produces SED plots.
- `outdir`: output directory (default `"out"`).

---

## 5) Outputs you should expect

CIGALE (pcigale) produces an output folder (`out/` by default) containing, e.g.:
- The configuration used and the input photometry file (`pcigale.ini`, `cigale_in.fits`).
- Best-fit parameters and derived quantities (CSV/FITS tables).
- Optional SED plots if `plot=True`.
- Optional SED FITS if `save_sed=True`.

> Tip: If you set `wait_for_input=True`, open the generated `pcigale.ini`, adjust modules and parameter grids (e.g., SFH, attenuation law, dust emission), save, then continue the notebook cell.

---

## 6) Common pitfalls & fixes

- **Name resolver fails:** use explicit RA/Dec with `SkyCoord(...)` instead of `SkyCoord.from_name(...)`.
- **No photometry returned:** increase the matching radius (e.g., `radius=2*u.arcsec`) or verify `NEDLVS` path is set and readable.
- **Missing bands/columns:** ensure FRB utilities are up-to-date and that `convert_mags_to_flux(...)` ran without errors.
- **CIGALE install issues:** verify you’re using the same Python version inside your conda env; consult CIGALE’s docs for platform-specific steps.
- **Wrapper import error (`from cigale import run`):** confirm the wrapper repo is installed (`pip install -e .`) or that its path is on `PYTHONPATH`.

---

## 7) (Optional) Save inputs to JSON

If you want to persist your inputs (galaxy name, z, coordinates, photometry summary):

```python
import json
payload = dict(gal_name=gal_name,
               gal_z=gal_z,
               ra=coord.ra.deg, dec=coord.dec.deg,
               ebv=photom.get("EBV", None))
with open("cigale_inputs.json", "w") as f:
    json.dump(payload, f, indent=2)
```

---

## 8) Repro tips

- Keep the exact versions: `pip freeze > requirements.txt` inside your env.
- Record the FRB repo commit if you’re using the latest main branch.
- Save the generated `pcigale.ini` alongside your outputs.