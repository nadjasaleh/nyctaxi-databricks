# group3_gp.testing — Table Reference

This document describes all tables in the `group3_gp.testing` schema, including their columns and data types.

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

---

## 4. `yellow`

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
| 22 | `tpep_dropoff_datetime` | string |
| 23 | `imp_surcharge` | string |
| 24 | `pickup_datetime` | string |
| 25 | `rate_code` | string |
| 26 | `vendor_id` | string |
| 27 | `dolocationid` | string |
| 28 | `improvement_surcharge` | string |
| 29 | `pulocationid` | string |
| 30 | `congestion_surcharge` | string |
| 31 | `airport_fee` | string |

**Total columns:** 31

---

## Summary

| Table | Columns |
|-------|--------:|
| `for_hire_vehicles` | 8 |
| `green` | 25 |
| `high_volume_fhv` | 26 |
| `yellow` | 31 |

> **Note:** Most columns are stored as `string` in the raw/bronze layer. Downstream cleaning notebooks cast them to appropriate types (timestamp, integer, decimal, double, etc.).
