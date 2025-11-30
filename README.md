# ✈️ Flight Fuel Consumption Prediction (PRC-2025)

**Team/Model Codename:** Wise Watermelon 🍉  
**Model Version:** v26

submission for prc2025 data challenge of predicting fuel consumption for specified flight segments
The submission contains two notebooks:
- `prepare_data.ipynb`
- `train_predict.ipynb`

Their processes are briefly summarised below

## Prepare Flight Data Notebook

This notebook (`prepare_data.ipynb`) is designed to **process raw flight trajectory and fuel data** and generate **structured, feature-rich datasets** suitable for modeling, analysis, and fuel prediction.

---

### **Overview**

The notebook performs the following high-level tasks:

1. **Load and merge aircraft metadata**
   - Reads aircraft characteristics from multiple sources, including:
     - `aircraft_df` (main aircraft data)
     - FAA aircraft database (`FAA_AIRCRAFT_DATA`)
     - OpenAP aircraft properties (`openap_prop_df`)
   - Handles synonyms and missing aircraft types.
   - Computes derived features such as wingspan, engine type, and MTOW.

2. **Load flight trajectory data**
   - Reads flight parquet files from `TRAIN_FLIGHTS_DIR`, `RANK_FLIGHTS_DIR`, and `FINAL_FLIGHTS_DIR`.
   - Merges trajectory data with flight metadata (`train_flightlist`, `rank_flightlist`, `final_flightlist`) for each flight.
   - Handles flights split across multiple parquet files.

3. **Resample and clean trajectory data**
   - Converts timestamps to UTC.
   - Ensures all required prediction periods are included, adding synthetic rows for missing intervals before takeoff or after landing.
   - Adjusts vertical rates and altitudes for pre-takeoff and post-landing segments.
   - Resamples trajectories to a fixed interval (60 seconds) and computes a **resample quality score**.

4. **Compute derived flight features**
   - Calculates distances from departure and arrival airports using a vectorized haversine formula.
   - Labels **ATM phases** (`DEP`, `ENR`, `ARR`) and **flight phases** (`GND`, `CL`, `DE`, `LVL`, `CR`, `NA`).
   - Estimates **fuel consumption** using the ACP/Acropole model (`FuelEstimator`), including:
     - Instantaneous fuel
     - Fuel flow
     - Phase-specific fuel (CL, CR, LVL, DE, GND)
   - Computes **thrust** and **drag** per timestamp.
   - Handles missing values intelligently by filling with minimum/maximum thresholds or phase-specific medians.

5. **Segment-level aggregation**
   - Breaks each flight into **fuel segments** defined by `train_fuel`, `rank_fuel`, or `final_fuel`.
   - Computes **segment-level statistics**:
     - Sum, mean, min, max, start/end values for key variables (altitude, fuel, fuel flow, CAS, drag, thrust, etc.)
     - Phase durations (both ATM phases and flight phases)
     - Climb height and total fuel per segment
   - Flags missing segments where no trajectory data is available.

6. **Export prepared datasets**
   - Prepares three datasets:
     - `prep_train_acropole.csv`
     - `prep_rank_acropole.csv`
     - `prep_final_acropole.csv`
   - Each dataset contains **flight metadata, trajectory statistics, fuel estimates, and phase-specific features** to be used by `train_predict.ipynb` for modeling and prediction.

---

### **Key Outputs**

| Variable Type                  | Description |
|--------------------------------|------------|
| `flight_id`                     | Unique flight identifier |
| `seg_start`, `seg_end`          | Start and end of each segment |
| `fuel_kg`                       | Target fuel value for segment |
| `flight_fuel`                   | Total estimated flight fuel |
| `phase` / `atm_phase`           | Traffic library- assigned and ATM (Air Traffic Management) heuristic assigned phases per timestamp |
| `altitude`, `groundspeed`, `TAS` | Trajectory variables |
| `fuel_flow`, `thrust`, `drag`  | Estimated physical parameters per timestamp |
| `*_dur`, `*_ct`        | Segment-level aggregated statistics such as mean, sum, max|

---

### **Dependencies**

The notebook is run in a conda environment defined by `data_prep_environment.yml`

---

### **Usage Notes**

- Ensure all flight parquet files are available in the corresponding directories (`TRAIN_FLIGHTS_DIR`, `RANK_FLIGHTS_DIR`, `FINAL_FLIGHTS_DIR`).
- Aircraft metadata files must be pre-downloaded. It is accessible in the `external_data` directory of this repo
  - FAA Aircraft Database (`FAA_AIRCRAFT_DATA`)
  - World Aiports excel file
    
- The notebook can take a significant amount of time for large datasets; consider **parallel processing** for speedup.

---

### **References**

- FAA Aircraft Characteristic Database: [https://www.faa.gov/airports/engineering/aircraft_char_database/](https://www.faa.gov/airports/engineering/aircraft_char_database/)
- ACP / Acropole fuel estimation library
- OpenAP aircraft performance database

Here is a comprehensive `README.md` file tailored for your GitHub repository. It encapsulates the entire workflow, the specific methodology you used (Stratified Group K-Fold, CatBoost), and the file structure.


## TRAIN_PREDICT NOTEBOOK

### 📖 Overview

This repository contains the solution pipeline for predicting aircraft fuel consumption based on flight telemetry, weather, and aircraft metadata. The solution utilizes a **CatBoost Ensemble** approach, leveraging robust feature engineering and a rigorous **Stratified Group K-Fold** validation strategy to ensure generalization across different aircraft types and flight profiles.

The pipeline processes training data, generates validation rankings, and produces final submissions in Parquet format.

## 🛠️ Key Methodology

### 1. Additional Feature Engineering
We transform raw flight logs into actionable features:
*   **Geospatial Analysis:** Implementation of the **Haversine formula** to calculate great-circle distances between waypoints and total flight distance.
*   **Temporal Features:** Computation of segment durations (in minutes) derived from UTC timestamps.
*   **Physics & Aerodynamics:** Aggregation of key metrics including `sum_drag`, `sum_thrust`, `sum_fuel_flow`, and `total_climb_height`.

### 2. Validation Strategy (Critical)
To prevent data leakage and ensure the model handles new flights correctly, we use **Stratified Group K-Fold Cross-Validation (k=5)**:
*   **Grouping:** Data is grouped by `flight_id`. This ensures that all segments of a single flight remain together in either the training or validation set, preventing the model from "memorizing" specific flight curves.
*   **Stratification:** We stratify based on a composite key of `missing_segment` status and `aircraft_type`. This ensures that difficult cases (rare planes or incomplete logs) are evenly distributed across folds.

### 3. Model & Inference
*   **Algorithm:** CatBoostRegressor (Gradient Boosting on Decision Trees).
*   **Optimization:** Hyperparameters were tuned using **Optuna** to minimize RMSE.
*   **Ensembling:** The final prediction is an average of the 5 models trained during cross-validation.
*   **Physics Constraint:** Predictions are clipped to ensure no fuel consumption value is lower than the minimum observed in the training set (non-negative constraint).

## 📂 Project Structure

```text
├── 📂 prc-2025-datasets/        # External Data Directory
│   ├── flights_train/           # Raw flight sensor data
│   ├── flights_rank/            # Validation flight data
│   ├── flights_final/           # Final test flight data
│   ├── flightlist_*.parquet     # Metadata lists
│   ├── fuel_*.parquet           # Target labels and submission templates
│   └── apt.parquet              # Airport database
│
├── 📓 model_training.ipynb      # Main pipeline notebook
├── 📄 prep_train_acropole_test.csv  # Preprocessed training cache
├── 📄 prep_rank_acropole_test.csv   # Preprocessed rank cache
├── 📄 prep_final_acropole_test.csv  # Preprocessed final cache
│
└── 📤 Submissions
    ├── wise-watermelon_v26.parquet    # Output for Ranking Phase
    └── wise-watermelon_final.parquet  # Output for Final Phase
```

## ⚙️ Prerequisites

The solution requires **Python 3.11.13** and the following libraries:

```python
catboost==1.2.8
pandas==2.2.3
numpy==1.26.4
scikit-learn==1.2.2
pathlib
```

## 🚀 Pipeline Workflow

The notebook follows a linear execution path:

1.  **Configuration:** Sets file paths for Train, Rank, and Final datasets.
2.  **Preprocessing:**
    *   Loads pre-computed CSVs exported from data preparation notebook
    *   Calculates distance and duration features using `extra_features()`.
    *   Fits `LabelEncoder` on the training set for categorical variables (e.g., `aircraft_type`, `engine_model`).
3.  **Feature Selection:** Filters the dataset to a curated list of ~40 features (`sel_ind`) identified through iterative importance analysis.
4.  **Training:**
    *   Splits data using `StratifiedGroupKFold`.
    *   Trains 5 independent CatBoost models.
    *   **CV Performance:** ~213.90 RMSE.
5.  **Inference (Rank & Final):**
    *   Applies the *same* encoders and feature engineering to the test sets.
    *   Predicts using all 5 models and averages the results.
    *   Clips outliers based on physical bounds.
6.  **Export:** Saves predictions to `wise-watermelon_v26.parquet` and `wise-watermelon_final.parquet`.

## 📊 Performance

| Metric | Score (5-Fold CV) |
| :--- | :--- |
| **RMSE** | **213.89** |
| **Train RMSE** | 135.51 |

## 📝 Usage

1.  Ensure the dataset paths in the "Configuration" cell match your local directory structure.
2.  Run the notebook cells sequentially.
3.  The notebook will train the models and automatically generate the submission files in the root directory.

---
*Created for the PRC-2025 Challenge.*
