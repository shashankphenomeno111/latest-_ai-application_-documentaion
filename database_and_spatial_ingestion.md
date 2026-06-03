# Database Setup, Spatial Ingestion & Migration Pipeline

This document details the database architecture of Project Sahyadri 2.0, the ingestion of spatial infrastructure datasets, and the migration pipeline used to transfer over **3.1 Million (31 Lakh) records** of market transactions from a local SQLite development environment to a production-ready PostgreSQL instance.

---

## 1. Database Architecture Overview

Project Sahyadri 2.0 utilizes a dual-database approach to balance development ease with production scaling:

```
               +-------------------------------------------+
               |            RAW DATA SOURCE FILES          |
               | (Spatial JSONs / Market parquet datasets) |
               +---------------------+---------------------+
                                     |
                                     v
               +---------------------+---------------------+
               |         SPATIAL INGESTION SCRIPT          |
               |       (scripts/ingest_spatial_data.py)    |
               +---------------------+---------------------+
                                     |
                                     v
               +---------------------+---------------------+
               |        LOCAL DEVELOPMENT DATABASE         |
               |              (SQLite: sahyadri_data.db)   |
               +---------------------+---------------------+
                                     |
                                     v
               +---------------------+---------------------+
               |          POSTGRESQL MIGRATOR SCRIPT       |
               |      (scripts/migrate_sqlite_to_pg.py)    |
               +---------------------+---------------------+
                                     |
                                     v
               +---------------------+---------------------+
               |         PRODUCTION SERVICE DATABASE       |
               |          (PostgreSQL: port 5432)          |
               +-------------------------------------------+
```

* **SQLite (`sahyadri_data.db`):** Used during local development and testing. It stores transaction logs and local spatial registers (soil labs, banks, and warehouses).
* **PostgreSQL:** The production-grade target database. It manages concurrent user requests, runs fast spatial query filtering, and executes high-speed lookups using custom index keys.

---

## 2. Spatial Data Ingestion

The ingestion script [ingest_spatial_data.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/scripts/ingest_spatial_data.py) is responsible for extracting, cleaning, and seeding the database with geographical and structural information for four key registries:

### A. MSWC Commodity Warehouses (`warehouses`)
* **Source:** `warehouse_data/spatial_dataset/mswc_warehouses.json`
* **Coordinate Cleanup:** GeoJSON coordinates are represented as `[longitude, latitude]` (WGS84 format). The script swaps these to `[latitude, longitude]` to ensure standard mapping integration.
* **Pricing Tier Calculation:** Establishes realistic storage rents per quintal per month based on the capacity:
  * **Large warehouses (> 20,000 MT):** ₹10.0 / qtl / month
  * **Medium warehouses (5,000 - 20,000 MT):** ₹12.0 / qtl / month
  * **Small local warehouses (< 5,000 MT):** ₹15.0 / qtl / month

### B. DCCB Cooperative Branches (`dccb_branches`)
* **Source:** `warehouse_data/spatial_dataset/dccb_cooperatives.json`
* **Interest Rate Hashing:** Since cooperative interest rates vary, the script generates realistic interest rates deterministically using MD5 hashes of the bank and branch names:
  $$\text{interest\_rate} = 7.0\% + (\text{MD5}(\text{bank\_name} + \text{branch\_name}) \bmod 10) \times 0.1\%$$
  This yields rates between **7.0%** and **7.9%**, simulating real lending terms.

### C. Soil Testing Laboratories (`soil_testing_labs`)
* **Source:** `warehouse_data/spatial_dataset/soil_testing_labs.json`
* **Information Captured:** District name, lab type (Mobile vs. Static), status (Active/Inactive), and capacity (samples analyzed per month).

### D. Krishi Vigyan Kendras (`kvk_stations`)
* **Source:** `warehouse_data/spatial_dataset/kvk_registry.json`
* **Information Captured:** KVK name, associated district, agricultural university/institute affiliation, and web URL for agricultural extension services.

---

## 3. The Coordinate Estimation Engine

When raw data sources contain missing or corrupted geographical coordinates, the ingestion pipeline runs a fallback **Spatial Resolution Engine**:

```
[Spatial Registry Entry]
           |
           v
 Is there a valid coordinate array?
           |
           +----> YES ----> Parse [Lng, Lat] --> Swap axes --> Store [Lat, Lng]
           |
           +----> NO  ----> Scan address text against location_coords.json
                                   |
                          Was match found?
                                   |
                                   +----> YES ----> Extract & Store [Lat, Lng]
                                   |
                                   +----> NO  ----> Fallback to Maharashtra 
                                                    District Centroid Coord
```

1. **Address Parsing:** The engine cleans and checks if any location key inside the lookup file `location_coords.json` exists in the entity's text address.
2. **District Centroids:** If no location key is found, it uses the default coordinates of the corresponding Maharashtra district (e.g. `Latur: [18.4088, 76.5603]`).
3. **State Centroid:** If the district is also unknown, it falls back to the geographic center of Maharashtra `[19.7515, 75.7139]`.

---

## 4. PostgreSQL Database Migration

Migrating large datasets (3.1 Million+ rows) from a single-file SQLite database to a structured PostgreSQL server can cause time-outs or memory crashes if handled poorly. The migration script [migrate_sqlite_to_pg.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/scripts/migrate_sqlite_to_pg.py) uses several optimization strategies:

### A. Bulk Streaming in Batches
Rather than loading all 31 Lakh rows into memory or inserting rows one-by-one (which would take hours), the script uses a generator cursor:
* **Batch Size (`CHUNK_SIZE`):** **25,000 rows** per transaction.
* **Bulk Executions:** Uses `psycopg2.extras.execute_values()` which translates batch chunks into optimized, single-query SQL insert statements.
* **Resulting Throughput:** Achieves migration speeds of **10,000+ rows per second**, completing the transfer in under **5 minutes**.

### B. High-Performance Indexing
Once the data is written, PostgreSQL builds indexes to support instantaneous frontend dashboard search queries:

| Index Name | Target Fields | Intended Use Case |
| :--- | :--- | :--- |
| `idx_transactions_comm_dist` | `(commodity, district)` | Speeds up commodity price lookups by region. |
| `idx_transactions_source_market` | `(source_type, market_name)` | Enhances APMC, private mandi, and corporate procurement comparisons. |
| `idx_transactions_date` | `(date)` | Accelerates chronological queries for the TFT model. |
