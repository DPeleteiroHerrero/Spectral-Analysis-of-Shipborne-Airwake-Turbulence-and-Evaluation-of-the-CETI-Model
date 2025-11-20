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

```text
superposition of psd 2/<position>/psdgrid_<position>_cases01-07.png
