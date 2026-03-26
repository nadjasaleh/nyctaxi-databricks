# nyctaxi-databricks

End-to-end data engineering pipeline for NYC Taxi & Limousine Commission (TLC) trip data on Databricks, following the **medallion architecture** (Bronze → Silver → Gold). The project ingests raw trip records for four taxi types — Yellow, Green, For-Hire Vehicles (FHV), and High Volume FHV (Uber, Lyft, etc.) — cleans and enriches them, and produces analytics-ready fact and dimension tables.

---

## Architecture Overview

```
ADLS (JSON/Parquet)          Bronze (raw)           Silver (cleaned)           Gold (analytics)
┌──────────────┐      ┌────────────────────┐    ┌───────────────────────┐    ┌─────────────────────┐
│ Landing Zone │ ───▶ │ group3_gp.bronze   │ ──▶│ group3_gp.silver      │ ──▶│ group3_gp.gold      │
│  (ADLS)      │      │  yellow            │    │  yellow_taxi_trips    │    │  dim_date           │
│              │      │  green             │    │  green_taxi_trips     │    │  dim_hour           │
│              │      │  for_hire_vehicles  │    │  fhv_taxi_trips       │    │  dim_zone           │
│              │      │  high_volume_fhv    │    │  high_volume_fhv      │    │  dim_company         │
│              │      │  taxi_zones         │    │  combined_taxi_trips  │    │  Fact tables (6)    │
│              │      │  taxi_zones_lookup  │    └───────────────────────┘    │  tabel_count        │
│              │      └────────────────────┘                                  └─────────────────────┘
└──────────────┘
```

**Catalog:** `group3_gp`  
**Cloud:** Azure (ADLS Gen2 storage account: `group3gpsa`)  
**Key technologies:** PySpark, Delta Lake, Apache Sedona (spatial joins), Databricks SQL

---

## Project Structure

```
nyctaxi-databricks/
├── README.md                          ← This file
├── .gitignore
├── README_Files/                      ← Detailed per-layer documentation
│   ├── README_Pre_Process.md              Pre-process notebook guide
│   ├── README_Bronze_Layer_Notebooks.md   Bronze layer notebook guide
│   ├── README_bronze_level_table_layout.md Bronze table schemas
│   ├── README_Data_Quality_Check.md       Data quality check guide
│   ├── README_Silver_Layer_Notebooks.md   Silver layer notebook guide
│   ├── README_Gold_Layer_Notebooks.md     Gold layer notebook guide
│   └── README_testing_data_Summary.md     Yellow taxi data quality summary
├── 0. Data quality check/             ← Data quality audits
│   ├── Data quality check.ipynb           Master quality check (all datasets)
│   ├── Yellow taxi data check.ipynb       Yellow taxi deep-dive
│   ├── Green taxi data check.ipynb        Green taxi deep-dive
│   ├── For H Vehicles data check.ipynb    FHV deep-dive
│   └── High vol fhv data check.ipynb      High Volume FHV deep-dive
├── 1_pre_process/                     ← Data preparation and ingestion
│   ├── Convert_parquet_to_json_upload_to_ADLS.ipynb
│   └── ingest_zones_shp.ipynb
├── 2_Bronze_Layer_Notebooks/          ← Raw data ingestion
│   ├── Bronze_Ingestion_From_ADLS.ipynb
│   └── 2.5_Create_Dimensions_Zones_Date_Tablecount.ipynb
├── 3_Silver_Layer_Notebooks/          ← Cleaning and enrichment
│   ├── 3 Yellow Taxi Silver Table.ipynb
│   ├── 3 Green Taxi Silver Table.ipynb
│   ├── 3 For Hire Taxi Silver Table.ipynb
│   ├── 3 High volume FHV Taxi Silver Table.ipynb
│   └── 3.5 join silver tables.ipynb
└── 4_Gold_Layer_Notebooks/            ← Dimension and fact tables
    ├── Create_Dim_Tables_Date_Hour_compan.ipynb
    ├── Fact_Table_All.ipynb
    └── Populate_Tablecount.ipynb
```

---

## Pipeline Stages

### 0. Data Quality Check

Systematic quality audits across all four TLC datasets stored in `group3_gp.testing`. All numeric columns are stored as `STRING` in the raw tables, so queries use `TRY_CAST` for safe conversion.

| Notebook | Target Table | Purpose |
| --- | --- | --- |
| Data quality check | All four tables | Master notebook — schemas, expected nulls, bad-data checks across all datasets |
| Yellow taxi data check | `group3_gp.testing.yellow` | Deep-dive audit: bad fares, distances, timestamps, ratecodes, payment types, vendors |
| Green taxi data check | `group3_gp.testing.green` | Deep-dive audit: similar checks, notes `lpep_` prefix and `ehail_fee` nulls |
| For H Vehicles data check | `group3_gp.testing.for_hire_vehicles` | Audit: timestamps, dirty base numbers, shared ride flags (sparse schema — 8 columns) |
| High vol fhv data check | `group3_gp.testing.high_volume_fhv` | Audit: timestamps, distances, fares, company breakdown (Uber/Lyft/Juno/Via) |

**Key findings:**
* Bad fares in Yellow are only 0.08% but spike in 2022–2023 (66% of all bad fares), with Curb Mobility (vendor 2) responsible for 84%
* Era transitions at 2017: lat/lon → zone IDs, column name changes (e.g. `vendor_id` → `vendorid`)
* High Volume FHV fare columns only populated from ~2021 onwards

> Detailed write-up: [`README_Files/README_Data_Quality_Check.md`](README_Files/README_Data_Quality_Check.md)  
> Yellow taxi findings: [`README_Files/README_testing_data_Summary.md`](README_Files/README_testing_data_Summary.md)

---

### 1. Pre-Processing

| Notebook | Purpose |
| --- | --- |
| Convert_parquet_to_json_upload_to_ADLS | Converts parquet files from the ADLS landing zone to JSON and re-uploads them for ingestion. Handles Yellow, Green, FHV, and High Volume FHV files. |
| ingest_zones_shp | Reads the NYC taxi zones shapefile (`taxi_zones.shp`) using GeoPandas, converts geometry to WKT, and writes to `bronze.taxi_zones`. Also creates `bronze.taxi_zones_lookup` as an external table from a CSV on ADLS. |

> Detailed write-up: [`README_Files/README_Pre_Process.md`](README_Files/README_Pre_Process.md)

---

### 2. Bronze Layer

Raw data ingestion from Azure Data Lake Storage into Delta tables with minimal transformation.

| Notebook | Purpose |
| --- | --- |
| Bronze_Ingestion_From_ADLS | Reads JSON files from `/Volumes/group3_gp/adlslanding/looped_data`, categorizes by filename (yellow/green/fhv/fhvhv), unions per category, and writes to `group3_gp.bronze` Delta tables. |
| 2.5_Create_Dimensions_Zones_Date_Tablecount | Creates the `gold.dim_zone` table (joining `bronze.taxi_zones` and `bronze.taxi_zones_lookup`) and initializes the `gold.tabel_count` tracking table. |

**Bronze tables:** `yellow`, `green`, `for_hire_vehicles`, `high_volume_fhv`, `taxi_zones`, `taxi_zones_lookup`, `testingnyc`

> Detailed write-up: [`README_Files/README_Bronze_Layer_Notebooks.md`](README_Files/README_Bronze_Layer_Notebooks.md)  
> Full schema reference: [`README_Files/README_bronze_level_table_layout.md`](README_Files/README_bronze_level_table_layout.md)

---

### 3. Silver Layer

Cleaned, type-cast, and enriched trip data. All notebooks follow a common pattern:
1. Read raw bronze table
2. Consolidate duplicate columns (coalesce old-era / new-era names)
3. Drop redundant columns
4. Cast to proper types (`timestamp`, `double`, `decimal(10,2)`, `integer`)
5. Map legacy string codes to descriptive labels
6. Filter invalid rows (null timestamps, dropoff ≤ pickup, negative fares, missing locations)
7. Write to silver Delta table

| Notebook | Source → Target | Notes |
| --- | --- | --- |
| 3 Yellow Taxi Silver Table | `testing.yellow` → `silver.yellow_taxi_trips` | Spatial join with Apache Sedona to impute zone IDs from pre-2017 lat/lon |
| 3 Green Taxi Silver Table | `testing.green` → `silver.green_taxi_trips` | Same spatial enrichment; drops `ehail_fee` and `trip_type` |
| 3 For Hire Taxi Silver Table | `testing.for_hire_vehicles` → `silver.fhv_taxi_trips` | No spatial join (no lat/lon in source); adds `service_type="fhv"` label |
| 3 High volume FHV Taxi Silver Table | `testing.high_volume_fhv` → `silver.high_volume_fhv` | Drops 9 columns; renames `tips`→`tip_amount`, `tolls`→`tolls_amount`, `trip_miles`→`trip_distance` |
| 3.5 join silver tables | All 4 silver tables → `silver.combined_taxi_trips` | Unions all taxi types into a unified schema, cleans invalid zone IDs, calculates missing trip distances using Sedona centroids, converts miles to km, deduplicates |

**Combined schema** (`silver.combined_taxi_trips`): `taxi_type`, `service_type`, `pickup_datetime`, `dropoff_datetime`, `trip_time`, `pulocationid`, `dolocationid`, `trip_distance`, `extra`, `fare_amount`, `mta_tax`, `tip_amount`, `tolls_amount`, `total_amount`, `payment_types`, `rate_codes`, `vendors`, `base_passenger_fare`, and calculated flags.

> Detailed write-up: [`README_Files/README_Silver_Layer_Notebooks.md`](README_Files/README_Silver_Layer_Notebooks.md)

---

### 4. Gold Layer

Star schema dimension and fact tables for analytics and dashboarding.

#### Dimension Tables

| Notebook | Tables Created | Description |
| --- | --- | --- |
| Create_Dim_Tables_Date_Hour_compan | `gold.dim_date` | Date dimension from 2008-01-01 to today: year, month, day, day_of_week, quarter, week_of_year, is_weekend |
| | `gold.dim_hour` | Hour dimension (0–23) with `period_of_day` (morning/noon/afternoon/evening/night) and `is_rush_traffic` |
| | `gold.dim_company` | Company dimension derived from distinct `service_type` × `taxi_type` combinations |
| 2.5_Create_Dimensions_Zones_Date_Tablecount | `gold.dim_zone` | Zone dimension joining `taxi_zones` polygons with lookup (zone_id, zone_name, borough, service_zone, geometry_wkt) |

#### Fact Tables

All created in the **Fact_Table_All** notebook from `silver.combined_taxi_trips` joined with dimension tables.

| Fact Table | Grain | Key Measures |
| --- | --- | --- |
| `gold.1_Fact_Revenue_And_Fares` | date × company × payment_type | total_trips, total/avg trip_distance, total/avg trip_time, total/avg revenue, total/avg tip |
| `gold.2_Fact_Trip_Demand_Volume` | date × hour × company | total_trips, total/avg trip_distance, total/avg trip_time |
| `gold.4_Fact_Company_and_Market_Share` | date × company | total_trips, total/avg revenue, total/avg tip, total/avg trip_distance, total/avg trip_time |
| `gold.5_Fact_Trip_Efficiency_and_Operations` | company × pickup_zone × dropoff_zone | total_trips, total/avg trip_distance, total/avg trip_time, total/avg amount |
| `gold.3_and_7_Fact_Geography_and_Zones_and_Covid` | date × company × pickup_zone × dropoff_zone | total_trips, total/avg trip_distance, total/avg trip_time, total/avg amount |
| `testing.Fact_Grouped_By_Hour_Date_Company_Zone` | date × hour × company × pickup_zone × dropoff_zone | total_trips, total_trip_distance, total_trip_time |

#### Row Count Tracking

| Notebook | Purpose |
| --- | --- |
| Populate_Tablecount | Inserts row counts at each pipeline stage into `gold.tabel_count` — bronze ingest (step 1), silver cleaning (step 2), combined silver (step 3), final combined (step 4) |

> Detailed write-up: [`README_Files/README_Gold_Layer_Notebooks.md`](README_Files/README_Gold_Layer_Notebooks.md)

---

## Execution Order

Run notebooks in the following order for a full pipeline refresh:

1. `1_pre_process/Convert_parquet_to_json_upload_to_ADLS` — Prepare source files
2. `1_pre_process/ingest_zones_shp` — Ingest zone reference data
3. `2_Bronze_Layer_Notebooks/Bronze_Ingestion_From_ADLS` — Ingest trip data to bronze
4. `2_Bronze_Layer_Notebooks/2.5_Create_Dimensions_Zones_Date_Tablecount` — Create zone dim + tracking table
5. `3_Silver_Layer_Notebooks/3 Yellow Taxi Silver Table` — Clean Yellow
6. `3_Silver_Layer_Notebooks/3 Green Taxi Silver Table` — Clean Green
7. `3_Silver_Layer_Notebooks/3 For Hire Taxi Silver Table` — Clean FHV
8. `3_Silver_Layer_Notebooks/3 High volume FHV Taxi Silver Table` — Clean High Volume FHV
9. `3_Silver_Layer_Notebooks/3.5 join silver tables` — Union all into combined silver
10. `4_Gold_Layer_Notebooks/Create_Dim_Tables_Date_Hour_compan` — Create dimension tables
11. `4_Gold_Layer_Notebooks/Fact_Table_All` — Create all fact tables
12. `4_Gold_Layer_Notebooks/Populate_Tablecount` — Record row counts at each stage

---

## Data Sources

* **NYC TLC Trip Record Data** — Yellow, Green, FHV, and High Volume FHV trip records
* **NYC Taxi Zone Shapefile** — Zone boundaries (EPSG:2263, NY State Plane)
* **Taxi Zone Lookup CSV** — Zone ID to borough/zone name mapping
* **Storage:** Azure Data Lake Storage Gen2 (`group3gpsa.dfs.core.windows.net`)

---

## Dependencies

* **Apache Sedona** — Spatial joins for lat/lon → zone ID imputation (Yellow & Green silver notebooks, combined join)
* **GeoPandas** — Shapefile parsing for zone boundary ingestion
* **PySpark / Databricks Runtime** — Core data processing
* **Delta Lake** — Table storage format throughout all layers

---

## Detailed Documentation

Each pipeline stage has a dedicated README in the `README_Files/` folder:

| README | Covers |
| --- | --- |
| [`README_Pre_Process.md`](README_Files/README_Pre_Process.md) | Pre-processing notebooks (parquet conversion, zone ingestion) |
| [`README_Bronze_Layer_Notebooks.md`](README_Files/README_Bronze_Layer_Notebooks.md) | Bronze ingestion notebooks (ADLS loading, dim_zone creation) |
| [`README_bronze_level_table_layout.md`](README_Files/README_bronze_level_table_layout.md) | Bronze table schemas and column details |
| [`README_Data_Quality_Check.md`](README_Files/README_Data_Quality_Check.md) | Data quality audit notebooks and findings |
| [`README_testing_data_Summary.md`](README_Files/README_testing_data_Summary.md) | Yellow taxi data quality deep-dive with statistics |
| [`README_Silver_Layer_Notebooks.md`](README_Files/README_Silver_Layer_Notebooks.md) | Silver layer cleaning, enrichment, and combined union |
| [`README_Gold_Layer_Notebooks.md`](README_Files/README_Gold_Layer_Notebooks.md) | Gold layer dimension tables, fact tables, and row tracking |
