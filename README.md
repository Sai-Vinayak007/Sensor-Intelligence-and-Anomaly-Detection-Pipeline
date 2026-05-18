# SentinelFlow
**Sensor Intelligence & Anomaly Detection Pipeline**

A time series analysis pipeline that loads sensor CSV data, cleans it, detects anomalies using multiple methods, and forecasts future anomalies using Prophet + Monte Carlo sampling.

---

## Folder Structure

```
sentinelflow/
├── timeseries_pipeline.py   # main script
├── config.yaml              # all settings live here
├── datasets/
│   └── sensor_data.csv      # your input data
└── outputs/                 # all plots + log saved here (auto-created)
```

---

## Requirements

```bash
pip install pandas numpy matplotlib seaborn scipy scikit-learn pyyaml prophet
```

---

## Input CSV Format

Needs a timestamp column and at least one numeric sensor column. Column names don't matter — all numeric columns are auto-detected.

```
time,SensorA,SensorB,SensorC
2026-05-18 00:00:00,23.4,51.2,0.87
2026-05-18 00:01:00,23.5,51.0,0.88
...
```

---

## Usage

```bash
# Default — reads config.yaml
python timeseries_pipeline.py

# Point to a different CSV without editing config
python timeseries_pipeline.py --csv datasets/mydata.csv

# Full override
python timeseries_pipeline.py --csv data.csv --time-col timestamp --output-dir results
```

---

## Configuration (`config.yaml`)

| Key | Default | What it does |
|---|---|---|
| `csv_path` | `datasets/sensor_data.csv` | Path to input CSV |
| `time_col` | `time` | Name of the timestamp column |
| `sensors` | `null` | Sensor columns to use. `null` = auto-detect all numeric |
| `resample_freq` | `min` | Resampling frequency (`min`, `H`, `D`) |
| `rolling_window` | `30` | Window size for rolling stats |
| `iqr_k` | `3.0` | IQR multiplier for outlier removal (3 = conservative) |
| `zscore_thresh` | `3.0` | Z-score threshold for anomaly flagging |
| `iso_contamination` | `0.05` | Expected anomaly fraction for Isolation Forest |
| `forecast_steps` | `60` | How many steps ahead to forecast |
| `noise_samples` | `200` | Monte Carlo draws per timestep for CI scoring |
| `output_dir` | `outputs` | Where to save plots and logs |

---

## What It Does

### Step 1 — Load & Explore
Reads the CSV, parses timestamps, prints shape, missing value counts, and descriptive stats.

### Step 2 — Raw Plots
Saves raw time series, histograms, boxplots, and a correlation heatmap.

### Step 3 — Clean
Fills missing values (forward-fill → interpolate), removes outliers with IQR k=3, then resamples to a regular frequency.

### Step 4 — Anomaly Detection (4 methods)
- **Z-score** — flags points where |Z| > threshold globally
- **Rolling ±3σ** — flags points outside a rolling mean ± 3 std band
- **Flat Line** — flags sensor freeze/failure zones (rolling std < threshold)
- **Isolation Forest** — ML-based multivariate anomaly detection

### Step 5 — Forecast & Predict *(requires Prophet)*
Trains a Prophet model per sensor, forecasts the next N steps, then scores the forecast using Monte Carlo sampling from the confidence interval — more statistically honest than scoring only the smooth mean.

---

## Output Files

| File | Description |
|---|---|
| `01_missing_value_map.png` | Where data is missing across time |
| `02_raw_time_series.png` | Raw sensor signals |
| `03_raw_distributions.png` | Histogram per sensor |
| `04_raw_boxplots.png` | Boxplots showing spread and outliers |
| `05_correlation_heatmap.png` | Cross-sensor correlations |
| `06_raw_vs_cleaned.png` | Before and after cleaning overlay |
| `07_rolling_mean_std.png` | Rolling mean ± 1 std dev |
| `08_zscore_anomalies.png` | Z-score anomaly locations |
| `09_rolling_anomalies.png` | Rolling σ anomaly locations |
| `10_flatline_detection.png` | Sensor failure / flat-line zones |
| `11_isolation_forest_anomalies.png` | Isolation Forest flagged points |
| `12_isolation_forest_score.png` | Anomaly score over time |
| `13_zscore_heatmap.png` | Z-score heatmap across all sensors |
| `14_combined_anomaly_map.png` | All detection methods side by side |
| `15_prophet_forecast.png` | Prophet forecast with 95% CI |
| `16_predicted_anomalies.png` | Predicted anomaly locations on forecast |
| `17_forecast_anomaly_score.png` | MC anomaly probability & mean score |
| `pipeline.log` | Full run log with counts, guards, and errors |

---

## Sample Results (2026-05-18, 1440 min, 3 sensors)

| Sensor | Missing | Z-score Anomalies | Rolling Anomalies |
|---|---|---|---|
| SensorA | 9.5% | 5 | 4 |
| SensorB | 16.3% | 0 | 1 |
| SensorC | 3.3% | 1 | 3 |

Isolation Forest detected **72 anomalies (5.0%)** across all sensors combined.
Prophet forecast for the next 60 minutes showed **no predicted anomalies**.

---

## Notes

- Re-running is safe — existing output files are skipped automatically. Delete a file to regenerate it.
- All settings are in `config.yaml`. You should rarely need to touch the Python script.
- Prophet is optional. Steps 1–4 run fine without it.
- The log file (`outputs/pipeline.log`) appends on every run — useful for comparing runs over time.