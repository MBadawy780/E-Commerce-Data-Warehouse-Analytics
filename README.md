# Olist E-Commerce Data Warehouse & Analytics

A full end-to-end data engineering and business intelligence project built on the Brazilian **Olist** e-commerce dataset. Raw CSV files are ingested through a three-layer medallion architecture (Bronze → Silver → Gold) using SQL Server and SSIS, then visualized in a Power BI dashboard.

---

## Project Files

| File | Description |
|---|---|
| `Load_Bronze.dtsx` | SSIS package — ingest raw CSVs into the Bronze layer |
| `Load_Silver.dtsx` | SSIS package — clean and transform Bronze into Silver |
| `Load_Gold.dtsx` | SSIS package — build the star-schema Gold layer |
| `DWH.sql` | SQL Server DDL — creates the `DWH` database and all schemas/tables |
| `OLIST.pbix` | Power BI dashboard for business analytics |

---

## Architecture Overview

```
CSV Source Files
      │
      ▼
┌─────────────┐
│   BRONZE    │  Raw ingestion — full fidelity, all columns nvarchar
│  (staging)  │  + Ingestion_DateTime, Source_File metadata columns
└──────┬──────┘
       │  Load_Silver.dtsx
       ▼
┌─────────────┐
│   SILVER    │  Cleansed & typed — proper data types, PKs enforced
│ (cleansed)  │  Computed columns: delivery_time_days, delay_vs_estimated,
│             │  approval_time_hours, delivery_performance_status
│             │  SCD support on customers (Valid_From / Valid_To / Is_Current)
└──────┬──────┘
       │  Load_Gold.dtsx
       ▼
┌─────────────┐
│    GOLD     │  Star schema — dimensions + fact table, ready for BI
│  (serving)  │  + Aggregate tables: agg_customer_360, agg_seller_performance
└─────────────┘
       │
       ▼
  OLIST.pbix  (Power BI)
```

---

## Data Sources

Nine CSV files sourced from the public [Olist dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce):

| CSV File | Description |
|---|---|
| `olist_customers_dataset.csv` | Customer identifiers and location |
| `olist_geolocation_dataset.csv` | Zip code → lat/lng lookup |
| `olist_orders_dataset.csv` | Order lifecycle timestamps and status |
| `olist_order_items_dataset.csv` | Line items per order (product, seller, price, freight) |
| `olist_order_payments_dataset.csv` | Payment method, installments, value |
| `olist_order_reviews_dataset.csv` | Customer review scores and comments |
| `olist_products_dataset.csv` | Product attributes and category |
| `olist_sellers_dataset.csv` | Seller location |
| `product_category_name_translation_dataset.csv` | Portuguese → English category names |

Place all CSV files in a `Source/` folder before running the packages (default path: `C:\...\Source\`).

---

## Prerequisites

- **SQL Server** (Express or higher, version 2019 / compatibility level 150)
- **SQL Server Integration Services (SSIS)**
- **SQL Server Data Tools (SSDT)** or Visual Studio with SSIS extension — to open/run `.dtsx` packages
- **Power BI Desktop** — to open `OLIST.pbix`

---

## Setup & Execution

### 1. Create the database

Run `DWH.sql` against your SQL Server instance. This creates:
- The `DWH` database
- Three schemas: `bronze`, `silver`, `gold`
- All tables in each layer (see schema details below)

```sql
-- In SSMS or sqlcmd:
-- Open DWH.sql and execute
```

### 2. Update connection strings

In each `.dtsx` package, update the flat-file connection managers to point to your local `Source/` folder, and update the OLE DB connection to match your SQL Server instance name.

### 3. Run SSIS packages in order

```
1. Load_Bronze.dtsx   ← ingest CSVs into bronze schema
2. Load_Silver.dtsx   ← clean and transform into silver schema
3. Load_Gold.dtsx     ← build star schema in gold schema
```

Each package can be executed from SSDT, Visual Studio, or via `dtexec`:

```bash
dtexec /File Load_Bronze.dtsx
dtexec /File Load_Silver.dtsx
dtexec /File Load_Gold.dtsx
```

### 4. Open the dashboard

Open `OLIST.pbix` in Power BI Desktop and update the data source connection to point to your SQL Server instance.

---

## Database Schema — Three Layers

### Bronze (Raw Ingestion)

All columns stored as `nvarchar` to preserve raw values exactly as they arrive. Each table includes:
- `Ingestion_DateTime` — when the row was loaded
- `Source_File` — which CSV file it came from
- `Hash_code` — on `olist_orders` for change detection

| Table | Key Columns |
|---|---|
| `bronze.olist_customers` | customer_id, customer_unique_id, city, state, zip |
| `bronze.olist_geolocation` | zip_code_prefix, lat, lng, city, state |
| `bronze.olist_orders` | order_id, customer_id, status, 5× timestamp columns |
| `bronze.olist_order_items` | order_id, order_item_id, product_id, seller_id, price, freight |
| `bronze.olist_order_payments` | order_id, payment_type, installments, value |
| `bronze.olist_order_reviews` | review_id, order_id, score, comment, timestamps |
| `bronze.olist_products` | product_id, category, dimensions (weight, length, height, width) |
| `bronze.olist_sellers` | seller_id, zip, city, state |
| `bronze.product_category_name_translation` | Portuguese name, English name |

### Silver (Cleansed & Typed)

Proper data types enforced, primary keys added, string columns trimmed. Notable additions:

- **`silver.orders`** — four computed columns derived at query time:
  - `delivery_time_days` — days from purchase to delivery
  - `delay_vs_estimated` — days late/early vs estimated delivery
  - `approval_time_hours` — hours from purchase to approval
  - `delivery_performance_status` — `'On Time'` / `'Late'` / `'In Progress'`

- **`silver.customers`** — SCD Type 2 columns: `Valid_From`, `Valid_To`, `Is_Current`

| Table | Primary Key |
|---|---|
| `silver.customers` | customer_id |
| `silver.geolocation` | geolocation_zip_code_prefix |
| `silver.orders` | order_id |
| `silver.order_items` | (order_id, order_item_id) |
| `silver.payments` | (order_id, payment_sequential) |
| `silver.reviews` | review_id |
| `silver.products` | product_id |
| `silver.sellers` | seller_id |
| `silver.product_category_name_translation` | product_category_name |

### Gold (Star Schema — BI-ready)

#### Fact Table: `gold.fact_order_items`

One row per order line item, fully denormalized for analytics.

| Column Group | Columns |
|---|---|
| Keys | order_id, order_item_id, customer_id, product_id, seller_id |
| Date FKs | purchase_date_id, approved_date_id, delivered_carrier_date_id, delivered_customer_date_id, estimated_delivery_date_id |
| Order status | order_status, delivery_performance_status |
| Delivery metrics | delivery_time_days, delay_vs_estimated, approval_time_hours |
| Financials | price, freight_value, total_item_value, total_items_value, total_freight_value, total_order_value, total_payment_value, payment_installments |
| Review | review_score, sentiment, review_response_hours |
| Product | product_category_name, category_english |

#### Dimension Tables

| Table | Columns |
|---|---|
| `gold.dim_customers` | customer_id, customer_unique_id, city, state, zip, lat, lng |
| `gold.dim_sellers` | seller_id, zip, city, state, lat, lng |
| `gold.dim_products` | product_id, category (PT + EN), weight, dimensions, volume_cm3 |
| `gold.dim_date` | date_id (YYYYMMDD), full_date, year, quarter, month, month_name, week_of_year, day_of_month, day_name, is_weekend |

#### Aggregate Tables (optional / commented out in DDL)

| Table | Description |
|---|---|
| `gold.agg_customer_360` | Per-customer: total orders, total spent, avg order value, lifespan days, avg review score |
| `gold.agg_seller_performance` | Per-seller: total orders, items sold, revenue, freight, avg review score, avg delivery days |

---

## Dashboard

`OLIST.pbix` connects to the `gold` schema and provides interactive analysis across:

- **Sales & Revenue** — order volumes, GMV trends, payment methods
- **Delivery Performance** — on-time rates, average delivery times, delay distribution
- **Customer Analytics** — geographic distribution, repeat purchase behavior
- **Seller Performance** — revenue ranking, review scores, delivery speed
- **Product Categories** — top categories by volume and revenue
- **Review Sentiment** — score distributions and response times
