# Olist E-commerce: Power BI Relational Data Model & Dashboard

Personal portfolio project: building a relational data model in Power BI from
raw CSV files, with data cleaning in Power Query and measures in DAX.

## Goal

Model a Brazilian e-commerce dataset (orders, items, payments, reviews,
customers, sellers, products), clean it in Power Query and build a dashboard
with DAX measures.

## Data source
Olist, & Sionek, A. (2018). *Brazilian E-Commerce Public Dataset by Olist* 
The data is **not included** in this repository.  Download the Brazilian E-Commerce Public Dataset by Olist from Kaggle
[Data set]. Kaggle. https://doi.org/10.34740/kaggle/dsv/195341
and place the CSV files in `data/raw/`

## Data model

| Table | Type | Grain (one row per...) | Key |
|---|---|---|---|
| fact_order_items | Fact | item of an order | order_id + order_item_id |
| fact_order_payments | Fact | payment of an order | order_id + sequential |
| fact_order_reviews | Fact | order (latest review) | order_id |
| fact_orders | Fact | order | order_id |
| dim_customers | Dimension | order (Olist creates a new customer id per order) | id |
| dim_products | Dimension | product | product_id |
| dim_sellers | Dimension | seller | seller_id |
| dim_calendar | Dimension | day | date |

## Key modelling decisions

- The original `customer_id` changes with every order; `customer_unique_id`
  identifies the person and is used to count customers.
- Only the latest review of each order is kept.
- Orders are grouped by status: *Completed*, *In Progress*, *Not Completed*.
  Revenue excludes *Not Completed* orders.
- Product categories are translated to English and consolidated.
- Seller cities are cleaned with a mapping table; geolocation is reduced to one
  row per zip code and only used to add coordinates.

See [data preparation and model documentation](docs/data-preparation.md).

## Status

Data model and cleaning completed. DAX measures and dashboard in progress.
