# Overview

You’ll build a small but realistic data pipeline around retail transactions. The goal is to ingest raw files, model clean analytics tables, enrich with FX rates, and produce reliable daily metrics—then explain your design choices. Everything runs locally with DuckDB + Python.

Estimated effort: ~6 hours. You’ll have up to 10 calendar days to submit. You are encouraged to use AI tools in this assignment, but be prepared to explain all the code that the models generate.

# Data you’ll receive

```
data/
  raw/
    online_retail_II.xlsx             # e-commerce transactions (Excel; Use both tabs from the excel)
    gbp.xml                           # ECB XML with EUR-base rates including GBP
    ukbankholidays-jul19.xls          # UK public holidays
```

> The retail file contains columns like InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country. Cancellations appear as negative quantities and/or InvoiceNo that starts with “C”. FX files are EUR-base reference rates (1 EUR = X units of currency, so if OBS_VALUE=0.70, then 10 GBP -> 10/0.70 = 14.29 EUR).

If you have trouble downloading the files you can find the Online Retail II dataset [here](https://archive.ics.uci.edu/dataset/502/online+retail+ii), the gbp.xml [here](https://www.ecb.europa.eu/stats/policy_and_exchange_rates/euro_reference_exchange_rates/html/eurofxref-graph-gbp.en.html) and the historical UK bank holidays [here](https://www.dmo.gov.uk/media/bfknrcrn/ukbankholidays-jul19.xls).

## Metric definitions

We'll help you by defining some core metrics:

* orders = count (distinct invoice_no) where invoice _no NOT LIKE 'C%'
* items = count(*) of line items (post-dedupe)
* net_qty = sum(qty) (returns negative)
* net_revenue_gbp = sum(qty * unit_price_gbp)
* net_revenue_eur = sum(qty * unit_price_eur) with EUR computed via forward-filled daily ECB GBP rate.

## FX join policy

Build a daily calendar of FX rates (Online Retail II covers days from 2008 to 2011); ECB rates are missing on weekends/holidays; forward-fill the most recent available rate.

# What to build

## 1. Ingest and standardize

* Load each raw file into DuckDB (parsing via Python and then loading to DuckDB is acceptable).
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
* `uv` for package management; you're free to use any packages you want, we've installed Pandas and DuckDB as starters.

## Submission

Submit by creating a private GitHub repo and inviting @robertsiipola and @jisaksen (read access, email robert.siipola@mustigroup.com once you've done this). Alternatively, zip your repo (without build/ and data/) and email a link.

## Evaluation rubric (what "good" looks like)

This a rough weighting (%) we're using for evaluating the assignment:

* Reproducibility (entrypoint, clear README): 25
* Modeling correctness (returns, 1 row/line-item, keys): 25
* Data quality rules (nulls, duplicates, types): 15
* Code quality (formatting, structure, logging): 20
* Communicating (reasoning in README, tradeoffs): 15


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

## Python Environment

Use UV to manage the virtual environment and project dependencies:

1. Create the local virtualenv (writes to `.venv/` by default):
   * `uv venv`
2. Activate it when working locally:
   * macOS/Linux: `source .venv/bin/activate`
   * Windows (PowerShell): `.venv\Scripts\Activate.ps1`
3. Install or update dependencies from `pyproject.toml` and `uv.lock`:
   * `uv sync`
4. If you add packages, run `uv add <package>` so the lockfile stays current.

## Running the Pipeline

The repository ships with a minimal CLI stub so applicants have a starting point:

```
python src/run.py --rawdir data/raw --db build/retail.duckdb --rebuild
```

Feel free to replace this script, introduce your own CLI framework, or refactor the orchestration modules as part of your submission.
