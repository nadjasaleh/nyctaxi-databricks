# 2. Bronze Layer Notebooks — Guide

This folder contains two notebooks that handle the initial data ingestion into the bronze layer and the creation of foundational reference tables. The first notebook loads raw trip data from ADLS into Delta tables; the second creates the zone dimension table and a row-count tracking table.

> **Note:** For detailed bronze table schemas, see [`group3_gp_bronze_README.md`](group3_gp_bronze_README.md).

---

## Notebooks Overview

| # | Notebook | Purpose | Tables Created/Modified |
| --- | --- | --- | --- |
| 1 | Bronze_Ingestion_From_ADLS | Ingest JSON trip data into bronze Delta tables | `group3_gp.bronze.yellow`, `.green`, `.for_hire_vehicles`, `.high_volume_fhv` |
| 2 | 2.5_Create_Dimensions_Zones_Date_Tablecount | Create zone dimension and row-count tracking table | `group3_gp.gold.dim_zone`, `group3_gp.gold.tabel_count` |

---

## 1. Bronze Ingestion From ADLS

**Path:** `2_Bronze_Layer_Notebooks/Bronze_Ingestion_From_ADLS`

Processes JSON files from a Unity Catalog volume and loads them into bronze Delta tables. Files are categorized by filename and unioned per category.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Create schema | `CREATE SCHEMA IF NOT EXISTS group3_gp.bronze` |
| 2 | Drop existing tables | Drops all four trip tables to allow clean re-ingestion |
| 3 | Documentation | Markdown: describes the source folder, target database, and processing approach |
| 4 | Ingest and write | PySpark: reads JSON files, categorizes by filename, unions per category, writes to Delta |

### Configuration

| Setting | Value |
| --- | --- |
| Source folder | `/Volumes/group3_gp/adlslanding/looped_data` |
| Target database | `group3_gp.bronze` |
| File format | JSON |
| Write mode | Overwrite (tables are dropped first) |

### Category mapping

Files are categorized by matching keywords in their filenames:

| Filename keyword | Bronze table |
| --- | --- |
| `yellow` | `group3_gp.bronze.yellow` |
| `green` | `group3_gp.bronze.green` |
| `fhv` | `group3_gp.bronze.high_volume_fhv` |
| `for hire` | `group3_gp.bronze.for_hire_vehicles` |

### Processing logic

1. List all files in the source volume
2. For each file, match the filename against the category map
3. Read the JSON file into a Spark DataFrame and add a `Year` column extracted from the filename
4. Collect all DataFrames per category into a list
5. Union all DataFrames within each category using `unionByName(allowMissingColumns=True)` to handle schema evolution across years
6. Write each unioned DataFrame to a Delta table in `group3_gp.bronze`

### Notes

* The notebook drops all four tables before re-ingesting — this is a full overwrite pattern, not incremental
* `unionByName(allowMissingColumns=True)` handles schema differences across years (e.g. columns added in later years like `congestion_surcharge`, `airport_fee`)
* All columns are ingested as-is from JSON — most trip fields arrive as `STRING` type
* The `Year` column is added during ingestion, extracted from the source filename

---

## 2. Create Dimensions: Zones, Date, Tablecount

**Path:** `2_Bronze_Layer_Notebooks/2.5_Create_Dimensions_Zones_Date_Tablecount`

Creates foundational reference tables that are used by downstream silver and gold notebooks. This is a single-cell SQL notebook.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Create dim_zone | SQL: drops and recreates `group3_gp.gold.dim_zone` by joining the two bronze zone tables |

### Table: `group3_gp.gold.dim_zone`

Created by joining `bronze.taxi_zones_lookup` (primary) with `bronze.taxi_zones` (geometry) on `LocationID`:

```sql
CREATE OR REPLACE TABLE group3_gp.gold.dim_zone AS
SELECT
    l.LocationID AS zone_id,
    COALESCE(l.Zone, z.zone) AS zone_name,
    COALESCE(l.Borough, z.borough) AS borough,
    l.service_zone,
    z.geometry_wkt
FROM group3_gp.bronze.taxi_zones_lookup l
LEFT JOIN group3_gp.bronze.taxi_zones z
    ON l.LocationID = z.LocationID;
```

| Column | Type | Source |
| --- | --- | --- |
| `zone_id` | int | `taxi_zones_lookup.LocationID` |
| `zone_name` | string | Coalesce of lookup `Zone` and zones `zone` |
| `borough` | string | Coalesce of lookup `Borough` and zones `borough` |
| `service_zone` | string | `taxi_zones_lookup.service_zone` |
| `geometry_wkt` | string | `taxi_zones.geometry_wkt` (WKT polygon, EPSG:2263) |

### Notes

* Uses `LEFT JOIN` so all locations from the lookup table are preserved, even if the shapefile is missing a geometry
* `COALESCE` ensures zone name and borough are populated from either source
* The `geometry_wkt` column contains Well-Known Text (WKT) representations of zone boundary polygons in EPSG:2263 (NY State Plane) projection
* The `tabel_count` tracking table is initialized in the `Populate_Tablecount` notebook (see Gold Layer README)
* This notebook should be run after zone reference data has been ingested (see Pre-Process folder) and before any silver-layer notebooks

---

## Dependencies

* **`group3_gp.bronze.taxi_zones`** — Zone boundary polygons (created by `1_pre_process/ingest_zones_shp`)
* **`group3_gp.bronze.taxi_zones_lookup`** — Zone ID lookup (created by `1_pre_process/ingest_zones_shp`)
* **ADLS Volume** — JSON trip data at `/Volumes/group3_gp/adlslanding/looped_data` (created by `1_pre_process/Convert_parquet_to_json_upload_to_ADLS`)

---

## Execution Order

Run these notebooks after pre-processing and before silver-layer cleaning:

1. **Bronze_Ingestion_From_ADLS** — Load raw trip data into bronze tables
2. **2.5_Create_Dimensions_Zones_Date_Tablecount** — Create zone dimension table
