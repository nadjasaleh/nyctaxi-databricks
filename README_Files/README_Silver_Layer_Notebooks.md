# Silver Layer Notebooks — Guide

This folder contains five notebooks that transform raw bronze-layer NYC TLC trip data into cleaned, typed, and enriched silver-layer Delta tables, plus a union notebook that combines all four into a single table. Each cleaning notebook reads from `group3_gp.testing` (bronze) and writes to `group3_gp.silver`.

---

## Notebooks Overview

| # | Notebook | Source Table | Target Table | Status |
| --- | --- | --- | --- | --- |
| 1 | Yellow Taxi Silver Table | `group3_gp.testing.yellow` | `group3_gp.silver.yellow_taxi_trips` | Complete |
| 2 | Green Taxi Silver Table | `group3_gp.testing.green` | `group3_gp.silver.green_taxi_trips` | Complete |
| 3 | For Hire Taxi Silver Table | `group3_gp.testing.for_hire_vehicles` | `group3_gp.silver.fhv_taxi_trips` | Complete |
| 4 | High volume FHV Taxi Silver Table | `group3_gp.testing.high_volume_fhv` | `group3_gp.silver.high_volume_fhv` | Complete |
| 5 | 3.5 join silver tables | All 4 silver tables | `group3_gp.silver.combined_taxi_trips` | Complete |

---

## Common Pipeline Pattern

All cleaning notebooks (1–4) follow the same general structure (Yellow and Green share the most code):

| Step | Description |
| --- | --- |
| **Step 1** | Read the raw bronze table |
| **Step 2** | Consolidate duplicate columns (coalesce old-era and new-era column names) |
| **Step 3** | Drop redundant / unuseful columns |
| **Step 4** | Cast all columns to proper types (`timestamp`, `double`, `decimal(10,2)`, `integer`) using `try_cast` for safety |
| **Step 4b** | Map legacy string codes to descriptive labels (payment types, vendor names, ratecodes) |
| **Step 5** | Filter out invalid rows (null timestamps, dropoff <= pickup, negative fares, missing locations) |
| **Step 6** | Write cleaned DataFrame to the silver Delta table (`mode("overwrite")`) |

### Spatial Enrichment (Yellow & Green only)

Pre-2017 rows have lat/lon coordinates instead of zone IDs. These notebooks use **Apache Sedona** to impute the missing `pulocationid` / `dolocationid`:

1. Load taxi zone polygons from `group3_gp.gold.dim_zone` (SRID 2263, NY State Plane)
2. Build point geometries from lat/lon, transform to SRID 2263
3. Spatial join using `ST_Contains(zone_geom, point_geom)` to find the enclosing zone
4. Merge imputed zone IDs back into the main DataFrame
5. Drop rows where location could still not be resolved

---

## 1. Yellow Taxi Silver Table

**Path:** `3_Silver_Layer_Notebooks/3 Yellow Taxi Silver Table`

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Install dependencies | `%pip install apache-sedona` |
| 2 | — | Restart Python to pick up new packages |
| 3 | Imports | SedonaContext + PySpark functions |
| 4 | — | Initialize Sedona spatial context |
| 5 | — | Load taxi zone polygons (`group3_gp.gold.dim_zone`) |
| 6 | Step 1 — Read table | Read `group3_gp.testing.yellow`, print initial row count |
| 7 | Step 2 — Consolidate duplicate columns | Coalesce `tpep_pickup_datetime` / `pickup_datetime`, `tpep_dropoff_datetime` / `dropoff_datetime`, `vendor_id` / `vendorid`, `rate_code` / `ratecodeid`; default `passenger_count` to 0 |
| 8 | Step 3 — Drop redundant columns | Remove originals after coalesce, plus `store_and_fwd_flag`, `imp_surcharge` |
| 9 | — | Rename cleaned columns to final names |
| 10 | Step 4 — Cast columns | Cast to `timestamp`, `decimal(10,2)`, `double`, `integer` using `try_cast` |
| 11 | Step 4b — Map legacy codes + cast to integer | Map payment types (`CRD`→Credit card, `CSH`→Cash, etc.), vendor IDs, ratecodes to descriptive strings; cast raw codes to integer |
| 12 | Step 5 — Filter bad rows | Remove rows with null timestamps, dropoff <= pickup, negative fares, missing locations |
| 13 | — | Preview post-2021 data |
| 14 | — | Add `row_id` via `monotonically_increasing_id()` for spatial join tracking |
| 15 | — | Build point geometries for rows missing pickup/dropoff zone IDs |
| 16 | — | Spatial join: `ST_Contains` to match lat/lon to taxi zone polygons |
| 17 | — | Merge imputed zone IDs back, drop helper columns, filter remaining nulls |
| 18 | Step 6 — Save silver table | Write to `group3_gp.silver.yellow_taxi_trips` (Delta, overwrite) |

### Column consolidation map

| Final column | Old-era source | New-era source |
| --- | --- | --- |
| `pickup_datetime` | `pickup_datetime` | `tpep_pickup_datetime` |
| `dropoff_datetime` | `dropoff_datetime` | `tpep_dropoff_datetime` |
| `vendor_id` | `vendor_id` | `vendorid` |
| `ratecode_id` | `rate_code` | `ratecodeid` |
| `passenger_count` | `passenger_count` (default 0) | `passenger_count` (default 0) |

### Columns dropped

`tpep_pickup_datetime`, `pickup_datetime` (originals), `tpep_dropoff_datetime`, `dropoff_datetime` (originals), `vendor_id`, `vendorid`, `improvement_surcharge`, `imp_surcharge`, `store_and_fwd_flag`, `ratecodeid`, `rate_code`, `fare_amount_invalid_flag`, `passenger_count` (original)

---

## 2. Green Taxi Silver Table

**Path:** `3_Silver_Layer_Notebooks/3 Green Taxi Silver Table`

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Install dependencies | `%pip install apache-sedona` |
| 2 | — | Restart Python |
| 3 | Imports | SedonaContext + PySpark functions |
| 4 | — | Initialize Sedona spatial context |
| 5 | — | Load taxi zone polygons |
| 6 | Step 1 — Read table | Read `group3_gp.testing.green`, print initial row count |
| 7 | Step 2 — Consolidate duplicate columns | Map `lpep_pickup_datetime` → `pickup_datetime`, `lpep_dropoff_datetime` → `dropoff_datetime`, `vendorid` → `vendor_id`, `ratecodeid` → `ratecode_id`; default `passenger_count` to 0 |
| 8 | Step 3 — Drop redundant columns | Remove `lpep_pickup_datetime`, `lpep_dropoff_datetime`, `vendorid`, `improvement_surcharge`, `store_and_fwd_flag`, `ratecodeid`, `passenger_count`, `ehail_fee`, `trip_type` |
| 9 | — | Rename cleaned columns to final names |
| 10 | Step 4 — Cast columns | Same type casting as Yellow |
| 11 | Step 4b — Map legacy codes + cast to integer | Same payment/vendor/ratecode mapping as Yellow |
| 12 | Step 5 — Filter bad rows | Same filtering logic as Yellow |
| 13 | — | Preview post-2021 data |
| 14 | — | Add `row_id` for spatial join |
| 15 | — | Build point geometries for missing zone IDs |
| 16 | — | Spatial join with taxi zone polygons |
| 17 | — | Merge imputed zones, drop helpers, filter nulls |
| 18 | — | Print final row count |
| 19 | — | Write to `group3_gp.silver.green_taxi_trips` (Delta, overwrite) |

### Columns dropped

`lpep_pickup_datetime`, `lpep_dropoff_datetime`, `vendorid`, `improvement_surcharge`, `store_and_fwd_flag`, `ratecodeid`, `passenger_count` (original), `ehail_fee`, `trip_type`

### Notes

* Green uses `lpep_` prefix instead of `tpep_` (yellow)
* `ehail_fee` is almost entirely null and is dropped in this layer
* `trip_type` (street hail vs dispatch) is also dropped

---

## 3. For Hire Taxi Silver Table

**Path:** `3_Silver_Layer_Notebooks/3 For Hire Taxi Silver Table`

Simpler pipeline — no spatial join needed (FHV data has no lat/lon columns, only zone IDs from 2017 onwards).

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Imports | PySpark functions (no Sedona needed) |
| 2 | Step 1 — Read table | Read `group3_gp.testing.for_hire_vehicles`, print initial row count |
| 3 | Step 3 — Drop redundant columns | Remove `sr_flag`, `affiliated_base_number`, `dispatching_base_num` |
| 4 | Step 4 — Cast columns | Cast `pickup_datetime` and `dropoff_datetime` to timestamp; add `service_type = "fhv"` and `taxi_type = "unknown"` labels; cast `pulocationid` / `dolocationid` to integer |
| 5 | Step 5 — Filter bad rows | Remove rows with null timestamps, dropoff <= pickup, or null location IDs |
| 6 | — | Write to `group3_gp.silver.fhv_taxi_trips` (Delta, overwrite) |

### Columns dropped

`sr_flag`, `affiliated_base_number`, `dispatching_base_num`

### Columns added

| Column | Value | Purpose |
| --- | --- | --- |
| `service_type` | `"fhv"` | Identifies the service category |
| `taxi_type` | `"unknown"` | Placeholder — FHV data does not specify taxi type |

### Notes

* No fare or distance columns exist in the raw FHV data — the silver table inherits this limitation
* No spatial enrichment is performed (pre-2017 rows without zone IDs are filtered out)

---

## 4. High volume FHV Taxi Silver Table

**Path:** `3_Silver_Layer_Notebooks/3 High volume FHV Taxi Silver Table`

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Imports | PySpark functions (no Sedona) |
| 2 | Step 1 — Read table | Read `group3_gp.testing.high_volume_fhv`, print row and column counts |
| 3 | Step 3 — Drop columns | Remove `sr_flag`, `originating_base_num`, `dispatching_base_num`, `congestion_surcharge`, `wav_request_flag`, `wav_match_flag`, `cbd_congestion_fee`, `shared_request_flag`, `access_a_ride_flag` |
| 4 | — | Rename columns: `tips` → `tip_amount`, `tolls` → `tolls_amount`, `trip_miles` → `trip_distance` |
| 5 | Step 4 — Cast columns | Cast `pickup_datetime` and `dropoff_datetime` to timestamp; adds `service_type = "High Volume FHV"` and `taxi_type = "unknown"`; casts location IDs to integer |
| 6 | Step 5 — Filter bad rows | Filter null timestamps, dropoff <= pickup, null location IDs |
| 7 | — | Write to `group3_gp.silver.high_volume_fhv` (Delta, overwrite) |

### Columns dropped

`sr_flag`, `originating_base_num`, `dispatching_base_num`, `congestion_surcharge`, `wav_request_flag`, `wav_match_flag`, `cbd_congestion_fee`, `shared_request_flag`, `access_a_ride_flag`

### Columns renamed

| Original | Renamed |
| --- | --- |
| `tips` | `tip_amount` |
| `tolls` | `tolls_amount` |
| `trip_miles` | `trip_distance` |

### Columns added

| Column | Value | Purpose |
| --- | --- | --- |
| `service_type` | `"High Volume FHV"` | Identifies the service category |
| `taxi_type` | `"unknown"` | Placeholder |

---

## 5. Join Silver Tables (Combined)

**Path:** `3_Silver_Layer_Notebooks/3.5 join silver tables`

Unions all four individual silver tables into a single unified table with a shared schema, performs additional cleaning, calculates missing distances using spatial centroids, and deduplicates.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Install dependencies | `%pip install apache-sedona` |
| 2 | — | Restart Python |
| 3 | — | Initialize SedonaContext |
| 4 | Load silver tables | Read all 4 silver tables: `fhv_taxi_trips`, `yellow_taxi_trips`, `green_taxi_trips`, `high_volume_fhv` |
| 5 | Define target schema and union | Define the 19-column target schema and align each table using `select` with `cast` and `lit(None)` for missing columns; union all 4 DataFrames |
| 6–7 | Clean combined data | Replace faulty values with nulls: invalid zone IDs (≤0 or >265), unknown payment/rate/vendor labels; add boolean flags for calculated fields (`calculated_trip_distance`, `calculated_trip_time`, `calculated_total_amount`) |
| 8–9 | Calculate missing distances | Use Sedona to compute zone centroids from `group3_gp.gold.dim_zone` geometry (EPSG:2263 → 4326); join pickup/dropoff centroids; calculate Haversine distance between centroids for rows missing `trip_distance` |
| 10–11 | Finalize distances | Replace null `trip_distance` with the calculated centroid distance; drop helper columns (`pu_lon`, `pu_lat`, `do_lon`, `do_lat`, `calc_distance`, `base_passenger_fare`, `fare_amount`) |
| 12 | Convert miles to km | Multiply all `trip_distance` values by 1.60934 |
| 13–14 | Deduplicate | Run `dropDuplicates()` (full-row), then a second pass on key columns: `taxi_type`, `pickup_datetime`, `dropoff_datetime`, `pulocationid`, `dolocationid`, `trip_distance`, `total_amount` |
| 15–16 | Save combined table | Write to `group3_gp.silver.combined_taxi_trips` (Delta, overwrite with schema overwrite) |

### Target Schema

| Column | Type | Notes |
| --- | --- | --- |
| `taxi_type` | string | e.g. "Yellow", "Green", "unknown" |
| `service_type` | string | e.g. "yellow", "green", "fhv", "High Volume FHV" |
| `pickup_datetime` | timestamp | |
| `dropoff_datetime` | timestamp | |
| `trip_time` | double | In seconds |
| `dolocationid` | int | Dropoff zone ID (1–265) |
| `pulocationid` | int | Pickup zone ID (1–265) |
| `trip_distance` | double | Converted to km in final step |
| `extra` | decimal(10,2) | |
| `fare_amount` | decimal(10,2) | Dropped in final step |
| `mta_tax` | decimal(10,2) | |
| `tip_amount` | decimal(10,2) | |
| `tolls_amount` | decimal(10,2) | |
| `total_amount` | decimal(10,2) | |
| `payment_types` | string | Descriptive label |
| `rate_codes` | string | Descriptive label |
| `vendors` | string | Descriptive label |
| `base_passenger_fare` | decimal(10,2) | HVFHV only; dropped in final step |
| `bcf` | decimal(10,2) | Black car fund (HVFHV only) |

### Calculated boolean flags added

| Flag | Meaning |
| --- | --- |
| `calculated_trip_distance` | `true` if original distance was null/zero — value was computed from zone centroids |
| `calculated_trip_time` | `true` if original trip_time was null — value was computed from pickup/dropoff timestamps |
| `calculated_total_amount` | `true` if original total_amount was null — value was computed as sum of fare components |

### Key transformations

* Zone IDs outside 1–265 are set to null
* Unknown/unrecognised payment types, rate codes, and vendors are set to null
* Missing `trip_time` is calculated as `dropoff_datetime - pickup_datetime` in seconds
* Missing `total_amount` is calculated as sum of fare components
* Missing `trip_distance` is estimated using Haversine distance between zone centroids
* All distances are converted from miles to kilometres (×1.60934)
* Two-pass deduplication: full-row, then key-column subset

---

## Summary of Silver Tables

| Silver Table | Source | Spatial Join | Legacy Code Mapping | Write Complete |
| --- | --- | --- | --- | --- |
| `group3_gp.silver.yellow_taxi_trips` | Yellow | Yes (Sedona) | Yes (payment, vendor, ratecode) | Yes |
| `group3_gp.silver.green_taxi_trips` | Green | Yes (Sedona) | Yes (payment, vendor, ratecode) | Yes |
| `group3_gp.silver.fhv_taxi_trips` | FHV | No | N/A (no coded fields) | Yes |
| `group3_gp.silver.high_volume_fhv` | High Vol FHV | No | No (columns renamed only) | Yes |
| `group3_gp.silver.combined_taxi_trips` | All 4 above | Yes (centroid distance) | N/A (uses labels from upstream) | Yes |

---

## Dependencies

* **Apache Sedona** — Required by Yellow, Green, and join notebooks for spatial joins (`%pip install apache-sedona`)
* **`group3_gp.gold.dim_zone`** — Taxi zone polygon reference table (SRID 2263, NY State Plane) used for lat/lon → zone ID imputation and centroid distance calculation
* **`group3_gp.testing.*`** — Bronze-layer source tables (all columns stored as `STRING`)
