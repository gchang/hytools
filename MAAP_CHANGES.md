# Changes from upstream

Upstream: https://github.com/EnSpec/hytools, tag `1.6.1` (the latest GitHub release) at commit a3f31864ee4977d82e35321a985534cef0e88215 (2025-08-19), by Adam Chlus, Zhiwei Ye, Ting Zheng, Natalie Queally, Evan Greenberg and Philip Townsend (University of Wisconsin-Madison and JPL), GNU General Public License v3.0 (`LICENSE.txt`). Please cite HyTools as `CITATION.cff` asks.

This copy starts from upstream's file tree at a3f3186 without its git history. It is unchanged except for the items below, which are the dated notices of modification that GPL-3.0 asks for. The LICENSE, the authors' notices and `CITATION.cff` are untouched. The 1.6.1 tag has no `.github/` folder, so nothing was left out; the GitHub Actions workflow that upstream added later is not included either (see the end-to-end check below).

## 2026-10-25: normalized difference indices are computed in float

`HyTools.ndi()` (`hytools/base.py`) subtracted and added the two bands in their stored data type. Many ENVI reflectance products are integers, for example AVIRIS-Classic L2 as `uint16` reflectance x 10000. The difference then wraps around whenever band 1 is darker than band 2 (and an `int16` sum can overflow), so a negative index came out as a large positive number. Every `ndi` mask (topo, BRDF and glint calc and apply masks, trait masks) and the FlexBRDF NDVI bins were wrong on such images.

On AVIRIS-Classic line f130410t01p00r09 with the example topo, BRDF and glint config, the water-exclusion term (`ndi` 550/2150 in [-1, 0]) kept only 191 pixels, so topographic and BRDF correction silently did nothing (land output equaled input), and the glint mask missed most clear water (49,618 pixels instead of 418,350).

Integer bands are now cast to float before the arithmetic. **Results change on integer images, because the old ones were wrong.** Float images (AVIRIS-NG, EMIT, NetCDF) are not touched and give exactly the same results as before.

## 2026-10-25: glint correction no longer needs a `truncate` setting

`apply_glint_correct()` read `glint['truncate']` directly, but the example and template configs never set it, so `scripts/image_correct.py` with the shipped topo/BRDF/glint example stopped with `KeyError: 'truncate'` when exporting. A missing setting now means no truncation. This is upstream commit eb1d0d2b682543d30b83f9fad61d0af2b6b435e5 (2026-07-16), applied unchanged. Configs that set `truncate` give the same results.

## 2026-10-25: `hytools.__version__`

The package now reports its installed version as `hytools.__version__`, read from the package metadata. This is the fallback part of upstream commit 770028404f0ac2f1c3af80c9509ea25938058f88 (2026-08-24); upstream's setuptools-scm `_version.py` is left out because this release sets a static version. No processing changes.

## 2026-10-25: end-to-end check (`tests/`)

The 1.6.1 release has no tests. `tests/untar_ball.py`, `tests/exec_ht.py` and `tests/correct_setting.py` are copied unchanged from upstream commit 268f3411a40f6e2c052ab8a4f0235af86f80b6c6 (2026-09-20). They unpack the public AVIRIS-Classic line above, run Gao glint, SCS+C topographic and FlexBRDF correction in memory and save `ci_test_plot.png`. There are no assertions: the check passes when it runs to the end. The README says how to run it. Its `random_seed` setting is ignored by this release (no seeded sampler).

## Known limitations, not changed

- **BRDF on a single line.** On f130410t01p00r09 (about 14% land, 47 degree sun) the FlexBRDF fit is ill-conditioned: the Li-dense kernel is about -1 everywhere, so the fitted isotropic and geometric terms are large and nearly cancel. The correction brightens land by about 1.3x near nadir and 2.2x at the swath edge (median 1.9x at 850 nm), which is not plausible. FlexBRDF is meant for groups of lines with more land, and whether to regularize the fit or require a minimum sample is a science choice, so it is left as it is. The fit also draws a random 90% sample of pixels (`hytools/brdf/flex.py:64`, `sample_perc` 0.1) without a seed, so repeated runs differ: two runs of the same config gave a median 860 nm BRDF factor of 1.92 and 1.85.
- **Trait maps on 2013 AVIRIS-Classic.** The 1323 and 1333 nm bands are zero-filled in this line but fall outside the template's bad-band range `[1337, 1430]`; they get all-zero BRDF coefficients, the corrected value is 0/0 = NaN on land, and the three example PLSR models in `data/trait_models/` (which use those bands) produce NaN for every pixel. Starting that bad-band range at 1310 nm avoids it for this sensor.
- `scripts/image_correct.py` writes an empty `_glint_coeffs_` JSON file when coefficient export is on, and `hytools/brdf/universal.py` needs `diagnostic_plots` in the BRDF config.

## Check

On 2026-10-25, in a fresh conda environment with Python 3.10 (the version upstream's CI uses), from a clone of this repository: `python -m build && python -m pip install dist/*.whl` (numpy 2.2.6, scipy 1.15.3, scikit-learn 1.7.2, ray 2.59.0; `hytools.__version__` is `1.6.1`). In a folder with the two tarballs from https://popo.jpl.nasa.gov/avcl/y13_data/ (`f130410t01p00r09_refl.tar.gz`, 136.6 MB, and `f130410t01p00r09.tar.gz`, 225.1 MB):

```sh
python <repo>/tests/untar_ball.py f130410t01p00r09 ./ ./
python <repo>/tests/exec_ht.py ./f130410t01p00r09rdn_refl_img_corr ./f130410t01p00r09rdn_e_obs_ort ./
```

The end-to-end check ran to the end (14 s, 1.5 GB) and its plot shows glint correction over the ocean and topo/BRDF correction over the peninsula. Then `scripts/image_correct.py` with `examples/configs/topo_brdf_glint_correct_config.json` (paths, `num_cpus` 2, `export.coeffs` on and the template's bad bands filled in; no `truncate` setting) and `scripts/trait_estimate.py` with the three example trait models both exited 0 (60 s and 2.0 GB; 122 s and 1.2 GB, summed over the Ray processes, 2 cores).

Compared with the same run of the unchanged 1.6.1 code:

- Apply masks: topo 68,948 pixels (was 111), BRDF 126,849 (was 190), glint 418,350 (was 49,618). A float recomputation outside HyTools gives the same glint mask and a BRDF mask within 22 pixels (126,871).
- Glint-corrected water equals an independent recomputation of Hochberg (2003) exactly at 440, 550, 660 and 850 nm (before, it differed by up to 0.26 reflectance).
- Topo and BRDF now change the image: land median reflectance at 850 nm goes from 0.180 to 0.380 (before, unchanged), and its correlation with cos(i) goes from +0.20 to -0.07. The SCS+C factor on land has a median of 1.02; the BRDF factor is the implausible one described above.
- Trait maps are still NaN on this line, for the reason above.

The package version is static (`1.6.1` in `pyproject.toml` and `setup.py`), so it is unchanged; this commit is tagged `1.6.1.post1`.
