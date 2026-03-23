%md
## Yellow Taxi Data Quality — Summary

### Dataset Overview

| Property | Value |
| --- | --- |
| Table | `group3_gp.testing.yellow` |
| Rows | 890,945,235 |
| Columns | 31 |
| Time span | 2011–2023 |
| Schema note | All numeric columns stored as `STRING` — requires explicit casting |

---

### Data Quality Checks

| Check | Count | % of Total |
| --- | ---: | ---: |
| Bad fares (excl. no-charge & voided) | 754,287 | 0.08% |
| Legitimate zero fares (no-charge / voided) | 249,905 | 0.03% |
| Bad distances (zero or negative) | 6,845,654 | 0.77% |
| Bad timestamps (dropoff ≤ pickup) | 1,772,754 | 0.20% |
| Invalid ratecode (99) | 18,553 | <0.01% |
| Stored offline (store_and_fwd_flag = Y) | 8,434,814 | 0.95% |
| Invalid passenger count (<1 or >8) | 2,697,961 | 0.30% |
| Duplicate rows (proxy) | 9,725,548 | 1.09% |

> **Ratecode check gap:** The current check uses `TRY_CAST(... AS INT)`, which silently returns NULL for decimal-formatted strings like `"99.0"`. This catches only 18,553 rows (`"99"`), missing an additional 347,274 rows stored as `"99.0"`. True total: **365,827**. Fix: use `TRY_CAST(... AS DOUBLE) = 99`.

---

### Payment Types

Legacy string codes (`CRD`, `CSH`, `NOC`, `DIS`, `UNK`) from the pre-2017 era coexist with numeric codes. Combined:

| Payment method | Trips | Share |
| --- | ---: | ---: |
| Credit card (1 + CRD) | 528,482,486 | 59.32% |
| Cash (2 + CSH) | 353,704,214 | 39.70% |
| Flex fare (0) | 2,677,659 | 0.30% |
| No charge (3 + NOC) | 2,672,358 | 0.30% |
| Dispute (4 + DIS) | 1,497,910 | 0.17% |
| Unknown (UNK + 5) / NULL | 1,910,608 | 0.21% |

---

### Rate Codes

Values stored inconsistently as both `"1"` and `"1.0"` across eras. Combined:

| Rate code | Trips |
| --- | ---: |
| Standard rate (1) | 863,116,587 |
| JFK (2) | 18,507,226 |
| Negotiated fare (5) | 2,936,163 |
| Newark (3) | 1,553,436 |
| Nassau / Westchester (4) | 806,149 |
| Group ride (6) | 10,278 |
| Unknown / invalid (99) | 365,827 |
| NULL | 3,585,579 |
| Other unrecognised codes | \~63,990 |

---

### Vendors

Two legacy string codes (`VTS`, `CMT`) represent \~53% of trips from the pre-2017 era.

| Vendor | Trips | Share |
| --- | ---: | ---: |
| Curb Mobility (2) | 240,268,506 | 26.97% |
| VTS (legacy) | 237,408,719 | 26.65% |
| CMT (legacy) | 236,221,325 | 26.51% |
| Creative Mobile Technologies (1) | 175,987,369 | 19.75% |
| NULL / Other | 1,059,316 | 0.12% |

---

### Bad Fares — Deep Dive

**Negative vs zero:** Negative fares (867,418) outnumber zero fares (137,478) by roughly 6:1.

**Zero-fare trips with distance:** 57% of zero-fare trips (77,812 of 137,478) still have a recorded trip distance — likely meter or logging errors rather than genuinely free rides.

**Zero-fare payment types:** Cash dominates (64,325), followed by no-charge (31,245) and credit card (27,746).

**Year concentration:** Bad fares spike in 2022–2023 (664,677 combined, 66% of all bad fares). Smaller clusters appear around 2015–2016 and 2020.

| Year | Bad fare count |
| --- | ---: |
| 2023 | 394,503 |
| 2022 | 270,174 |
| 2020 | 103,827 |
| 2016 | 93,133 |
| 2015 | 92,912 |
| Others | \~50,347 |

**Vendor concentration:** Curb Mobility (vendor 2) accounts for **84%** of all bad fares (842,249 of \~1,004,896). This is disproportionate given it handles only 27% of total trips.