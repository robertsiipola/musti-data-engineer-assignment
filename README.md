# Overview

You’ll build a small but realistic data pipeline around retail transactions. The goal is to ingest raw files, model clean analytics tables, enrich with FX rates, and produce reliable daily metrics—then explain your design choices. Everything runs locally with DuckDB + Python.

Estimated effort: ~4–6 hours. You’ll have up to 10 calendar days to submit. You are encouraged to us AI tools in this assignment, but be prepared to explain all the code that the models generate.

# Data you’ll receive

```
data/
  raw/
    online_retail_II.xlsx             # e-commerce transactions (Excel; Use both tabs from the excel)
    gbp.xml                           # historical EUR FX
    ukbankholidays-jul19.xls          # UK public holidays
```

> The retail file contains columns like InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country. Cancellations appear as negative quantities and/or InvoiceNo that starts with “C”. FX files are EUR-base reference rates (1 EUR = X units of currency). Work offline—do not fetch anything from the internet.


# What to build

## 1. Ingest and standardize

* Load each raw file into DuckDB (read directly from Excel/XML).
* Normalize data types (timestamps, numerics, text), trim whitespace, and de-duplicate using reasonable business keys.
* Create a reproducible local database file (e.g., build/retail.duckdb).

## 2. Dimensional modeling

* dim_product (stock_code, description, first_seen, last_seen)
* dim_customer (customer_id, country)
* dim_calendar (date, is_weekend, is_uk_holiday, iso_week, month, year)
* fct_sales (invoice_no, invoice_ts, customer_id, stock_code, qty, unit_price_gbp, gross_amount_gbp)
    * Treat cancellations/returns correctly (net to zero when appropriate).
    * Ensure one row = one line item.

## 3. Currency enrichment (GBP → EUR)

* Build a daily FX table for GBP using the provided ECB files.
* Produce fct_sales_eur with unit_price_eur and gross_amount_eur.
    * Remember: ECB files give GBP per 1 EUR; therefore amount_eur = amount_gbp / rate_gbp_per_eur.


## 4. Produce an analytics table or view:

* agg_store_day (date, country, orders, items, net_qty, net_revenue_gbp, net_revenue_eur)
    * “Net” means after accounting for returns/cancellations.
    * Join to dim_calendar to include holiday/weekend flags.


# Deliverables

* Code in a small repo structure:

  ```
  README.md
  /src          # Python entrypoint(s) and helpers
  /sql          # Optional: views/materializations
  /data/raw     # provided inputs (keep file names as-is)
  /build        # created .duckdb and outputs (ok to .gitignore)
  ```

* Entrypoint: e.g. python src/run.py --rawdir data/raw --db build/retail.duckdb --rebuild
    * It should create all objects and final outputs with no manual steps.
    * A Makefile would be helpful!

* README.md covering:
    * How to run (commands), dependencies, and expected outputs.
    * Modeling choices (how you handled returns, keys, FX joins).
    * Data quality rules and how failures surface.
	* (Optional but nice) a short diagram of your tables, and a couple of example queries analysts could run.

## Tech Constraints

* Local only; no cloud services
* Python 3.13
* No heavy frameworks needed (dbt optional, but not expected)
* UV for package management

# Local Setup

Install the tooling below before running the project commands:

1. Install `uv` (fast Python package manager from Astral):
   * macOS/Linux: `curl -LsSf https://astral.sh/install.sh | sh`
   * Windows (PowerShell): `powershell -ExecutionPolicy Bypass -c "irm https://astral.sh/install.ps1 | iex"`
2. Install Git LFS so the large Excel workbook is pulled correctly:
   * macOS (Homebrew): `brew install git-lfs`
   * Windows: download the installer from https://git-lfs.com and run it
   * Linux: follow the distribution instructions at https://git-lfs.com
3. Run `git lfs install` once after installing Git LFS.
