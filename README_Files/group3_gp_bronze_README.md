# group3_gp.bronze — Table Reference

This document describes all tables in the `group3_gp.bronze` schema, including their columns and data types.

---

## 1. `for_hire_vehicles`

For-hire vehicle (FHV) trip records.

| # | Column | Data Type |
|---|--------|----------|
| 1 | `dispatching_base_num` | string |
| 2 | `dolocationid` | string |
| 3 | `dropoff_datetime` | string |
| 4 | `pickup_datetime` | string |
| 5 | `pulocationid` | string |
| 6 | `sr_flag` | string |
| 7 | `Year` | int |
| 8 | `affiliated_base_number` | string |

**Total columns:** 8  
**Table type:** Managed

---

## 2. `green`

Green taxi trip records.

| # | Column | Data Type |
|---|--------|----------|
| 1 | `dropoff_latitude` | string |
| 2 | `dropoff_longitude` | string |
| 3 | `extra` | string |
| 4 | `fare_amount` | string |
| 5 | `lpep_dropoff_datetime` | string |
| 6 | `lpep_pickup_datetime` | string |
| 7 | `mta_tax` | string |
| 8 | `passenger_count` | string |
| 9 | `payment_type` | string |
| 10 | `pickup_latitude` | string |
| 11 | `pickup_longitude` | string |
| 12 | `ratecodeid` | string |
| 13 | `store_and_fwd_flag` | string |
| 14 | `tip_amount` | string |
| 15 | `tolls_amount` | string |
| 16 | `total_amount` | string |
| 17 | `trip_distance` | string |
| 18 | `trip_type` | string |
| 19 | `vendorid` | string |
| 20 | `Year` | int |
| 21 | `improvement_surcharge` | string |
| 22 | `dolocationid` | string |
| 23 | `pulocationid` | string |
| 24 | `congestion_surcharge` | string |
| 25 | `ehail_fee` | string |

**Total columns:** 25  
**Table type:** Managed

---

## 3. `high_volume_fhv`

High-volume for-hire vehicle (HVFHV) trip records (e.g. Uber, Lyft).

| # | Column | Data Type |
|---|--------|----------|
| 1 | `dispatching_base_num` | string |
| 2 | `dolocationid` | string |
| 3 | `dropoff_datetime` | string |
| 4 | `hvfhs_license_num` | string |
| 5 | `pickup_datetime` | string |
| 6 | `pulocationid` | string |
| 7 | `sr_flag` | string |
| 8 | `Year` | int |
| 9 | `access_a_ride_flag` | string |
| 10 | `airport_fee` | string |
| 11 | `base_passenger_fare` | string |
| 12 | `bcf` | string |
| 13 | `congestion_surcharge` | string |
| 14 | `driver_pay` | string |
| 15 | `on_scene_datetime` | string |
| 16 | `originating_base_num` | string |
| 17 | `request_datetime` | string |
| 18 | `sales_tax` | string |
| 19 | `shared_match_flag` | string |
| 20 | `shared_request_flag` | string |
| 21 | `tips` | string |
| 22 | `tolls` | string |
| 23 | `trip_miles` | string |
| 24 | `trip_time` | string |
| 25 | `wav_match_flag` | boolean |
| 26 | `wav_request_flag` | string |

**Total columns:** 26  
**Table type:** Managed

---

## 4. `taxi_zones`

Taxi zone boundary polygons with geometry data. Used for spatial enrichment (lat/lon → zone ID mapping).

| # | Column | Data Type |
|---|--------|----------|
| 1 | `OBJECTID` | int |
| 2 | `Shape_Leng` | double |
| 3 | `Shape_Area` | double |
| 4 | `zone` | string |
| 5 | `LocationID` | int |
| 6 | `borough` | string |
| 7 | `geometry_wkt` | string |

**Total columns:** 7  
**Table type:** Managed  
**Frequently joined with:** `taxi_zones_lookup`

---

## 5. `taxi_zones_lookup`

Taxi zone lookup table mapping location IDs to borough, zone name, and service zone.

| # | Column | Data Type |
|---|--------|----------|
| 1 | `LocationID` | int |
| 2 | `Borough` | string |
| 3 | `Zone` | string |
| 4 | `service_zone` | string |

**Total columns:** 4  
**Table type:** External  
**Frequently joined with:** `taxi_zones`, `group3_gp.testing.yellow`

---

## 6. `testingnyc`

A subset of yellow taxi trip columns (new-era only, post-2016). Appears to be a testing or staging table with a reduced schema — no lat/lon, no legacy column name variants.

| # | Column | Data Type |
|---|--------|----------|
| 1 | `dolocationid` | string |
| 2 | `extra` | string |
| 3 | `fare_amount` | string |
| 4 | `improvement_surcharge` | string |
| 5 | `mta_tax` | string |
| 6 | `passenger_count` | string |
| 7 | `payment_type` | string |
| 8 | `pulocationid` | string |
| 9 | `ratecodeid` | string |
| 10 | `store_and_fwd_flag` | string |
| 11 | `tip_amount` | string |
| 12 | `tolls_amount` | string |
| 13 | `total_amount` | string |
| 14 | `tpep_dropoff_datetime` | string |
| 15 | `tpep_pickup_datetime` | string |
| 16 | `trip_distance` | string |
| 17 | `vendorid` | string |

**Total columns:** 17  
**Table type:** Managed

---

## 7. `yellow`

Yellow taxi trip records.

| # | Column | Data Type |
|---|--------|----------|
| 1 | `dropoff_datetime` | string |
| 2 | `dropoff_latitude` | string |
| 3 | `dropoff_location` | struct\<coordinates:array\<double\>, type:string\> |
| 4 | `dropoff_longitude` | string |
| 5 | `extra` | string |
| 6 | `fare_amount` | string |
| 7 | `mta_tax` | string |
| 8 | `passenger_count` | string |
| 9 | `payment_type` | string |
| 10 | `pickup_latitude` | string |
| 11 | `pickup_location` | struct\<coordinates:array\<double\>, type:string\> |
| 12 | `pickup_longitude` | string |
| 13 | `ratecodeid` | string |
| 14 | `store_and_fwd_flag` | string |
| 15 | `tip_amount` | string |
| 16 | `tolls_amount` | string |
| 17 | `total_amount` | string |
| 18 | `tpep_pickup_datetime` | string |
| 19 | `trip_distance` | string |
| 20 | `vendorid` | string |
| 21 | `Year` | int |
| 22 | `imp_surcharge` | string |
| 23 | `pickup_datetime` | string |
| 24 | `rate_code` | string |
| 25 | `vendor_id` | string |
| 26 | `dolocationid` | string |
| 27 | `improvement_surcharge` | string |
| 28 | `pulocationid` | string |
| 29 | `tpep_dropoff_datetime` | string |
| 30 | `congestion_surcharge` | string |
| 31 | `airport_fee` | string |

**Total columns:** 31  
**Table type:** Managed

---

## Summary

| Table | Columns | Type | Description |
|-------|--------:|------|-------------|
| `for_hire_vehicles` | 8 | Managed | FHV trip records |
| `green` | 25 | Managed | Green taxi trip records |
| `high_volume_fhv` | 26 | Managed | High-volume FHV trip records (Uber, Lyft, etc.) |
| `taxi_zones` | 7 | Managed | Zone boundary polygons with WKT geometry |
| `taxi_zones_lookup` | 4 | External | Zone ID → borough / zone name / service zone lookup |
| `testingnyc` | 17 | Managed | Reduced yellow taxi subset (new-era columns only) |
| `yellow` | 31 | Managed | Yellow taxi trip records |

> **Note:** Most trip columns are stored as `string` in the bronze layer. Downstream cleaning notebooks (Silver Layer) cast them to appropriate types (timestamp, integer, decimal, double, etc.). The `taxi_zones` and `taxi_zones_lookup` tables are reference data with properly typed columns.
