# Global Maritime Traffic & Port Congestion Pipeline

**Portfolio project** | Data pipeline, data quality, feature engineering, and dimensional modelling case study

Dataset source: [NOAA AIS Data Handler](https://coast.noaa.gov/htdata/CMSP/AISDataHandler/)

## Project Overview

This project processes NOAA AIS (Automatic Identification System) vessel tracking data into a cleaned, analytics-ready dimensional model for maritime traffic and congestion analysis around the Port Authority of New York and New Jersey (PANYNJ) region.

The focus of this repository is the **data pipeline and modelling layer**, not dashboard design. No Power BI dashboard has been started for this project yet.

The aim is to show a realistic operational analytics workflow: raw file profiling, schema validation, regional filtering, cleaning rules, behavioural feature engineering, and final model validation.

## Engineering and Analytics Focus

This project is mainly a data engineering / analytics engineering portfolio piece, with analyst-facing outputs prepared for future exploration.

- **Data engineering focus:** chunked processing, schema checks, safe output writing, cleaning rules, reproducible transformations, and parquet outputs.
- **Analytics engineering focus:** movement-state features, dimensional modelling, fact/dimension table design, and validated joins.
- **Future analyst focus:** traffic volume, vessel behaviour, spatial density, dwell-time, and congestion analysis.

## Dataset Overview

- **Source:** NOAA Marine Cadastre AIS daily vessel transmission files
- **Data grain:** one AIS transmission per vessel position broadcast at a specific timestamp
- **Scope:** January 2024, PANYNJ region
- **Reason for regional scope:** full US AIS data was too large for in-memory processing on a 16GB RAM machine
- **Final modelled dataset size:** ~10.85 million vessel-position events

## Pipeline Phases

### Phase 1: Loading and Profiling (`01 - Load_profile.ipynb`)

Validated the raw AIS files before cleaning:

- MMSI format and metadata consistency checks
- Coordinate range validation for latitude and longitude
- Movement signal validation for SOG, COG, and Heading
- Sentinel value identification (`SOG = 102.3`, `COG = 360`, `Heading = 511`)
- Zero-value dimension investigation for Length, Width, and Draft
- Duplicate detection
- Cross-day stability checks across sampled daily files

References used: ITU-R M.1371 AIS technical specification and NOAA Marine Cadastre AIS data dictionary.

### Phase 2: Regional Filtering (`02 - Scope Region Dataset.ipynb`)

Filtered the January 2024 AIS dataset to a PANYNJ-centred operating area:

- Checked schema consistency across 31 daily files
- Used chunked processing to avoid loading the full source into memory
- Applied a two-pass regional filter:
  - identify vessels with at least one transmission inside the PANYNJ bounding box
  - retain those vessels' full movement history for the month
- Handled `MMSI = 0` separately
- Used a temporary write pattern for safer file output

PANYNJ bounding box: `LAT 40.45-40.75`, `LON -74.25 to -73.85`

### Phase 3: Cleaning and Curation (`03 - Cleaning & Curation.ipynb`)

Produced a cleaned analytical dataset without imputing or inventing values:

- Converted AIS sentinel values to nulls
- Converted invalid zero vessel dimensions to nulls
- Standardised text fields such as VesselName, CallSign, and IMO
- Resolved duplicates using MMSI and timestamp logic
- Ran vessel-dimension validation checks, including ratio checks
- Corrected confirmed Length/Width swaps for 2 vessels

### Phase 4: Behavioural Feature Engineering (`04 - Behavioural Feature Engineering.ipynb`)

Derived vessel movement features from position histories:

| Feature | Description |
| ------- | ----------- |
| `Timestamp` | Reconstructed timestamp from source date and time fields |
| `prev_timestamp`, `prev_LAT`, `prev_LON` | Previous vessel observation using grouped shifts |
| `time_delta_seconds` | Time between consecutive transmissions |
| `distance_m` | Haversine distance between vessel positions |
| `speed_mps` | GPS-derived speed from position changes |
| `prev_speed_mps` | Previous calculated speed |
| `acceleration_mps2` | Change in speed divided by time delta |
| `movement_state` | Behaviour category based on speed band |

Movement states: `STATIONARY`, `VERY_SLOW`, `SLOW`, `NORMAL`, `HIGH_SPEED`, `UNKNOWN`

### Phase 5: Dimensional Modelling (`05 - Modelling.ipynb`)

Built an analytics-ready star schema from the behavioural dataset.

**Fact table:**

| Table | Grain | Rows |
| ----- | ----- | ---- |
| `fact_vessel_events` | One row per vessel position observation | 10,854,660 |

**Dimension tables:**

| Table | Grain | Rows |
| ----- | ----- | ---- |
| `dim_vessel` | One row per unique vessel MMSI | 868 |
| `dim_date` | One row per calendar day | 31 |
| `dim_location` | One row per rounded spatial grid cell | 166,532 |
| `dim_movement_state` | One row per movement state | 6 |

Exact latitude and longitude are retained in the fact table for calculation. Rounded grid cells are used in `dim_location` for spatial aggregation.

All foreign-key joins were validated with `validate="many_to_one"`. Final join checks returned zero missing foreign keys.

## Current Status

**Pipeline and modelling phase complete.**

No dashboard layer has been started yet. The repository currently demonstrates the data preparation and modelling work that would support future analysis or dashboarding.

## Future Analysis Ideas

Potential next steps for the analyst-facing layer:

- Vessel traffic volume by day, hour, and vessel type
- Movement-state patterns as congestion indicators
- Spatial density heatmaps by rounded grid cell
- Dwell-time and low-speed behaviour near port areas
- Speed and acceleration anomaly checks
- Vessel category comparison across cargo, tanker, passenger, tug, and other classes

## Repository Notes

Raw and processed AIS data files are intentionally excluded from Git because they are very large. This repository should track the notebooks, README, and project metadata only.

The generated CSV and parquet outputs can be recreated by running the notebooks in order once the source AIS files are available locally.

## Known Limitations

- **January 2024 only:** no seasonal comparison is available yet.
- **PANYNJ scope only:** analysis is limited to the selected New York / New Jersey port region.
- **No dashboard started:** this repo currently stops at the modelled data layer.
- **GPS-derived speed vs AIS SOG:** `speed_mps` is calculated from position changes, not directly from the vessel's own speed log.
- **Irregular transmission intervals:** AIS broadcast frequency varies by vessel class, speed, and receiver coverage.
- **Approximate port boundary:** the PANYNJ bounding box is an analytical scope, not a formal berth or terminal polygon.
- **Movement-state thresholds are approximate:** speed bands are practical analysis rules, not externally calibrated labels.
- **AIS data quality:** AIS data can include gaps, spoofed positions, invalid metadata, and equipment artefacts.

## Technical Stack

- Python
- pandas
- numpy
- Jupyter Notebook
- Parquet / pyarrow
- Dimensional modelling
- Data quality validation

## Repository Structure

```text
Maritime Logistics Intelligence/
|
|-- Data/
|   |-- Raw/           # Local raw AIS files, not tracked
|   |-- Processed/     # Generated outputs, not tracked
|
|-- Notebooks/
|   |-- 01 - Load_profile.ipynb
|   |-- 02 - Scope Region Dataset.ipynb
|   |-- 03 - Cleaning & Curation.ipynb
|   |-- 04 - Behavioural Feature Engineering.ipynb
|   |-- 05 - Modelling.ipynb
|
|-- README.md
|-- .gitignore
```

## References

- NOAA Marine Cadastre - AIS Data Dictionary  
  https://coast.noaa.gov/data/marinecadastre/ais-data-dictionary.pdf
- ITU-R M.1371 - Technical characteristics for AIS
- U.S. Coast Guard Navigation Center - AIS Guidance  
  https://www.navcen.uscg.gov/ais
