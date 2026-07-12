# sql-data-warehouse-project

A data warehouse built in SQL Server using the medallion architecture (bronze, silver, gold), following Baraa Khatib Salkini's Data With Baraa course. This project ingests raw CSV data from two source systems, cleans and standardizes it, then models it into a star schema ready for reporting and analytics.

## Project Overview

**Goal:** build an end-to-end data warehouse that takes raw operational data from two disconnected source systems and turns it into a single, reliable, business-ready dataset.

**Approach:** medallion architecture. Three layers, each with one clearly scoped job, so that data quality, transformation logic, and business modeling never mix in the same layer.

**Sources:** CSV files from two systems, CRM and ERP. No live database connection, no API, just flat files landing in a folder.

## Data Architecture

```
Sources (CSV files)          Data Warehouse                  Consumers
┌─────────────┐        ┌──────────────────────┐        ┌──────────────────┐
│    CRM      │───┐    │   BRONZE  (tables)   │        │  BI / Reporting  │
│    ERP      │───┼───▶│   raw, as-is         │        │  (Power BI /     │
└─────────────┘   │    ├──────────────────────┤        │   Tableau)       │
                   │    │   SILVER  (tables)   │        ├──────────────────┤
                   └───▶│   cleaned, standard  │───────▶│  Ad hoc SQL      │
                        ├──────────────────────┤        │  analysis        │
                        │   GOLD  (views)      │        ├──────────────────┤
                        │   business ready     │───────▶│  Machine         │
                        └──────────────────────┘        │  learning        │
                                                          └──────────────────┘
```

### Layer specifications

| Layer | Object Type | Load Method | Transformations | Data Model |
|---|---|---|---|---|
| **Bronze** | Tables | Full load (truncate & insert) | None, raw as-is | None, matches source exactly |
| **Silver** | Tables | Full load (truncate & insert) | Cleansing, standardization | None, still one-to-one with source |
| **Gold** | Views | None, virtual | Business logic, joins, aggregation | Star schema (facts + dimensions) |

**Why bronze exists:** traceability. If something breaks downstream, the raw source data is still sitting there untouched, so the root cause is always findable.

**Why gold is views, not tables:** the gold layer is the one place where "always fresh, no separate refresh job" outweighs raw query speed. If gold views ever get slow enough to hurt reporting, that's the trigger to swap specific views for CTAS tables, not a reason to abandon views everywhere.

## Source Systems

| System | Tables | Format |
|---|---|---|
| **CRM** | Customer info, product info, sales details | CSV |
| **ERP** | Customer location, customer additional info, product category | CSV |

## Naming Conventions

- **Case:** snake_case throughout. All lowercase, words separated by underscores.
- **Language:** English.
- **Reserved words:** never used as object names (no table called `table`, no column called `date`, etc).

| Layer | Table naming rule | Example |
|---|---|---|
| Bronze | `sourcesystem_entity` | `crm_cust_info`, `erp_cust_az12` |
| Silver | Same as bronze, one-to-one, no renaming | `crm_cust_info`, `erp_cust_az12` |
| Gold | `category_entity`, business-aligned, no source prefix | `dim_customers`, `dim_products`, `fact_sales` |

Gold drops the source-system prefix because gold tables often combine multiple sources into one business object, so a single source name would be misleading.

## Repository Structure

```
sql-data-warehouse-project/
├── datasets/              # Raw CSV source files (CRM/, ERP/)
├── docs/                  # Data architecture, data flow, data model diagrams
├── scripts/
│   ├── bronze/            # DDL + load scripts for bronze layer
│   ├── silver/            # DDL + transformation scripts for silver layer
│   └── gold/               # DDL for gold views (ddl_gold.sql)
├── tests/
│   └── quality_checks_gold.sql   # Validation for star schema integrity
└── README.md
```

## How to Run

1. Create the database and schemas:
   ```sql
   CREATE DATABASE datawarehouse;
   USE datawarehouse;
   CREATE SCHEMA bronze;
   CREATE SCHEMA silver;
   CREATE SCHEMA gold;
   ```
2. Run `scripts/bronze/ddl_bronze.sql` to create bronze tables, then load them via `BULK INSERT` from the CSV files in `datasets/`.
3. Run the silver layer stored procedure to truncate, clean, and reload from bronze into silver.
4. Run `scripts/gold/ddl_gold.sql` to create the gold-layer views on top of silver.
5. Run `tests/quality_checks_gold.sql` after any change to bronze or silver to confirm the star schema still holds (surrogate key uniqueness, fact-to-dimension relationships intact).

## Data Quality Notes

The silver layer is where known source-data issues get resolved, including inconsistent customer key formats between CRM and ERP (stray prefix characters that break joins if left untouched) and missing or invalid values in customer and product fields. Every transformation rule here has a reason documented in the silver DDL script, not just applied silently.

## About Me

*(Nick — add a short bio and links here: LinkedIn, GitHub, etc.)*

## License

*(Add license here, or note this is a personal learning project if it's not intended for reuse.)*
