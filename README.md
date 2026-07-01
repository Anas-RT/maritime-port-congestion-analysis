# Maritime Port Congestion Analysis

This project uses NOAA AIS vessel tracking data to build a cleaned and modelled dataset for analysing vessel movement around the Port Authority of New York and New Jersey (PANYNJ) region.

The main focus is the data pipeline: loading large raw files, checking data quality, filtering the region, cleaning vessel records, creating movement features, and shaping the output into fact and dimension tables.

There is no dashboard for this project yet. At this stage, the repo is about getting the data into a reliable state for future analysis.

Dataset source: [NOAA AIS Data Handler](https://coast.noaa.gov/htdata/CMSP/AISDataHandler/)

## Why I Built This

I wanted a project that was closer to real operational data work than a simple EDA notebook. AIS data is large, messy, and full of edge cases, so it was a good way to practise:

- working with files that are too large to casually load all at once
- checking schema consistency across daily data files
- handling known placeholder and sentinel values
- cleaning vessel metadata without inventing values
- creating movement features from timestamped positions
- building a model that could later support analysis or reporting

## Dataset Scope

- **Source:** NOAA Marine Cadastre AIS daily vessel transmission files
- **Period:** January 2024
- **Region:** PANYNJ / New York and New Jersey port area
- **Grain:** one row per AIS vessel position broadcast
- **Final modelled size:** about 10.85 million vessel-position records

I scoped the project to PANYNJ because the full US dataset was too large for my local machine. The region is still large enough to make the project realistic and operationally interesting.

## Project Workflow

| Phase | Notebook | What it does |
| ----- | -------- | ------------ |
| 1 | `01 - Load_profile.ipynb` | Profiles the raw files and checks the main data quality issues |
| 2 | `02 - Scope Region Dataset.ipynb` | Filters January 2024 AIS data to vessels active in the PANYNJ area |
| 3 | `03 - Cleaning & Curation.ipynb` | Cleans sentinel values, vessel dimensions, text fields, and duplicates |
| 4 | `04 - Behavioural Feature Engineering.ipynb` | Adds movement features such as distance, speed, acceleration, and movement state |
| 5 | `05 - Modelling.ipynb` | Builds the final fact and dimension tables |

## Key Data Checks

Some of the checks handled in the notebooks include:

- MMSI format and vessel metadata consistency
- latitude and longitude ranges
- AIS placeholder values such as `SOG = 102.3`, `COG = 360`, and `Heading = 511`
- zero-value vessel dimensions such as Length, Width, and Draft
- duplicate vessel transmissions
- schema consistency across all 31 daily files
- foreign-key completeness in the final model

## Cleaning and Feature Engineering

The cleaning step keeps the data honest. Known AIS sentinel values are converted to nulls, invalid zero dimensions are treated as missing, and vessel text fields are standardised. I only corrected values where there was a clear reason, such as confirmed Length/Width swaps for 2 vessels.

The feature engineering step reconstructs vessel movement over time by comparing each position to the previous position for the same vessel. This creates fields such as:

| Feature | Meaning |
| ------- | ------- |
| `time_delta_seconds` | Time between vessel transmissions |
| `distance_m` | Haversine distance between consecutive positions |
| `speed_mps` | GPS-derived speed from position changes |
| `acceleration_mps2` | Change in speed over time |
| `movement_state` | Simple movement band such as stationary, slow, normal, or high speed |

## Final Model

The final output is a small star schema that can be used for future analysis.

**Fact table**

| Table | Grain | Rows |
| ----- | ----- | ---- |
| `fact_vessel_events` | One row per vessel position observation | 10,854,660 |

**Dimension tables**

| Table | Grain | Rows |
| ----- | ----- | ---- |
| `dim_vessel` | One row per vessel MMSI | 868 |
| `dim_date` | One row per day | 31 |
| `dim_location` | One row per rounded spatial grid cell | 166,532 |
| `dim_movement_state` | One row per movement state | 6 |

The final joins were validated with `validate="many_to_one"`, and the model finished with zero missing foreign keys.

## Current Status

The pipeline and modelling work is complete.

The dashboard and final analysis layer have not been started yet. The next step would be to use the model to analyse traffic volume, slow-moving vessels, spatial density, and possible congestion patterns.

## Possible Next Steps

- Analyse traffic volume by day, hour, and vessel type
- Look at stationary and slow-moving vessels as congestion signals
- Build spatial density views using the location grid
- Compare movement patterns across vessel categories
- Add a dashboard once the analysis questions are clearer

## Repository Notes

Raw and processed AIS files are not tracked in Git because they are very large. The repo is intended to track the notebooks, project documentation, and code needed to recreate the outputs locally.

## Limitations

- The project only covers January 2024.
- The region is limited to the PANYNJ area.
- There is no dashboard yet.
- The PANYNJ bounding box is an analytical scope, not an official port boundary.
- AIS data can contain gaps, invalid metadata, spoofed positions, and equipment artefacts.
- `speed_mps` is calculated from position changes, so it is not the same as the vessel-reported SOG field.

## Tools Used

- Python
- pandas
- numpy
- Jupyter Notebook
- Parquet / pyarrow
- Dimensional modelling

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

- NOAA Marine Cadastre AIS Data Dictionary  
  https://coast.noaa.gov/data/marinecadastre/ais-data-dictionary.pdf
- ITU-R M.1371 - Technical characteristics for AIS
- U.S. Coast Guard Navigation Center - AIS Guidance  
  https://www.navcen.uscg.gov/ais
