# MetaL-MCF

**MetaL-MCF: An Interpretable and Scalable Meta-Learning Framework for Modeling Global Mangrove Carbon Flux**

This repository contains the Python code associated with the MetaL-MCF manuscript. The workflow combines predictor preparation, machine-learning training and validation, chunk-based global upscaling, NetCDF export, annual aggregation, trend analysis, and publication figures.

## Framework overview

- **Base learners:** Random Forest (RF), Extra Trees (ETR), XGBoost (XGB), LightGBM (LGBM), and CatBoost (CAT).
- **Meta-learner:** RidgeCV in a stacking regressor.
- **Optimization:** multi-objective Optuna optimization for R², RMSE, and overfitting.
- **Validation and diagnostics:** train/test evaluation, cross-validation, residual plots, Q-Q plots, and model-comparison figures.
- **Interpretability:** SHAP, partial dependence, and permutation importance.
- **Upscaling:** chunked prediction from yearly predictor files, followed by Parquet and NetCDF export.
- **Trend analysis:** Hamed–Rao modified Mann–Kendall tests and Theil–Sen slopes.

## Repository structure

```text
MetaL-MCF/
├── CITATION.cff
├── README.md
├── requirements.txt
├── scripts/
│   ├── config.py
│   ├── 01_data_extraction_and_preparation.py
│   ├── 02_train_metal_mcf.py
│   ├── 03_upscale.py
│   ├── 04_predictions_to_netcdf.py
│   ├── 05_annual_sum.py
│   ├── 06_trend_analysis.py
│   ├── 07_publication_analysis.py
│   ├── 08_model_comparison.py
│   └── 09_carbon_flux_trend_maps.py
├── data/
│   ├── coordinates/coor_world.csv
│   ├── modeling/                 # place modeling supplementary data here
│   └── upscaling_data/           # place yearly upscaling folders here
└── results/                      # generated models, tables, grids, and figures
```

Large modeling and upscaling datasets are not included in the code repository. Distribute them with the manuscript's supplementary files or through a DOI-backed data repository, and add the access link to this README before publication.

## 1. Installation

Python **3.11** is recommended. Create and activate a virtual environment, then install the dependencies.

### Windows PowerShell

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### macOS or Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Cartopy may download Natural Earth map data when figures are first created. An internet connection may therefore be needed for the first plotting run.

## 2. Place the supplementary data

For modeling and upscaling with the prepared supplementary files, use this layout:

```text
data/
├── coordinates/
│   └── coor_world.csv
├── modeling/
│   └── All_Reco.csv
└── upscaling_data/
    ├── 2000/
    │   ├── DTR_2000.csv
    │   ├── GDD_2000.csv
    │   ├── VPD_2000.csv
    │   ├── TIavg_2000.csv
    │   ├── TImax_2000.csv
    │   ├── TImin_2000.csv
    │   ├── SI_2000.csv
    │   ├── E0_2000.csv
    │   ├── PI_2000.csv
    │   ├── SSH_2000.csv
    │   ├── SST_2000.csv
    │   ├── DO_2000.csv
    │   └── chla_2000.csv
    ├── 2001/
    └── .../
```

The coordinate file supplied with the code is already included. The preprocessing script uses it only when nearest-grid extraction is required.

### Modeling CSV requirements

`data/modeling/All_Reco.csv` must contain these columns:

```text
DTR, GDD, VPD, TIavg, TImax, TImin, SI, E0, PI, SSH, SST, DO, Chla, Reco
```

Additional columns are permitted but are not used by the training script. Rows with missing values in any required feature or target column are removed before training.

| Column | Meaning in the workflow |
|---|---|
| DTR | Diurnal temperature range |
| GDD | Growing degree days |
| VPD | Vapor pressure deficit |
| TIavg, TImax, TImin | Temperature-index predictors |
| SI | Salinity index |
| E0 | Canopy quantum-yield predictor |
| PI | Light-limited photosynthesis predictor |
| SSH | Sea surface height |
| SST | Sea surface temperature |
| DO | Dissolved oxygen |
| Chla | Chlorophyll-a |
| Reco | Default target variable; change consistently for another flux |

### Upscaling CSV requirements

Each yearly folder must contain all 13 predictor files listed above. Within each CSV:

- The coordinate columns must be either `X, Y` or `lon, lat`.
- Every remaining column represents one date, normally named `YYYYMMDD`.
- Predictor files must use aligned coordinates and date columns.
- Missing predictor records are excluded from prediction.
- Filename case matters on Linux: chlorophyll-a is read as `chla_YYYY.csv`.

## 3. Configure paths and the carbon-flux target

**Only `scripts/config.py` needs path configuration under normal use.** By default, input is read from the repository's `data/` directory and output is written to `results/`. Absolute paths from the author's computer have been removed.

To use folders elsewhere, edit these settings in `scripts/config.py`:

```python
DATA_DIR = Path(r"D:\your_folder\MetaL-MCF\data")       # Windows example
RESULTS_DIR = Path(r"D:\your_folder\MetaL-MCF\results") # Windows example
TARGET = "Reco"
```

On macOS/Linux, use paths such as `Path("/home/user/MetaL-MCF/data")`.

Environment variables can override the configured locations without editing code:

- `METAL_MCF_DATA_DIR`
- `METAL_MCF_RESULTS_DIR`
- `METAL_MCF_TARGET`
- `METAL_MCF_N_TRIALS`

### Important target consistency

The uploaded training example was originally configured for **Reco**, so `TARGET = "Reco"` is the default. To model GPP or NEE, change **all inputs consistently**:

1. Set `TARGET = "GPP"` or `TARGET = "NEE"` in `scripts/config.py`.
2. Provide `data/modeling/All_GPP.csv` or `data/modeling/All_NEE.csv`.
3. Ensure the CSV contains a target column with exactly the same name.
4. Train a new stacking model before upscaling.

Do not use a Reco-trained model to generate GPP or NEE predictions. Output filenames and figure labels follow the configured target.

## 4. Run the workflow

Run commands from the repository root. The prepared supplementary data allow you to start at step 2; step 1 is optional raw-data preprocessing.

```bash
# Optional: raw-data extraction and predictor preparation
python scripts/01_data_extraction_and_preparation.py

# Train, validate, interpret, and save the stacking model
python scripts/02_train_metal_mcf.py

# Generate global predictions from yearly predictor files
python scripts/03_upscale.py

# Convert Parquet predictions to NetCDF
python scripts/04_predictions_to_netcdf.py

# Aggregate time-dependent grids to annual sums
python scripts/05_annual_sum.py

# Calculate modified Mann–Kendall and Theil–Sen trend products
python scripts/06_trend_analysis.py

# Produce publication analyses and figures
python scripts/07_publication_analysis.py
python scripts/08_model_comparison.py
python scripts/09_carbon_flux_trend_maps.py
```

| Script | Purpose | Main prerequisite |
|---|---|---|
| `01_data_extraction_and_preparation.py` | Raw NetCDF/CSV extraction, predictor calculations, annual driver summaries, and driver figures | Raw source datasets; review section-specific filenames |
| `02_train_metal_mcf.py` | Optimization, training, validation, interpretability, and model export | `data/modeling/All_<TARGET>.csv` |
| `03_upscale.py` | Chunked global prediction | Trained `STACK_Model.pkl` and yearly predictor folders |
| `04_predictions_to_netcdf.py` | Parquet-to-NetCDF conversion | `predictions_<TARGET>_YYYY.parquet` |
| `05_annual_sum.py` | Annual sums | `<TARGET>_YYYY.nc` |
| `06_trend_analysis.py` | Pixel-wise trend statistics | At least three annual NetCDF files |
| `07_publication_analysis.py` | Time-series, spatial, regional, and statistical figures | Annual NetCDF files |
| `08_model_comparison.py` | Observed-versus-predicted model panels | Prediction CSV files from step 2 |
| `09_carbon_flux_trend_maps.py` | Six-panel trend map with zonal/meridional profiles | Annual files and outputs from step 6 |

### Preprocessing note

`01_data_extraction_and_preparation.py` preserves the author's sequential preprocessing and exploratory-analysis blocks. It includes year-specific examples (including SST 2007 and LAI 2000/2020) and several source-file naming conventions. Review and adapt the relevant section before running it; it is not required when prepared modeling and upscaling supplementary files are supplied.

### Computational settings

The original analysis uses **100 Optuna trials per base learner** and parallel model/cross-validation execution. For a quick functionality test only, set `METAL_MCF_N_TRIALS` to a small number or change `N_TRIALS` in `scripts/config.py`. Do not use a reduced-trial run as the manuscript result.

Upscaling memory use can be adjusted with `CHUNK_SIZE` and `PREDICT_BATCH_SIZE` in `scripts/config.py`.

The trend and publication scripts currently use the manuscript period **2000–2020**. Adjust their year filters and labels if your input period differs. Annual aggregation sums the time dimension; confirm that this operation matches the temporal units of your target before interpreting annual totals.

## 5. Outputs

With the default Reco configuration, outputs are written to:

```text
results/
├── Advanced_ML_Framework_All_Reco/
│   ├── STACK_Model.pkl
│   ├── All_Model_Results.csv
│   ├── Final_Optimized_Parameters.csv
│   ├── *_Predictions.csv
│   ├── SHAP_*.*
│   └── other diagnostics and figures
└── Reco_Results/
    ├── predictions_Reco_YYYY.parquet
    ├── Reco_YYYY.nc
    ├── Reco_YYYY_annual.nc
    ├── Reco_Sen_Slope.nc
    ├── Reco_Sen_Slope_Significant.nc
    ├── Reco_MK_Z.nc
    ├── Reco_MK_pvalue.nc
    ├── Reco_MK_Significance.nc
    ├── Publication_Figures/
    └── Reco_Trend_Analysis_Nature_Profiles.*
```

For GPP, the target component of these names changes automatically.

## Reproducibility and responsible use

- The random seed is fixed at 42 in the training script.
- Model parameters, environment information, metrics, and predictions are saved by the training step.
- Exact package versions should be recorded in the environment used for the final manuscript run, for example with `pip freeze`.
- Compilation and reduced synthetic training/upscaling/export tests can check the software handoff, but they do not replace running and validating the full scientific workflow with the final supplementary data.
- Verify source-data licenses, predictor units, date alignment, missing-data conventions, and target units before publication.

## Citation

If you use this code, cite the associated manuscript:

> Al Safadi, K. *MetaL-MCF: An Interpretable and Scalable Meta-Learning Framework for Modeling Global Mangrove Carbon Flux*. Manuscript.

Add the final journal citation, manuscript DOI, and supplementary-data DOI here when available. `CITATION.cff` provides software citation metadata for GitHub.

## License

No license was specified with the source code, so no license file is included. Add an appropriate `LICENSE` file before public release if you intend to permit reuse, modification, or redistribution.
