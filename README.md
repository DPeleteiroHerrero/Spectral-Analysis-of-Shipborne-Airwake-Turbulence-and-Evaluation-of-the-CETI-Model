# Spectral-Analysis-of-Shipborne-Airwake-Turbulence-and-Evaluation-of-the-CETI-Model
# Airwake PSD Analysis & Model Fitting

This repository contains two main scripts:

- `run_airwake_psd_superposition_grid.py` – generates PSD comparison plots for all probe positions.  
- `karman_optimizer_batch.py` – batch-fits turbulence models to all computed PSDs and creates overlay figures + summary tables.

---

## `run_airwake_psd.py`

This script processes airwake measurements for multiple ship motion cases and probe positions to compare their turbulence characteristics.

### What it does

- Scans the `AirwakeData/` directory for all case/position folders  
  (e.g. `Case01_x__..._y__..._z__..._RegularSh`).
- For each **probe position**:
  - Loads `measurements.csv` (and optionally `metadata.csv`) for all selected cases.
  - Extracts the velocity components `u_m_s`, `v_m_s`, `w_m_s`.
  - Cleans the signals and estimates the sampling frequency.
  - Computes 1D power spectral densities (PSDs) using a custom Welch-type method.
  - Saves per-case PSD data as CSV files (frequency vs PSD).
- For each position, creates a **2×2 PSD figure**:
  - `u_m_s` (top-left), `v_m_s` (top-right), `w_m_s` (bottom-left),
  - a large legend with all cases (bottom-right).
- Uses consistent, colorblind-safe colors per case and logs processing steps to a log file.

Output figures are stored under:

`superposition of psd 2/<position>/psdgrid_<position>_cases01-07.png`

---

## `karman_optimizer_batch.py`

This script batch-fits turbulence models to all measured PSDs found under `AirwakeData/`.

### What it does

- Recursively scans `AirwakeData/` for all `psd_*.csv` files.
- For each PSD file:
  - Infers **case**, **probe position**, **motion type** and **component** from the path/filename.
  - Loads the measured PSD (`f_Hz`, `Pxx`) and reads the mean flow speed \(U_0\) from a nearby `metadata.csv`.
  - Fits three models in a given frequency band:
    1. **Simplified von Kármán (VK)** model (unweighted log-PSD residuals).  
    2. **Simplified VK with 1/f weighting** (emphasises low frequencies).  
    3. **Second-order transfer-function (TF2)** model, using \(|H|^2\) with \(PSD_\text{in} \equiv 1\) and **no additive noise**.
  - Saves several overlay figures **in the same folder as the PSD**:  
    - `fit_overlay_<stem>.png` (VK)  
    - `fit_overlay_weighted_<stem>.png` (VK, 1/f-weighted)  
    - `fit_overlay_tf2_<stem>.png` (TF2, knee-centred)  
    - `fit_overlay_both_<stem>.png` (measured vs VK vs TF2)  
    - `fit_overlay_all_<stem>.png` (measured vs VK, weighted VK, TF2)
- For **Case06 at `x__14p961_y__7p087_z__7p480`**, creates an extra two-panel figure:
  - `fit_overlay_all_uv_Case06_x__14p961_y__7p087_z__7p480.png`  
  showing measured PSD + VK (unweighted and 1/f-weighted) for **u** and **v** side by side, with a single legend.
- Writes two CSV summaries in the script directory:
  - `karman_fit_summary.csv` – full fit diagnostics for every PSD, including the **von Kármán parameters** for both the simple and 1/f-weighted fits (the **weighted VK fit** is the one actually used/interpreted in the CETI analysis).
  - `tf2_params_simple.csv` – compact table of TF2 parameters only.

---

## `tf2_graphs.py`

This script is a post-processing / visualization tool for `tf2_params_simple.csv`.

### What it does

- Reads `tf2_params_simple.csv` and parses:
  - `case`, `position`, `component` (`u`, `v`, `w`)
  - All numeric variables (e.g. `K`, `wz_rad_s`, `zetz`, `wp_rad_s`, `zetp`, …).
- Extracts the probe coordinates from `position` (e.g. `x__11p811_y__0p000_z__7p480` → x, y, z in metres).
- For **each case** and **each numeric variable**, creates:
  - 2D **maps** (x–y, x–z, y–z) colored by the chosen variable:
    - `maps_<Case>_<var>_all.png` and component-specific versions (`_u`, `_v`, `_w`).
  - **Multi-slice line plots** of the variable vs x / y / z with different slices in the other coordinates:
    - `lines_<Case>_<var>_vs_x_multi.png`, `..._vs_y_multi.png`, `..._vs_z_multi.png`.
  - **3-panel line figures**:
    - `lines_<Case>_<var>_vs_xyz.png` – var vs x, y, z (no slicing, duplicates collapsed).
    - `lines_<Case>_<var>_vs_xyz_sliced.png` – var vs x, y, z using slices at (approximately) constant other coordinates.

In practice, the **sliced 3-panel plots**  
`lines_<Case>_<var>_vs_xyz_sliced.png`  
were the ones I actually used and could interpret most clearly.

---

## `ceti_validate.py`

This script validates the **CETI TF2 model** against the measured PSDs.

### What it does

- Reads the TF2 parameter table `tf2_params_simple.csv` (one row per PSD).
- For each row:
  - Loads the corresponding measured PSD (using the `csv_rel` path).
  - Reconstructs the CETI TF2 model PSD on the same frequency grid using the fitted parameters  
    (`K`, `wz_rad_s`, `zetz`, `wp_rad_s`, `zetp`).
  - Restricts the comparison to a chosen frequency band `[fmin, fmax]`.
  - Computes several validation metrics, including:
    - Log-binned RMS error in log10 space.
    - Percentage of points within ±2 dB and ±3 dB.
    - Band-limited variance ratio (∫S_model df / ∫S_meas df).
    - L2 distance between **normalized cumulative variance curves**.
  - (Optional) Saves an **overlay PSD plot** (measured vs CETI TF2) for each PSD if `--plot` is used:
    - `validate_ceti_overlay_<psd_stem>.png`
  - (Optional) Synthesizes a **time series** consistent with the model PSD for spot checks if `--synthesize` is used:
    - `validate_ceti_synth_<psd_stem>.npy`
- Collects all metrics into a single CSV summary:
  - `ceti_validation_summary.csv` (path configurable via `--out`).

---

## `ceti_flag_mismatches.py`

Small helper script to quickly scan CETI validation results and flag problematic cases.

### What it does

- Looks for `ceti_validation_summary.csv` in the same folder (produced by the CETI validation script).
- Applies built-in **pass/fail thresholds**:
  - `pct_within_2dB` ≥ 70%
  - `rms_log_binned` ≤ 0.12
  - `var_ratio_model_over_meas` ∈ [0.88, 1.12]
- Any row failing **any** of these becomes a **mismatch**.
- Writes a filtered CSV with only the failing rows:
  - `ceti_mismatches.csv`
- Prints a concise summary of mismatches to the console (showing RMS, 2 dB %, variance ratio, and overlay path).
- Tries to automatically **open `ceti_mismatches.csv`** in the default application (Windows/macOS/Linux), so you can inspect the flagged cases immediately.

---

## `ceti_analyze_summary.py`  *(analysis of CETI validation summary)*

This script analyzes the global CETI validation results stored in `ceti_validation_summary.csv`.

### What it does

- Reads `ceti_validation_summary.csv` and defines a boolean `is_mismatch` for each PSD:
  - Uses the `status` column if present (`"mismatch"` → True),  
    otherwise reconstructs pass/fail from typical thresholds (2 dB %, RMS, variance ratio).
- Parses from `csv_rel`:
  - **Case** (Case01, Case02, …)
  - **Component** (`u`, `v`, `w`)
  - Probe coordinates **x, y, z** (from `x__.._y__.._z__..` in the filename).

It then computes and writes:

- Overall mismatch percentage (printed to console).
- **By case** mismatch rates → `ceti_by_case.csv`
- **By component** mismatch rates → `ceti_by_component.csv`
- **By height** \(z\) mismatch rates → `ceti_by_height.csv` (if z is available)
- **By x** position mismatch rates → `ceti_by_x.csv` (if x is available)
- **By y** position mismatch rates → `ceti_by_y.csv` (if y is available)
- **Best and worst examples** by RMS error (with overlay paths) → `ceti_best_worst.csv`
- A plain-text summary for reporting → `ceti_overview.txt`

All key statistics are also printed in a readable form to the terminal.

