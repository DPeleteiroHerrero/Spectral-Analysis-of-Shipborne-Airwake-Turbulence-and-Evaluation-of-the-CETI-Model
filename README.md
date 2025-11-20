# Spectral-Analysis-of-Shipborne-Airwake-Turbulence-and-Evaluation-of-the-CETI-Model
# Airwake PSD Analysis & Model Fitting

This repository contains two main scripts:

- `run_airwake_psd_superposition_grid.py` – generates PSD comparison plots for all probe positions.  
- `karman_optimizer_batch.py` – batch-fits turbulence models to all computed PSDs and creates overlay figures + summary tables.

---

## `run_airwake_psd_superposition_grid.py`

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
  - `karman_fit_summary.csv` – full fit diagnostics for every PSD.
  - `tf2_params_simple.csv` – compact table of TF2 parameters only.

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
