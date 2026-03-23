# 0. Data Quality Check — Notebook Guide

This folder contains five notebooks that perform systematic data quality audits across the four NYC TLC trip datasets stored in `group3_gp.testing`. All numeric columns in these tables are stored as `STRING`, so queries use `TRY_CAST` / `CAST` for safe type conversion throughout.

---

## Notebooks Overview

| # | Notebook | Target Table | Purpose |
| --- | --- | --- | --- |
| 1 | Data quality check | All four tables | Master notebook — explores schemas, documents expected nulls, and runs bad-data checks across all datasets in one place |
| 2 | Yellow taxi data check | `group3_gp.testing.yellow` | Deep-dive quality audit for yellow taxi trips |
| 3 | Green taxi data check | `group3_gp.testing.green` | Deep-dive quality audit for green taxi trips |
| 4 | For H Vehicles data check | `group3_gp.testing.for_hire_vehicles` | Quality audit for FHV (For-Hire Vehicle) trips |
| 5 | High vol fhv data check | `group3_gp.testing.high_volume_fhv` | Quality audit for high-volume FHV trips (Uber, Lyft, etc.) |

---

## 1. Data quality check (Master Notebook)

**Path:** `0. Data quality check/Data quality check`

The central orchestration notebook that covers all four datasets. It performs:

* **Schema exploration** — Reads each table, prints row counts, column counts, and schema. Displays sample rows from both old-era (lat/long) and new-era (zone ID) formats.
* **Expected nulls documentation** — Defines which columns are legitimately null per era (e.g., `pickup_latitude` is null post-2017, `congestion_surcharge` is null pre-2019) to avoid false-positive quality flags.
* **Unexpected nulls scan** — Iterates over all columns in each table, counts nulls, and flags only those not in the expected-null list.
* **Bad data checks (SQL)** — Runs consolidated quality checks for each dataset:
  * Bad fares (excluding no-charge / voided trips)
  * Bad distances (zero or negative)
  * Bad timestamps (dropoff <= pickup)
  * Invalid ratecodes, passenger counts, base numbers
  * Duplicate row detection (proxy via full-row grouping)

---

## 2. Yellow taxi data check

**Path:** `0. Data quality check/Yellow taxi data check`  
**Table:** `group3_gp.testing.yellow` (~890M rows, 31 columns, 2011–2023)

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Schema and row count | Reads the table, prints row count, column count, schema, and `describe()` summary stats |
| 2 | Bad data checks | Single-pass SQL using `UNION ALL` — counts bad fares, legitimate zero fares, bad distances, bad timestamps, invalid ratecodes (99), store-and-forward flag, invalid passenger counts, and duplicate rows |
| 3 | Payment type breakdown | Maps both legacy string codes (`CRD`, `CSH`, `NOC`, `DIS`, `UNK`) and numeric codes to labels with trip counts and percentages |
| 4 | Rate code breakdown | Coalesces old (`rate_code`) and new (`ratecodeid`) columns; maps codes 1–6 and 99 to descriptions using `TRY_CAST(... AS DOUBLE)` |
| 5 | Vendor breakdown | Coalesces old (`vendor_id`) and new (`vendorid`) columns; maps to vendor names (Creative Mobile Technologies, Curb Mobility, legacy VTS/CMT) |
| 6 | Bad fares deep dive | PySpark analysis: zero vs negative fares, distance presence on zero-fare trips, payment type distribution, year concentration, and vendor concentration |
| 7 | Summary | Markdown cell with a full written summary of all findings (also exported as `README_testing_data_Summary.md`) |

### Key findings

* Bad fares are only 0.08% of total but spike in 2022–2023 (66% of all bad fares)
* Curb Mobility (vendor 2) accounts for 84% of all bad fares despite handling only 27% of trips
* Ratecode check using `TRY_CAST(... AS INT)` misses 347K rows stored as `"99.0"` — should use `TRY_CAST(... AS DOUBLE) = 99`
* Credit card payments dominate at 59%, cash at 40%

---

## 3. Green taxi data check

**Path:** `0. Data quality check/Green taxi data check`  
**Table:** `group3_gp.testing.green`

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Table overview and summary stats | Row count, column count, schema, and `describe()` |
| 2 | Bad data quality checks | SQL checks: bad fares (excl. no-charge/voided), legitimate zero fares, bad distances, bad timestamps, invalid ratecodes (99), invalid passenger counts |
| 3 | Trip type breakdown | Maps `trip_type` codes: 1 / 1.0 = Street hail, 2 / 2.0 = Dispatch |
| 4 | Year distribution | Trip volume per year to identify coverage gaps |
| 5 | Payment type breakdown | Maps payment codes (including `.0` variants) to labels with counts and percentages |
| 6 | Vendor breakdown | Maps `VendorID` to vendor names |

### Notes

* Green taxis use `lpep_` prefix (not `tpep_` like yellow)
* `ehail_fee` column is almost entirely null — flagged for removal in Silver layer
* Values stored inconsistently as both `"1"` and `"1.0"` across eras (applies to `trip_type`, `payment_type`, `ratecodeid`)

---

## 4. For H Vehicles data check

**Path:** `0. Data quality check/For H Vehicles data check`  
**Table:** `group3_gp.testing.for_hire_vehicles`

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | (Table overview) | Row count, column count, and schema |
| 2 | Bad data checks | SQL checks: bad timestamps (dropoff <= pickup), dirty `affiliated_base_number` values (not matching `B#####` pattern), null `pulocationid` |
| 3 | Shared ride flag distribution | Distribution of `sr_flag`: null = non-shared, `'1'` = shared ride |
| 4 | Dirty base number samples | Lists `affiliated_base_number` values that don't match the expected `B#####` regex pattern, with occurrence counts |
| 5 | Year distribution | Trip volume per year |

### Notes

* Very sparse schema — only 8 columns, no fare or distance data
* TLC only required base number, datetime, and location for FHV trips
* `dropoff_datetime` and dropoff location only available from 2017 onwards
* `affiliated_base_number` links to the base that received the original dispatch request
* Valid base numbers follow the pattern `B` followed by exactly 5 digits (e.g., `B02395`)

---

## 5. High vol fhv data check

**Path:** `0. Data quality check/High vol fhv data check`  
**Table:** `group3_gp.testing.high_volume_fhv`

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | (Table overview) | Row count, column count, and schema |
| 2 | Bad data checks | SQL checks: bad timestamps, bad distances (`trip_miles <= 0`), bad fares (populated but <= 0), bad driver pay (populated but <= 0) |
| 3 | Company breakdown | Maps `hvfhs_license_num` to companies: HV0002 = Juno, HV0003 = Uber, HV0004 = Via, HV0005 = Lyft |
| 4 | Fare availability by year | Shows fare column population rate per year — confirms nulls pre-2021 are expected |
| 5 | Shared ride breakdown | Shared request/match flags broken down by company. Note: Lyft (HV0005) overcounts shared rides by flagging requested-but-unmatched rides |
| 6 | WAV trip breakdown | Wheelchair-accessible vehicle (WAV) request vs match rates |
| 7 | Year distribution | Trip volume per year |

### Notes

* Fare columns (`base_passenger_fare`, `driver_pay`, tips, tolls, etc.) only populated from ~2021 onwards — nulls in earlier years are expected, not errors
* `on_scene_datetime` only populated for WAV (wheelchair-accessible) trips
* Lyft overcounts shared rides — also flags requested-but-unmatched rides as shared
* Four companies identified by license number: Uber dominates volume, followed by Lyft, with Juno and Via as smaller players

---

## Common Patterns Across Notebooks

* **All-STRING schema:** Every numeric column is stored as `STRING` in the testing tables. All notebooks use `TRY_CAST` or `CAST` for safe conversion.
* **Era transitions:** Pre-2017 data uses lat/long coordinates; post-2017 uses zone IDs (`pulocationid` / `dolocationid`). Column names also changed (e.g., `vendor_id` → `vendorid`, `rate_code` → `ratecodeid`).
* **Expected vs unexpected nulls:** Each notebook distinguishes between structurally expected nulls (era-dependent columns, surcharges introduced in later years) and genuine data quality issues.
* **Single-pass SQL checks:** Bad data checks use `UNION ALL` patterns to scan each table once rather than running separate queries per check.
