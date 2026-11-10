# TDOA_ClosedForm_DD_MATLAB

Closed-form three-receiver TDOA localisation in **double-double (≈106-bit) arithmetic**, written for
MATLAB (base product, no toolboxes) and executed/verified with **GNU Octave 8.4.0**.
**It has not been run in MATLAB** (see `docs/compatibility.md`).

Two separate solution paths are provided:

| Mode | Constraint | Algebra | Use |
|---|---|---|---|
| `Operational_FixedENUUp` (default) | emitter local **Up = 100 m** in Receiver 1's ENU frame (a plane) | PDF (4.33)–(4.36) + plane ⇒ **quadratic** in the reference range r0 | the operational problem: receivers `[0,0,0]`, `[106.066,106.066,0]`, `[200,0,0]` m, emitter `[50,30,100]` m |
| `Reference_WGS84_H0` | emitter **on the WGS-84 ellipsoid, geodetic height 0** (PDF eq. 4.31) | PDF (4.32)–(4.47), **quartic** in r0, 8 candidates | traceability to the supplied method on a separate synthetic satellite scenario only |

Local Up = 100 m is **not** geodetic height 100 m (for the synthetic origin at h = 50 m the default emitter has
ellipsoidal height 150.000266… m). An exact-geodetic-height-100 m mode is **not implemented** (no closed form
derived; see `docs/assumptions_discrepancies.md`, A-3).

## Quick start

```matlab
% from any working directory
run('/path/to/TDOA_ClosedForm_DD_MATLAB/setup_project.m')   % or cd there and call setup_project
out = run_demo();                 % 30-pulse operational burst + reference scenario -> results/demo_*
S   = run_sweep('production');    % required 34 x 72 range/azimuth grid, 30 pulses/cell -> results/sweep_production
S   = run_sweep('accurate');      % same grid, 20 independent bursts/cell (1,468,800 solves)
S   = run_sweep('fast');          % smoke-test subgrid only
out = run_interactive();          % prompts: Receiver 1 WGS-84 position, emitter [E,N,U], sigma_ns
ok  = test_all();                 % 132 automated checks -> results/tests/test_report.txt
```

GNU Octave (headless) — figures need an X server, e.g.:

```sh
xvfb-run -a octave --no-gui -q --eval "run_demo(); run_sweep('production'); test_all();"
```

`run_demo` options: `run_demo('sigma_ns','10','seed',7,'figures',false,'emitter_enu',{'50','30','100'})`.

## Inputs, units and validation

* Receiver 1 geodetic position: WGS-84 latitude/longitude in **degrees** (east-positive), ellipsoidal height in
  **metres**; latitude ∈ [−90, 90], longitude ∈ [−180, 360] (wrapped to (−180, 180]), height ∈ [−12 km, 1e8 m].
  The automated demo uses the clearly labelled `SYNTHETIC_DEMO_ORIGIN` (40.0°, 20.0°, 50.0 m) — not a surveyed site.
* Emitter `[E, N, U]` metres; Enter = `[50, 30, 100]`. A `U` different from the configured known Up is rejected
  (`Config:upConflict`) unless `allow_input_z_as_known_up = true` (interactive: confirm with `y`).
* `sigma_ns` ∈ [0, 30] ns, default 30 — it bounds the **standard deviation**, not individual samples.
* All non-integer decimals (coordinates such as `106.066`, `sigma_ns`, geodetic inputs, the pulse interval) are
  given as **strings** and parsed directly to double-double (`dd_parse`). Central configuration:
  `src/config/default_config.m`.

## Model summary

* `TOA_i,p = t_emit,p + d_i/c`, `TDOA_i1 = TOA_i − TOA_1 = (d_i − d_1)/c` (i = 2, 3), c = 299 792 458 m/s,
  30 pulses, PRI = 1 ms (configurable; cancels in TDOA).
* Required noise model: `obs TDOA = true TDOA + ε`, ε iid N(0, σ²), R_t = σ² I₂ (TOAs are not perturbed).
  Optional, separately labelled `independent_toa` model gives R_t = σ²[2 1; 1 2].
* Gaussian samples: Philox4x32-10 counter-based integer PRNG (key = seed, stream; counter = pulse, burst, cell,
  draw) → exact 104-bit uniforms → Box–Muller evaluated in double-double. σ = 0 bypasses the generator.
* Every pulse yields one row with a status: `UNIQUE_VALID`, `AMBIGUOUS`, `INFEASIBLE`, `DEGENERATE_GEOMETRY`,
  `NUMERICAL_FAILURE`. Statistics are computed over `UNIQUE_VALID` pulses only and labelled conditional.

## Outputs (per run folder under `results/`)

| File | Content |
|---|---|
| `pulse_measurements.csv` | emission time, ideal delays/TOAs, ideal TDOAs, actual noise, noisy TDOAs (s and ns), range noise, identity check |
| `pulse_locations.csv` | status, true/selected position in ENU/ECEF/geodetic, errors, residuals, tolerance, conditioning |
| `pulse_candidates.csv` | every candidate (with pulse id, slot, rejection reason) |
| `summary_metrics.csv`, `noise_statistics.csv` | counts, means (ENU→ECEF→geodetic), bias, covariance, RMSE, GDOP, CRLB, noise statistics with sample counts |
| `sweep_*.csv` | 34×72 RMSE matrix, long table (2448 rows), row/column extrema, valid/ambiguous/failure/GDOP/CRLB matrices |
| `*_hilo.csv` | **exact** companions: every DD column as `__hi`, `__lo` (`%.17g`, bit-exact) |
| `run_metadata.json` | configuration, parsed inputs, seed, environment, timings |
| `fig_*.pdf/.png` | vector PDF + 300-dpi PNG figures generated from the same run structures |

## Layout

```
setup_project.m run_demo.m run_interactive.m run_sweep.m test_all.m
src/config  src/dd  src/coords  src/sim  src/solvers  src/stats  src/sweep  src/io  src/plot  src/run
tests/ (test_*.m, oracle_data/*.csv)   tools/oracle/*.py (development-time oracles, Python+mpmath+sympy)
docs/  report/ (report.tex, report.pdf, figures/)   results/ (executed outputs)
```

## Dependencies

Runtime: base MATLAB (targeted ≥ R2016b, **untested**) or GNU Octave ≥ 8 (tested 8.4.0). No toolboxes, no
internet, no MEX. Development-time only (not needed to run): Python 3 with `mpmath` and `sympy`
(`tools/oracle/`), `pdflatex` for the report.

## Integrity

`MANIFEST.sha256` lists every file. Verify with `sha256sum -c MANIFEST.sha256` (Linux) or
`shasum -a 256 -c MANIFEST.sha256` (macOS) from the project root; regenerate with `sh tools/make_manifest.sh`.
`docs/archive_verification.txt` records the clean-extraction run (132/132 checks, demo CSVs byte-identical).

## Documentation

`docs/requirements_traceability.md`, `docs/assumptions_discrepancies.md`, `docs/precision_audit.md`,
`docs/compatibility_and_known_issues.md`, `docs/validation_report.md`, `docs/archive_verification.txt`,
`report/report.pdf` (technical report; source `report/report.tex`, `report/refs.bib`, inputs generated from the run
CSVs by `tools/report/make_report_inputs.py`).

License: MIT (`LICENSE`). All double-double code is original to this project.
