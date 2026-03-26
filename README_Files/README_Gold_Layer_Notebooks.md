# 4. Gold Layer Notebooks — Guide

This folder contains three notebooks that build the analytics-ready star schema in `group3_gp.gold`. The first creates dimension tables, the second creates fact tables aggregated at various grains, and the third records row counts at each pipeline stage for data lineage tracking.

---

## Notebooks Overview

| # | Notebook | Purpose | Tables Created |
| --- | --- | --- | --- |
| 1 | Create_Dim_Tables_Date_Hour_compan | Create dimension tables for date, hour, and company | `gold.dim_date`, `gold.dim_hour`, `gold.dim_company` |
| 2 | Fact_Table_All | Create all fact tables from combined silver data | 6 fact tables (see below) |
| 3 | Populate_Tablecount | Record row counts at each pipeline stage | `gold.tabel_count` |

---

## 1. Create Dimension Tables (Date, Hour, Company)

**Path:** `4_Gold_Layer_Notebooks/Create_Dim_Tables_Date_Hour_compan`

Creates three dimension tables used as join keys in all fact tables. Source data comes from `group3_gp.silver.combined_taxi_trips`.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Documentation | Markdown: lists the tables being created |
| 2 | Imports | PySpark functions for date manipulation |
| 3 | dim_date | Generate date dimension and write to `gold.dim_date` |
| 4 | dim_hour | Generate hour dimension and write to `gold.dim_hour` |
| 5 | dim_company | Create company dimension from distinct service/taxi types and write to `gold.dim_company` |

### Table: `group3_gp.gold.dim_date`

Generated programmatically from 2008-01-01 to current date using `sequence()` and `explode()`.

| Column | Type | Description |
| --- | --- | --- |
| `date_id` | long | Auto-generated via `monotonically_increasing_id()` |
| `date` | date | Calendar date |
| `year` | int | Year |
| `month` | int | Month number (1–12) |
| `month_name` | string | Month name (e.g. "January") |
| `day` | int | Day of month |
| `day_of_week_name` | string | Day name (e.g. "Monday") |
| `day_of_week_num` | int | Day of week number (1=Sunday, 7=Saturday) |
| `is_weekend` | boolean | `true` for Saturday and Sunday |
| `quarter` | int | Quarter (1–4) |
| `week_of_year` | int | ISO week number |
| `is_covid_period` | boolean | `true` for dates between 2020-03-01 and 2021-06-30 |

### Table: `group3_gp.gold.dim_hour`

Static 24-row table (hours 0–23).

| Column | Type | Description |
| --- | --- | --- |
| `hour_id` | long | Hour of day (0–23) |
| `period_of_day` | string | Time bucket: `morning` (5–11), `noon` (12–13), `afternoon` (14–17), `evening` (18–21), `night` (22–4) |
| `is_rush_traffic` | boolean | `true` for hours 7–9 and 16–18 |

### Table: `group3_gp.gold.dim_company`

Derived from distinct `(service_type, taxi_type)` combinations in `silver.combined_taxi_trips`.

| Column | Type | Description |
| --- | --- | --- |
| `company_id` | int | Auto-generated via `dense_rank()` |
| `service_type` | string | e.g. "yellow", "green", "fhv", "High Volume FHV" |
| `taxi_type` | string | e.g. "Yellow", "Green", "unknown" |

### Notes

* `dim_date` includes an `is_covid_period` flag (2020-03-01 to 2021-06-30) for pandemic impact analysis
* `dim_hour` uses simple time-of-day bucketing suitable for demand pattern analysis
* `dim_company` uses `dense_rank()` for gapless sequential IDs
* The `dim_zone` table is created in the Bronze Layer folder (`2.5_Create_Dimensions_Zones_Date_Tablecount`), not here

---

## 2. Fact Tables

**Path:** `4_Gold_Layer_Notebooks/Fact_Table_All`

Creates six fact tables by joining `group3_gp.silver.combined_taxi_trips` with the dimension tables. Each fact table is aggregated at a different grain for specific analytical use cases.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Test joins | Exploratory join of all dimension tables to verify keys work correctly |
| 2 | Documentation | Markdown: lists all fact tables and their locations |
| 3 | Fact 1 — Revenue and Fares | Aggregate by date × company × payment type |
| 4 | Fact 2 — Trip Demand Volume | Aggregate by date × hour × company |
| 5 | Fact 4 — Company and Market Share | Aggregate by date × company |
| 6 | Fact 5 — Trip Efficiency and Operations | Aggregate by company × pickup zone × dropoff zone |
| 7 | Fact 3 & 7 — Geography and Zones and Covid | Aggregate by date × company × pickup zone × dropoff zone |

### Fact table details

#### `group3_gp.gold.1_Fact_Revenue_And_Fares`

| Grain | Dimensions joined |
| --- | --- |
| date × company × payment_type | `dim_date`, `dim_company` |

| Measure | Description |
| --- | --- |
| `total_trips` | COUNT(*) |
| `total_trip_distance` / `avg_trip_distance` | SUM / AVG of trip_distance |
| `total_trip_time` / `avg_trip_time` | SUM / AVG of trip_time |
| `total_revenue` / `avg_revenue` | SUM / AVG of total_amount |
| `total_tip` / `avg_tip` | SUM / AVG of tip_amount |

---

#### `group3_gp.gold.2_Fact_Trip_Demand_Volume`

| Grain | Dimensions joined |
| --- | --- |
| date × hour × company | `dim_date`, `dim_hour`, `dim_company` |

| Measure | Description |
| --- | --- |
| `total_trips` | COUNT(*) |
| `total_trip_distance` / `avg_trip_distance` | SUM / AVG of trip_distance |
| `total_trip_time` / `avg_trip_time` | SUM / AVG of trip_time |

---

#### `group3_gp.gold.4_Fact_Company_and_Market_Share`

| Grain | Dimensions joined |
| --- | --- |
| date × company | `dim_date`, `dim_company` |

| Measure | Description |
| --- | --- |
| `total_trips` | COUNT(*) |
| `total_revenue` / `avg_revenue` | SUM / AVG of total_amount |
| `total_tip` / `avg_tip` | SUM / AVG of tip_amount |
| `total_trip_distance` / `avg_trip_distance` | SUM / AVG of trip_distance |
| `total_trip_time` / `avg_trip_time` | SUM / AVG of trip_time |

---

#### `group3_gp.gold.5_Fact_Trip_Efficiency_and_Operations`

| Grain | Dimensions joined |
| --- | --- |
| company × pickup_zone × dropoff_zone | `dim_company`, `dim_zone` (pickup + dropoff) |

| Measure | Description |
| --- | --- |
| `total_trips` | COUNT(*) |
| `total_trip_distance` / `avg_trip_distance` | SUM / AVG of trip_distance |
| `total_trip_time` / `avg_trip_time` | SUM / AVG of trip_time |
| `total_amount` / `avg_amount` | SUM / AVG of total_amount |

> **Note:** No date dimension — this table is aggregated across all time for route-level efficiency analysis.

---

#### `group3_gp.gold.3_and_7_Fact_Geography_and_Zones_and_Covid`

| Grain | Dimensions joined |
| --- | --- |
| date × company × pickup_zone × dropoff_zone | `dim_date`, `dim_company`, `dim_zone` (pickup + dropoff) |

| Measure | Description |
| --- | --- |
| `total_trips` | COUNT(*) |
| `total_trip_distance` / `avg_trip_distance` | SUM / AVG of trip_distance |
| `total_trip_time` / `avg_trip_time` | SUM / AVG of trip_time |
| `total_amount` / `avg_amount` | SUM / AVG of total_amount |

> **Note:** This is the most granular fact table. Combined with `dim_date.is_covid_period`, it supports both geographic and COVID impact analysis.

---

#### `group3_gp.testing.Fact_Grouped_By_Hour_Date_Company_Zone`

Stored in the **testing** schema (not gold). Created in cell 1 as an exploratory join test.

| Grain | Dimensions joined |
| --- | --- |
| date × hour × company × pickup_zone × dropoff_zone | All 4 dimensions |

| Measure | Description |
| --- | --- |
| `total_trips` | COUNT(*) |
| `total_trip_distance` | SUM of trip_distance |
| `total_trip_time` | SUM of trip_time |

---

### Join pattern

All fact tables follow the same join pattern against `silver.combined_taxi_trips`:

```sql
FROM group3_gp.silver.combined_taxi_trips t
JOIN group3_gp.gold.dim_date d     ON date(t.pickup_datetime) = d.date
JOIN group3_gp.gold.dim_company c  ON t.service_type = c.service_type
                                   AND t.taxi_type = c.taxi_type
JOIN group3_gp.gold.dim_hour h     ON hour(t.pickup_datetime) = h.hour_id
JOIN group3_gp.gold.dim_zone zpu   ON t.pulocationid = zpu.zone_id
JOIN group3_gp.gold.dim_zone zdo   ON t.dolocationid = zdo.zone_id
```

Not all dimensions are used in every fact table — each includes only the dimensions relevant to its grain.

---

## 3. Populate Table Count

**Path:** `4_Gold_Layer_Notebooks/Populate_Tablecount`

Records row counts at each pipeline stage for data lineage and quality tracking. Counts are stored in `group3_gp.gold.tabel_count`.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Create tracking table | Drop and recreate `gold.tabel_count` with columns: `step_id`, `source`, `row_count`, `reason` |
| 2 | Documentation | Markdown: "Bronze count" |
| 3 | Bronze counts (step 1) | Insert row counts from all 4 bronze tables + total |
| 4 | Silver counts (step 2) | Insert row counts from all 4 individual silver tables |
| 5 | Documentation | Markdown: "Combined silver" |
| 6 | Combined counts (steps 3–4) | Insert sum of silver counts (step 3) and actual combined table count (step 4) |
| 7 | View results | `SELECT * FROM gold.tabel_count ORDER BY step_id DESC` |

### Table: `group3_gp.gold.tabel_count`

| Column | Type | Description |
| --- | --- | --- |
| `step_id` | int | Pipeline stage: 1=Bronze, 2=Silver, 3=Combined (sum), 4=Combined (actual) |
| `source` | string | Table/stage identifier (e.g. "Yellow", "Yellow_Silver", "Combined_Final") |
| `row_count` | bigint | Number of rows |
| `reason` | string | Stage description (e.g. "Ingest", "After filtering, dropping obviously invalid values, missing geodata") |

### Pipeline stages tracked

| Step | Sources counted | Reason |
| --- | --- | --- |
| 1 | `bronze.for_hire_vehicles`, `bronze.high_volume_fhv`, `bronze.yellow`, `bronze.green`, Ingest Total | Ingest |
| 2 | `silver.fhv_taxi_trips`, `silver.high_volume_fhv`, `silver.yellow_taxi_trips`, `silver.green_taxi_trips` | After filtering, dropping obviously invalid values, missing geodata |
| 3 | Combined (sum of step 2 silver counts) | After filtering, dropping duplicates, obviously invalid values, missing geodata |
| 4 | `silver.combined_taxi_trips` (actual count) | After filtering, dropping duplicates, obviously invalid values, missing geodata |

### Notes

* Step 3 vs Step 4 comparison reveals how many rows were removed during the union and deduplication in `3.5 join silver tables`
* The table is dropped and recreated on each run — not appended to
* This should be run last in the pipeline, after all fact tables have been created

---

## Dependencies

* **`group3_gp.silver.combined_taxi_trips`** — Source for all dimension and fact tables
* **`group3_gp.gold.dim_date`** — Used in fact table joins
* **`group3_gp.gold.dim_hour`** — Used in fact table joins
* **`group3_gp.gold.dim_company`** — Used in fact table joins
* **`group3_gp.gold.dim_zone`** — Used in fact table joins (created in Bronze Layer folder)
* **All bronze and silver tables** — Used by Populate_Tablecount for row counting

---

## Execution Order

Run these notebooks after all silver-layer processing is complete:

1. **Create_Dim_Tables_Date_Hour_compan** — Create dimension tables first
2. **Fact_Table_All** — Create all fact tables (requires dimensions)
3. **Populate_Tablecount** — Record row counts (run last)
