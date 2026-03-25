# 1. Pre-Process — Notebook Guide

This folder contains two notebooks that prepare raw source data before it can be ingested into the bronze layer. The first converts and uploads trip data files; the second ingests geographic reference data for taxi zones.

---

## Notebooks Overview

| # | Notebook | Purpose | Output |
| --- | --- | --- | --- |
| 1 | Convert_parquet_to_json_upload_to_ADLS | Converts parquet trip data files to JSON and re-uploads to ADLS | JSON files in `LoopedNYCData/` container |
| 2 | ingest_zones_shp | Ingests NYC taxi zone shapefile and lookup CSV into bronze tables | `group3_gp.bronze.taxi_zones`, `group3_gp.bronze.taxi_zones_lookup` |

---

## 1. Convert Parquet to JSON and Upload to ADLS

**Path:** `1_pre_process/Convert_parquet_to_json_upload_to_ADLS`

This single-cell notebook reads parquet trip data files from the ADLS landing zone, converts them to JSON format, and writes them back to a different ADLS container for downstream bronze ingestion.

### Configuration

| Setting | Value |
| --- | --- |
| Storage account | `group3gpsa` |
| Source directory | `abfss://landing@group3gpsa.dfs.core.windows.net/MonthlyNycData` |
| Target directory | `abfss://landing@group3gpsa.dfs.core.windows.net/LoopedNYCData` |
| File pattern | `(yellow|green|fhv|fhvhv)_tripdata_(2025|2026).parquet` |

### Processing logic

1. Lists all files in the source directory
2. Filters files matching the naming pattern using a regex: `yellow`, `green`, `fhv`, or `fhvhv` trip data for years 2025–2026
3. For each matching file:
   * Reads the parquet file into a Spark DataFrame
   * Maps the trip type to a human-readable label (e.g. `"yellow"` → `"Yellow"`, `"fhvhv"` → `"High Volume FHV"`)
   * Writes the DataFrame as JSON to the target directory with a descriptive subfolder name

### Trip type labels

| File prefix | Label |
| --- | --- |
| `yellow` | Yellow |
| `green` | Green |
| `fhv` | For Hire Vehicles |
| `fhvhv` | High Volume FHV |

### Notes

* The notebook is a single cell — all logic is self-contained
* Only processes 2025–2026 data (the regex pattern limits the year range)
* The JSON output in `LoopedNYCData/` is what the bronze ingestion notebook reads from

---

## 2. Ingest Zones Shapefile

**Path:** `1_pre_process/ingest_zones_shp`

Ingests NYC taxi zone geographic boundaries and a lookup table into the bronze layer. These reference tables are used downstream for spatial enrichment (lat/lon → zone ID imputation) in the silver layer.

### Cells

| Cell | Title | Description |
| --- | --- | --- |
| 1 | Create schema | `CREATE SCHEMA IF NOT EXISTS group3_gp.bronze` |
| 2 | Install GeoPandas | `!pip install geopandas` |
| 3 | Import | `import geopandas as gpd` |
| 4 | Read shapefile | Read taxi zone shapefile from `/Volumes/group3_gp/adlslanding/geodata_test/taxi_zones.shp` using GeoPandas |
| 5 | Convert geometry | Convert geometry column to WKT string (`geometry_wkt`), drop the native geometry column |
| 6 | Write taxi_zones | Convert to Spark DataFrame and write to `bronze.taxi_zones` (Delta, overwrite) |
| 7 | Create taxi_zones_lookup | Create external CSV table `bronze.taxi_zones_lookup` pointing to `abfss://landing@group3gpsa.dfs.core.windows.net/taxi_zone_lookup.csv` |

### Tables created

| Table | Format | Source | Description |
| --- | --- | --- | --- |
| `group3_gp.bronze.taxi_zones` | Delta (managed) | Shapefile | Zone boundaries with WKT geometry, borough, zone name (7 columns) |
| `group3_gp.bronze.taxi_zones_lookup` | CSV (external) | CSV on ADLS | Zone ID → borough, zone name, service zone mapping (4 columns) |

### Notes

* The shapefile uses **EPSG:2263** (NY State Plane) coordinate reference system
* Geometry is stored as WKT strings rather than native geometry types for Delta compatibility
* The lookup table is external (reads directly from ADLS CSV) while the zones table is managed Delta
* Both tables are frequently joined downstream: `taxi_zones_lookup` provides names, `taxi_zones` provides geometry for spatial operations

---

## Dependencies

* **GeoPandas** — Required for reading the shapefile (`!pip install geopandas`)
* **Azure Data Lake Storage** — Source and target storage (`group3gpsa.dfs.core.windows.net`)
* **Unity Catalog Volume** — Shapefile accessed via `/Volumes/group3_gp/adlslanding/geodata_test/`

---

## Execution Order

Run these notebooks before any bronze ingestion:

1. **Convert_parquet_to_json_upload_to_ADLS** — Prepares trip data files
2. **ingest_zones_shp** — Ingests geographic reference data
