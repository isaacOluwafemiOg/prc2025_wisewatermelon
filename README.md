# prc2025_wisewatermelon
submission for prc2025 data challenge of predicting fuel consumption for specified flight segments
The submission contains two notebook:
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
     - `prep_train_acropole_test.csv`
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

## **Usage Notes**

- Ensure all flight parquet files are available in the corresponding directories (`TRAIN_FLIGHTS_DIR`, `RANK_FLIGHTS_DIR`, `FINAL_FLIGHTS_DIR`).
- Aircraft metadata files must be pre-downloaded. It is accessible in the `external_data` directory of this repo
  - FAA Aircraft Database (`FAA_AIRCRAFT_DATA`)
  - World Aiports excel file
    
- The notebook can take a significant amount of time for large datasets; consider **parallel processing** for speedup.

---

## **References**

- FAA Aircraft Characteristic Database: [https://www.faa.gov/airports/engineering/aircraft_char_database/](https://www.faa.gov/airports/engineering/aircraft_char_database/)
- ACP / Acropole fuel estimation library
- OpenAP aircraft performance database

