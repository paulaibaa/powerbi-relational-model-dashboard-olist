# Data preparation and model documentation

This document summarises the Power Query transformations, the relationships of the
data model and the reasoning behind each decision.

Source: *Brazilian E-Commerce Public Dataset by Olist* (9 CSV files, about 100k orders
from 2016 to 2018). The raw data is not included in this repository.

---

## 1. Workflow

```
9 raw CSV files  ->  Power Query (cleaning, typing, merges)  ->  Data model (relationships)  ->  DAX measures  ->  Dashboard
```

General rules applied to every table:

| Rule | Why |
|---|---|
| Automatic type detection turned off; every column typed explicitly | Auto-detection converts zip codes to numbers and drops leading zeros |
| Identifiers and zip codes are **text** | 33% of seller zip codes start with `0`; as numbers they lose a digit and merges silently fail |
| Decimal columns typed with the **en-US** locale | The files use a dot as decimal separator; a Spanish locale would misread them |
| Exact-match replacement lists instead of chained "replace text" steps | Substring replacements depend on step order and can corrupt already-fixed values |
| Row counts checked after every merge | A merge on a non-unique key silently duplicates rows |

---

## 2. Model overview

| Table | Type | Grain (one row per...) | Key | Rows |
|---|---|---|---|---|
| `fact_orders` | Fact | order | `order_id` | 99,441 |
| `fact_order_items` | Fact | item (unit) of an order | `order_id` + `order_item_id` | 112,650 |
| `fact_order_payments` | Fact | payment of an order | `order_id` + `sequential` | 103,886 |
| `fact_order_reviews` | Fact | order (latest review only) | `order_id` | 98,673 |
| `dim_customers` | Dimension | order (see note below) | `id` | 99,441 |
| `dim_products` | Dimension | product | `product_id` | 32,951 |
| `dim_sellers` | Dimension | seller | `seller_id` | 3,095 |
| `dim_calendar` | Dimension | day | `date` | 1,096 |
| `dim_geolocation` | Helper (not loaded) | zip code | `zip_code_prefix` | 19,010 |

Note on customers: Olist generates a **new customer id for every order** (`id` in
`dim_customers`, `customer_id` in `fact_orders`). The person is identified by
`customer_unique_id` (96,096 people, 2,997 of them with more than one order).

---

## 3. Relationships

| # | One side | Many side | Cardinality | Why |
|---|---|---|---|---|
| 1 | `fact_orders[order_id]` | `fact_order_items[order_id]` | one-to-many | An order has one or more items |
| 2 | `fact_orders[order_id]` | `fact_order_payments[order_id]` | one-to-many | An order can be paid in several payments |
| 3 | `fact_orders[order_id]` | `fact_order_reviews[order_id]` | one-to-one | Only the latest review per order is kept |
| 4 | `fact_orders[customer_id]` | `dim_customers[id]` | one-to-one | One customer row per order |
| 5 | `dim_products[product_id]` | `fact_order_items[product_id]` | one-to-many | A product appears in many order items |
| 6 | `dim_sellers[seller_id]` | `fact_order_items[seller_id]` | one-to-many | A seller fulfils many order items |
| 7 | `dim_calendar[date]` | `fact_orders[purchase_date_fk]` | one-to-many | Time analysis by purchase date |


---

## 4. Power Query transformations by table

### `fact_orders` (source: `olist_orders_dataset.csv`)

| Step | Why |
|---|---|
| Renamed date columns to shorter names | Readability |
| Order status in proper case (`Delivered`, `Canceled`...) | DAX filters use these exact values |
| Date columns typed as date-time | Needed to compute delivery times |
| **`status_group`** column: `Completed` (Delivered), `In Progress` (Created, Approved, Invoiced, Processing, Shipped), `Not Completed` (Canceled, Unavailable) | 461 canceled orders still have priced items; the group lets measures exclude non-sales |
| **`purchase_date_fk`** (purchase date without time) | Foreign key to `dim_calendar`; the time part would prevent matching |
| Merged with the customers table on the per-order customer id to bring `customer_unique_id` | Lets the model count real people, not per-order ids |

### `fact_order_items` (source: `olist_order_items_dataset.csv`)

| Step | Why |
|---|---|
| `price` and `freight_value` typed as decimal (en-US) | Dot decimal separator |
| `order_item_id` as integer, ids as text, `shipping_limit_date` as date-time | Correct types |

Grain: one row per unit. There is no quantity column: buying the same product twice gives two rows.

### `fact_order_payments` (source: `olist_order_payments_dataset.csv`)

| Step | Why |
|---|---|
| Columns renamed (`value`, `installments`, `type_payment`, `sequential`) | Shorter names |
| Payment type formatted (`credit_card` -> `Credit Card`) | Readable labels in visuals |
| `value` as decimal (en-US); counters as integers | Correct types |

Grain: one row per payment. `sequential` numbers the payments of an order (2,961 orders have more than one).

### `fact_order_reviews` (source: `olist_order_reviews_dataset.csv`)

| Step | Why |
|---|---|
| Removed comment title and message | 88% of titles and 59% of messages are empty; text is not analysed |
| Kept **only the latest review per order** (group by `order_id`, keep the row with the maximum answer timestamp) | 551 orders had more than one review; `review_id` is not unique (814 repeats). 99,224 -> 98,673 rows; the average score is unchanged (4.086) |
| Both dates converted to date only | Simpler analysis in days |
| **`response_days`** = answer date - creation date | Time the customer takes to answer the survey |

The latest review is chosen with a group-by and a maximum.

### `dim_products` (source: `olist_products_dataset.csv` + category translation)

| Step | Why |
|---|---|
| Renamed columns; removed `product_name_lenght` and `product_description_lenght` | Fixes the source typo; the two columns are not used |
| Left-outer merge with the category translation table | Keeps all products; 610 have no category |
| Missing category -> `Unknown`; untranslated category keeps its original name | No blank categories |
| Underscores replaced by spaces and proper case | Readable labels |
| Single mapping list (exact match, one pass) to fix typos and merge near-duplicate categories (`Home Confort`, `Fashio Female Clothing`, `Costruction Tools`...) | About 73 raw categories reduced to about 45, so charts stay readable |
| photos_qty nulls | Kept as null |

### `dim_sellers` (source: `olist_sellers_dataset.csv`)

| Step | Why |
|---|---|
| Zip code kept as text | Leading zeros (1,027 of 3,095 sellers) |
| Trimmed and collapsed repeated spaces in `city` | Values like `sao  paulo` |
| Mapping list of 66 exact replacements applied to `city` | Typos (`garulhos`), state or country appended (`pinhais/pr`), non-city values (an email, a number). 611 -> 553 distinct cities, 82 rows changed |
| Coordinates (`latitude`, `longitude`) added with a merge on zip code | Enables map visuals without a second relationship path |

Known issue: 35 sellers have a state that does not match their zip code (mostly `SP`).
It was documented rather than fixed.

### `dim_customers` (source: `olist_customers_dataset.csv`)

| Step | Why |
|---|---|
| Zip code as text | Leading zeros (about 24% of customers) |
| Two typos corrected (`piumhii`, `santa barbara d oeste`) | Consistency with the other tables |
| Coordinates added with a merge on zip code | 279 rows have no match (0.3%) |
| **Not reduced to one row per person** | Each order keeps the address it was made with (about 250 people changed zip code) and it avoids extra steps. People are counted with `DISTINCTCOUNT` on `customer_unique_id` |

### `dim_geolocation` (helper query, not loaded into the model)

| Step | Why |
|---|---|
| City lowercased, accents removed, spaces collapsed | 8,011 spellings reduced to about 5,970 (`sao paulo` / `são paulo`) |
| Removed identical rows | 1,000,163 rows; many exact repeats |
| Removed coordinates outside Brazil | 33 rows |
| Grouped by zip code: mean latitude and longitude, most frequent city and state | A zip prefix is an area with many sample points; it must be unique to merge without duplicating rows. Result: 19,010 rows |

### `dim_calendar` (DAX calculated table)

- One row per day from 2016-01-01 to 2018-12-31.
- Columns: `year`, `quarter`, `year_quarter`, `month_number`, `month_name`, `month_short`,
  `year_month`, `weekday_number`, `weekday_name`, `day_type`. Month and day names use the
  `en-US` locale.
- Month and day names sorted by their numeric column; the table is marked as a date table.