# Ecommerce Sales and Returns: Databricks to Power BI

An end-to-end analytics project for an ecommerce business. Data is ingested and transformed in Databricks, served as gold-layer tables, and modelled and reported in Power BI. It covers sales, products, customers, shipments and returns, with currency-aware revenue and time-intelligence measures.

## Contents

- [Overview](#overview)
- [Pipeline](#pipeline)
- [Data](#data)
- [Data model](#data-model)
- [Measures](#measures)
- [Report pages](#report-pages)
- [Key findings](#key-findings)
- [Design decisions and data notes](#design-decisions-and-data-notes)
- [How to open this project](#how-to-open-this-project)

## Overview

The model turns six gold-layer tables into a single semantic model that answers four questions:

1. How is the business performing (revenue, orders, average order value)?
2. Which products and categories drive revenue?
3. Where do returns and fulfilment problems occur?
4. Who buys, and through which channel?

**Tools:** Azure Data Lake Storage Gen2, Databricks (ingestion and transformation), Power BI Desktop, DAX, Power Query (M).

## Pipeline

```
ADLS Gen2 ──▶ Databricks ingestion ──▶ Databricks transformations ──▶ Gold tables ──▶ Power BI ──▶ Report
(raw files)                                                        (ecommerce.gold)   (Import mode)
```

1. **Ingest:** raw ecommerce files are read from Azure Data Lake Storage Gen2 (ADLS Gen2) into Databricks. The tables keep ingestion metadata (`ingest_timestamp`, `_ingested_at`, `_source_file`, `file_name`, `processed_time`) from this step.
2. **Transform:** the data is cleaned and reshaped in Databricks into analysis-ready gold tables (`gld_` prefix) in the `gold` schema of the `ecommerce` catalog.
3. **Serve:** Power BI Desktop connects to the Databricks SQL warehouse and loads the gold tables in Import mode.
4. **Model and report:** relationships, DAX measures and the report pages are built in Power BI on top of those tables.

> **To complete:** add how ingestion runs (for example notebooks, Auto Loader or jobs), what transformations were applied (for example deduplication, type casting, joins), whether bronze and silver layers sit before gold, and any scheduling. These details are not in the Power BI model, so they need to come from you.

## Data

Source: the `gold` schema of the `ecommerce` catalog in Databricks, loaded in Import mode.

| Table | Role | Approx. rows |
|---|---|---|
| `gld_order_items` | Fact: one row per order line | 1.19M |
| `gld_dim_products` | Dimension: product, category, brand | 50,000 |
| `gld_dim_customers` | Dimension: customer, region, country, state | 299,700 |
| `gld_dim_date` | Dimension: calendar (2024-01-01 to 2025-12-31) | 730 |
| `gld_order_shipments` | Shipment per order, with carrier | 679,731 |
| `gld_order_returns` | Return per order, with reason | 20,194 |
| `dim_currency` | Static USD conversion rates (added in the model) | 7 |

Orders span **Jan 2024 to Aug 2025** (679,731 orders, 7 currencies).

## Data model

```
gld_dim_date ──────┐
gld_dim_products ──┤
gld_dim_customers ─┼──▶ gld_order_items ◀── dim_currency
                   │            │
                   │            ▼
                   │     gld_order_shipments ◀── gld_order_returns
```

- Star schema with `gld_order_items` as the fact table.
- All relationships are many-to-one with single-direction filtering.
- `gld_dim_date` is marked as the date table, and `month_name` sorts by a hidden `month_number` column.
- Technical columns (ingest timestamps, file names, keys) are hidden, and numeric columns are set to not auto-summarize, so reports use measures.
- All measures live in a dedicated `_Measures` table, grouped into display folders.

## Measures

| Folder | Measures |
|---|---|
| Volume | Orders, Order Lines, Units Sold, Customers, Units per Order, Orders per Customer |
| Pricing | Average Discount % |
| Revenue (USD) | Gross Revenue, Net Revenue, Discount Amount, Average Order Value |
| Revenue (Local Currency) | Gross Revenue, Net Revenue, Discount Amount, Average Order Value, Currencies in Scope |
| Returns | Returns, Returned Orders, Return Rate, Return Rate (Complete Months), Share of Returns % |
| Time Intelligence | Orders YTD / PY / YoY %, Net Revenue PY / YoY % (USD and local) |

Net revenue is `quantity × unit_price × (1 − discount_pct / 100)`.

## Report pages

Four pages, each answering one question:

| Page | Question | Main visuals |
|---|---|---|
| Executive Overview | How is the business doing? | KPI cards, monthly revenue by year, revenue by category, orders by channel |
| Product and Category | What drives revenue? | Orders vs revenue scatter, category table, top brands |
| Returns and Fulfilment | Where do things go wrong? | Return rate by month, returns by reason, return rate by carrier, category-by-reason matrix |
| Customers and Channels | Who buys, and how? | Revenue and customers by region, channel mix by month, channel summary |

## Key findings

- **Electronics earns about 76% of net revenue** (about $165M of $217.6M), although Home & Kitchen has the most orders.
- **Order value is flat** at about $320 across months and across Mobile and Website, so revenue moves with order volume.
- **Return rate is about 3%** and is nearly identical across categories and carriers.
- **Damaged and Wrong Item account for about half of all returns.**
- **Mobile accounts for about 55% of orders.**

## Design decisions and data notes

- **Multi-currency prices.** `unit_price` is stored in seven currencies (INR, USD, GBP, AED, SGD, CAD, AUD), so summing it directly is meaningless. The local-currency revenue measures return a value only when one currency is in scope. The USD measures convert through `dim_currency`.
- **Static exchange rates.** The rates are ECB reference rates for 23 Oct 2026, crossed through EUR (AED uses its fixed peg of 3.6725 per USD). One spot rate is applied to every period, which suits comparisons across categories and channels but is not true historical revenue. A date-based rate table would be the next step.
- **Incomplete latest month.** Returns for recent orders are not fully logged, so the return rate for the last month looks artificially low. `Return Rate (Complete Months)` blanks that month for trend charts.
- **Returns and category.** Returns relate to shipments, so a filter on category does not reach the returns table directly. `Returned Orders` uses `TREATAS` on order ID to slice returns by category, product and date correctly. Use it instead of `Returns` when splitting by those fields.
- **Discount scale.** `discount_pct` is stored in percentage points (0 to 44), so measures divide it by 100.
- **Partial 2025.** 2025 has 8 months of orders, so year-over-year comparisons for 2025 are not like-for-like.

