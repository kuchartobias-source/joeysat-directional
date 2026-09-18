# Data for the JoeySat directional-analysis website

Everything here is precomputed. The browser does no analysis: it reads these files and
draws them. Sliders stop only on grid points; nothing is interpolated between them.

All of it comes from one place, the reproduction package
`directional_analysis/` — `steps/s02_extract.load_clusters()` for the population and
`lib/fitting.bootstrap_slope()` / `band_profile()` for every slope and band. The
minimum-clusters-per-band threshold is passed explicitly everywhere (30); no call relies
on a default.

**No cluster-level data appears in any file.** Only medians, counts, standard errors,
slopes, intervals, rasters and histograms. The map's median-β layer is additionally
suppressed below 20 clusters in a cell, so no cell can identify a cluster.

## The analysis these files describe

```
data      data_cache.parquet              raw cache; h1cleaned is NOT used
phase I   2024-01-11 .. 2024-10-19        polygon saa_polygon_ap9_600km.json
phase II  from 2025-02-18                 polygon saa_polygon_ap9_1175km.json (native)
contour   AP9 > 10 MeV, level 0.003 of the spatial maximum, antimeridian wrap ON
Size >= 2                                 a one-pixel cluster has no axis
W2D < 1.8    occupancy < 32               nominal; both are slider axes
regions by signed distance d from the contour (negative inside):
    core  d <= -b     guard  -b < d < +b  (excluded)     reference  d >= +b
    nominal b = 10; at b = 0 core is simply inside and reference outside
|I| bands  width 6, centres 27..63, range 24..66
per band   median β, SE from a 1000-replicate bootstrap, 68 % interval
fit        weighted straight line, w = 1/SE²; a band with n < 30 is drawn but NOT fitted
           a fit exists only with >= 4 fitted bands spanning >= 18 deg, else null
```

Angles and slopes are rounded to 4 decimals on writing; counts are integers; a missing
value is `null`, never `NaN`.

### One boundary cluster

One phase-II cluster sits exactly on the `W2D < 1.8` boundary: it stores as `1.79999995`
in float32, so whether it passes depends on whether the comparison happens before or
after the cast. The nominal count of 43313 uses the extraction's own `fid` arrays, which
is the article's number; this is commented at the point in the code where it matters.

## Files

| file | what it holds |
|---|---|
| `explorer_grid.json` | the W2D × b × occ grid, 4680 records |
| `scans.json` | 1D scans: contour level × b, LET quintiles, |I| range |
| `isotropic.json` | the isotropic control, averaged over 40 seeds |
| `map_phase_I.json`, `map_phase_II.json` | per-phase map layers |
| `map_common.json` | coastlines, shared by both phases |
| `profiles_full.json` | median β over the full inclination range |
| `site_numbers.json` | every number quoted in the site text, with its source |
| `ops.json` | exposure, occupancy and cadence diagnostics |
| `figures.json` | the robustness gallery: captions and provenance |
| `manifest.json` | definitions, axes, seeds, grid statistics, check results |
| `data.js` | all of the above wrapped as `window.JOEYSAT` |

`data.js` exists because the page is opened over `file://` at first, where browsers
refuse `fetch()` on local JSON. The JSON files remain the canonical source; `data.js` is
a generated wrapper around exactly the same objects.

### `explorer_grid.json`

```
{"_meta": {axes, phases, contour_level, n_records},
 "records": {"<w2d>|<b>|<occ>|<phase>": { ... }, ...}}
```

The key is the four slider positions joined by `|`: W2D with one decimal, `none` where a
cut is absent, e.g. `"1.8|10|32|I"` or `"none|0|none|II"`. One record:

```
"w2d": 1.8 | null      "b": 10.0      "occ": 32 | null      "phase": "I"
"n":   {fiducial, core, guard, reference}        core+guard+reference == fiducial
"core", "reference": {
   "bins": {"center": [7], "n": [7], "median": [7], "se": [7], "fitted": [7 bool],
            "median_sparse": [7], "se_sparse": [7]}
   "fit":  {"slope", "lo", "hi", "n_bins", "span", "intercept}  or null
}
```

`median` and `se` are `null` in a band with fewer than 30 clusters — such a band does not
enter the fit. `fit` is `null` when the usability rule fails; the band data is still
there, so the page can show why.

`median_sparse` and `se_sparse` carry the median of exactly those under-populated bands
(`1 <= n < 30`), so the page can draw them as hollow markers the way Fig. 3 of the article
does. They are `null` wherever the band is fitted or empty — the two pairs never overlap.
`se_sparse` is additionally `null` below five clusters, where bootstrapping a median means
nothing. **Neither field enters the fit**: `fitted` and `fit` are computed from the counts
alone and are unchanged by their presence.

### `scans.json`

Three keys, each a map to records shaped like the grid's:

- `level_x_b` — `"<level>|<b>|<phase>"`, contour level in {0.002, 0.003, 0.005} against
  the whole b axis at W2D 1.8, occ 32. This is what carries the halo argument: the
  phase-II reference at b = 0 runs from −0.145 to −0.174 across the three levels.
- `let_quintile` — `"Q1".."Q5"` per phase, each with `let_edges` in keV/µm.
- `i_range` — `"24_66"` (nominal, 7 bands) and `"18_72"` (9 bands, centres 21..69).

### `isotropic.json`

Per phase: `bins` over nine centres (21..69) with the mean and spread of the band median
over seeds, plus `slope_mean`, `slope_sd`, `n_seeds`, `level`.

The slope is fitted over the **nominal seven** centres, and `slope_fitted_over` says so.
This matters: `isotropic_prediction()` chains all bands through one generator, so asking
for nine centres draws a different random stream and gives a different spread. The seven
-centre call is the one the article's statement refers to. Seeds are 124000..124039.

The control is a **null hypothesis, not a correction**. Nothing anywhere is normalised by
it, and no single seed's value is quotable on its own.

### `map_phase_*.json`

- `contours` — the polygon at each of the three levels, as a list of rings already split
  at the antimeridian so longitudes stay in [−180, 180]. Phase I gives one ring, phase II
  three, because its contour genuinely crosses the seam (its vertices run to lon −192.5).
- `signed_distance` — 180 × 360 raster on 1° cells, integers in tenths of a degree,
  negative inside the contour, `missing` where undefined. The page draws the guard band
  for any b as |d| < b from this, with no extra data.
- `abs_inclination` — same grid, |I| at that phase's altitude and epoch.
- `inclination_isolines` — |I| contours at 24..66, split at the seam.
- `proton_density` — the fiducial selection only. `count` on 2° cells; `median_beta` on
  **5° cells** with `min_cell_n = 20`. The median needs the coarser cell: at 2° the
  busiest phase-I cell holds 18 clusters, so a 20-cluster floor would leave that layer
  empty.
- `layers` — the same rasters for other selections, so the map can follow the sliders.
  - `layers.all` — **every** proton with an elevation angle and `Size >= 2`, with no width
    or occupancy limit: 444,737 (phase I) and 746,907 (phase II), against the 24,461 and
    43,313 that survive the fiducial cuts. This is the introductory map; the fiducial
    selection is one position of the sliders, not the default view. Carries `count_2deg`
    (2° cells) as well as the median.
  - `layers.w2d` — one raster per point of the W2D axis, at occupancy < 32.
  - `layers.occ` — one raster per point of the occupancy axis, at W2D < 1.8.

  `layers.w2d["1.8"]` is element-for-element identical to `proton_density.median_beta`,
  which is the check that both are computing the same thing.

**Every median-β raster on the map — `proton_density` and all three `layers` — is in
TENTHS OF A DEGREE as an integer**, with the value in `missing` (−32768) where the cell is
below `min_cell_n` or empty. Divide by 10 to get degrees. Counts are plain integers.

The median is a **median**, not a mean, in every layer, matching the statistic used
throughout the analysis.

### `profiles_full.json`

Per phase, `core` and `reference` over 22 bands of 3° from 0 to 66, plus `coarse` over 9
bands of 6° from 18 to 72. Each carries `center`, `n`, `median`, `se`. **No fit** — this
is a curve drawn against the three predicted behaviours, not a slope measurement. `se` is
the bootstrap spread of the band median, the same 1000 replicates used everywhere else.

### `ops.json`

`tacq_median` (median frame exposure per 5° cell, seconds, `null` where no frame),
`occupancy_hist` (frames per occupancy band, per region, at b = 10), `duty_cycle_pct`,
`frames_per_day`, `live_time_core_h`.

### `figures.json`

One entry per gallery tile: `file`, `svg`, `title`, `caption_draft`, `selection`,
`regenerated`, `script`, `generated`. Every tile was regenerated on the current
definition; none was copied from the archive.

## Reproducing

With the package present and its extraction built:

```
python w1_task_a_par.py --workers 8     # the grid  (resumable; rerun to continue)
python w1_task_a.py --pack              # grid -> explorer_grid.json
python w1_task_bce.py                   # scans, isotropic control, site numbers
python w1_task_d.py                     # map layers
python w1b_ghj.py                       # full profiles, coastlines, ops
python w1b_gallery.py                   # the ten gallery figures
python w1b_pack.py                      # manifest, data.js, sizes, check 11
```

The grid writes each record to a JSON-lines shard as it is computed and fsyncs after
every cell, so it can be interrupted at any moment and resumed with the same command.
Records are independent functions of their parameters and a fixed seed, so the parallel
run reproduces the sequential one exactly; this was verified on six complete cells (60
records, all byte-identical) before the parallel run was used.
