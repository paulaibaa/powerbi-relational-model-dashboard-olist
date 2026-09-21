# DAX measures and calculated columns

This document explains the main measures of the report, what they mean and how they are
calculated. All measures live in a table called `dax`. Values in the "Value" columns are for the
whole period, with no slicers applied.

## Rules that apply to most measures

| Rule | Why |
|---|---|
| Sales measures exclude orders whose `status_group` is **Not Completed** (canceled and unavailable) | Canceled orders still have priced items; they should not count as sales |
| Ratios use `DIVIDE` | It returns blank instead of an error when the denominator is 0 |
| Percent-of-total measures use `REMOVEFILTERS` on the axis column | The denominator is the total of all categories, states or bands, not just the visible ones |
| Order counts are taken from `fact_order_items` | Filters from products or sellers reach items but not `fact_orders`, so counting orders in items keeps them consistent with revenue |
| Delivery measures only consider delivered orders | Orders still in transit have no delivery date |

`status_group` groups the order status: **Completed** (delivered), **In Progress** (created,
approved, invoiced, processing, shipped) and **Not Completed** (canceled, unavailable).

## Sales and orders

| Measure | Meaning | Value |
|---|---|---|
| `revenue` | Item prices of orders that count as sales (freight not included) | R$ 13,494,400.74 |
| `total_orders` | Distinct orders with at least one item, excluding Not Completed | 98,199 |
| `orders` | Distinct orders with at least one item, any status (used in the status donut) | 98,666 |
| `orders_completed`, `orders_inprogress`, `orders_notcompleted` | Orders with items in each status group | 96,478 / 1,721 / 467 |
| `items_sold` | Number of items sold (each row of `fact_order_items` is one unit) | 112,101 |
| `avg_order_value` | Revenue per order | R$ 137.42 |
| `avg_item_price` | Revenue per item | R$ 120.38 |
| `items_per_order` | Basket size | 1.14 |
| `freight` | Shipping cost paid by the customer on the items sold | R$ 2,241,126.29 |
| `freight_pct` | Freight as a share of revenue | 16.61% |
| `revenue_pct_category` | Share of total revenue of each product category | 9.31% for Health & Beauty |
| `revenue_pct_state` | Share of total revenue of each customer state | 38.27% for SP |

```dax
revenue =
CALCULATE (
    SUM ( fact_order_items[price] ),
    fact_orders[status_group] <> "Not Completed"
)

total_orders =
CALCULATE (
    DISTINCTCOUNT ( fact_order_items[order_id] ),
    fact_orders[status_group] <> "Not Completed"
)

avg_order_value = DIVIDE ( [revenue], [total_orders] )

revenue_pct_category =
DIVIDE (
    [revenue],
    CALCULATE ( [revenue], REMOVEFILTERS ( dim_products[category] ) )
)
```

## Customers

Olist creates a new customer id for every order, so people are counted with `customer_unique_id`.

| Measure | Meaning | Value |
|---|---|---|
| `customers` | Distinct people with a valid order | 94,990 |
| `avg_customer_spend` | Revenue per customer | R$ 142.06 |
| `orders_per_customer` | Valid orders per customer | 1.03 |
| `repeat_customers` | People with more than one valid order | 2,888 |
| `repeat_customers_pct` | Share of customers who repeat | 3.04% |
| `revenue_pct_customer_type` | Share of revenue of one-time vs repeat customers | 94.44% / 5.56% |
| `customers_pct_spend_band` | Share of customers in each spend band | 3.94% above R$ 500 |
| `revenue_pct_spend_band` | Share of revenue of each spend band | 25.68% above R$ 500 |

```dax
customers =
CALCULATE (
    DISTINCTCOUNT ( dim_customers[customer_unique_id] ),
    fact_orders[status_group] <> "Not Completed"
)

repeat_customers =
COUNTROWS (
    FILTER (
        VALUES ( dim_customers[customer_unique_id] ),
        CALCULATE ( DISTINCTCOUNT ( fact_orders[order_id] ), fact_orders[status_group] <> "Not Completed" ) > 1
    )
)
```

`repeat_customers` loops over each person and counts their valid orders. Inside `FILTER`,
`CALCULATE` turns the current person into a filter (context transition), so the count is per person.

## Delivery

A delivery is **late** when the delivery date is after the estimated date, compared by day
(the estimated date has no time). The delivery columns are created in Power Query on `fact_orders`.

| Measure | Meaning | Value |
|---|---|---|
| `avg_delivery_days` | Days from purchase to delivery | 12.56 |
| `avg_promised_days` | Days from purchase to the estimated date | 23.74 |
| `delivered_orders_days` | Delivered orders with a delivery date | 96,470 |
| `late_orders` | Delivered orders that arrived after the estimated date | 6,534 |
| `late_delivery_pct` | Late orders over delivered orders | 6.77% |
| `avg_handling_days` | Days from purchase until the seller hands the order to the carrier | 3.2 |
| `avg_shipping_days` | Days from carrier pick-up to the customer | 9.3 |
| `delivered_items` | Items in delivered orders | 110,189 |
| `late_items_pct` | Late items over delivered items (works by seller or route) | 6.59% |
| `avg_delivery_days_items` | Delivery days averaged by item (works by seller or route) | 12.47 |

```dax
late_delivery_pct = DIVIDE ( [late_orders], [delivered_orders_days] )

late_items_pct =
DIVIDE (
    CALCULATE ( COUNTROWS ( fact_order_items ), fact_orders[delivery_status] = "Late" ),
    [delivered_items]
)

avg_delivery_days_items = AVERAGEX ( fact_order_items, RELATED ( fact_orders[delivery_days] ) )
```

The `_items` versions exist because a seller filter reaches order items but not `fact_orders`.

## Satisfaction

| Measure | Meaning | Value |
|---|---|---|
| `avg_review_score` | Average review score (one review per order, the latest) | 4.09 |
| `low_score_pct` | Share of reviews with 1 or 2 stars | 14.69% |
| `low_score_pct_late` | Same, only for late orders | 62.42% |
| `low_score_pct_on_time` | Same, only for on-time orders | 9.27% |
| `avg_response_days` | Days between the survey being sent and answered | 2.6 |

```dax
low_score_pct =
DIVIDE (
    CALCULATE ( COUNTROWS ( fact_order_reviews ), fact_order_reviews[review_score] <= 2 ),
    COUNTROWS ( fact_order_reviews )
)

low_score_pct_late = CALCULATE ( [low_score_pct], fact_orders[delivery_status] = "Late" )
```

## Sellers and products

| Measure | Meaning | Value |
|---|---|---|
| `active_sellers` | Sellers with at least one valid sale | 3,053 |
| `avg_revenue_per_seller` | Revenue per active seller | R$ 4,420 |
| `seller_under_1k_pct` | Share of active sellers with less than R$ 1,000 of revenue | 53.68% |
| `top_10pct_sellers_share` | Share of revenue of the 10% of sellers who sell the most | 67.50% |
| `products_sold` | Distinct products sold | 32,729 |
| `items_per_product` | Items sold per product | 3.43 |
| `products_sold_once_pct` | Share of products sold a single time | 54.95% |

```dax
top_10pct_sellers_share =
VAR seller_with_sales = FILTER ( dim_sellers, [revenue] > 0 )
VAR n = ROUNDUP ( 0.1 * COUNTROWS ( seller_with_sales ), 0 )
VAR top_sellers = TOPN ( n, seller_with_sales, [revenue] )
RETURN
    DIVIDE ( SUMX ( top_sellers, [revenue] ), [revenue] )
```

The measure keeps the sellers with sales (3,053), takes the top 10% (306) by revenue and divides
their revenue by the total.

## Payments

Payments are order-level and include amounts that are not item prices (freight, vouchers), so
`payment_value_total` (R$ 16.01M) is not comparable with `revenue` (R$ 13.49M).

| Measure | Meaning | Value |
|---|---|---|
| `payment_value_total` | Sum of all payments | R$ 16,008,872.12 |
| `payment_share` | Share of the total paid by each payment method | 78.34% credit card |
| `credit_card_share` | Credit card share of the total paid | 78.34% |
| `avg_installments` | Average installments of credit card payments | 3.51 |
| `card_value_share_by_installments` | Share of card value paid in each number of installments | 19.46% in 1, 17.63% in 10 |

```dax
credit_card_share =
DIVIDE (
    CALCULATE ( [payment_value_total], fact_order_payments[type_payment] = "Credit Card" ),
    [payment_value_total]
)
```

## Calculated columns and tables

| Object | Table | Definition |
|---|---|---|
| `dim_calendar` | (table) | `CALENDAR ( DATE ( 2016, 1, 1 ), DATE ( 2018, 12, 31 ) )` with `year`, `month`, `month_name`, `day`, `quarter`, `year_quarter`, `weekday_number` and `day_type` |
| `delay_bucket` | `fact_orders` | Days late or early against the estimated date, in six ranges from "Early 8+ days" to "Late 8+ days" |
| `route` | `fact_order_items` | "Same state" if seller and customer are in the same state, otherwise "Different state" |
| `customer_orders` | `dim_customers` | Valid orders of the person |
| `customer_type` | `dim_customers` | "Repeat" if `customer_orders` is above 1, otherwise "One-time" |
| `customer_spend` | `dim_customers` | Revenue of the person |
| `spend_band` | `dim_customers` | Five ranges of `customer_spend`, below R$ 50 to above R$ 500 |
| `payment_band` | `fact_order_payments` | Five ranges of payment value, below R$ 50 to above R$ 500 |

Calculated columns are computed when the data is refreshed, so they do not respond to slicers.
A customer who bought in 2017 and 2018 is a "Repeat" customer even when the year slicer shows 2018.

```dax
customer_orders =
CALCULATE (
    DISTINCTCOUNT ( fact_orders[order_id] ),
    ALLEXCEPT ( dim_customers, dim_customers[customer_unique_id] ),
    fact_orders[status_group] <> "Not Completed"
)

route =
VAR customer_state =
    LOOKUPVALUE ( dim_customers[state], dim_customers[customer_id], RELATED ( fact_orders[customer_id] ) )
RETURN
    IF ( RELATED ( dim_sellers[state] ) = customer_state, "Same state", "Different state" )
```

`ALLEXCEPT` removes every filter on `dim_customers` except the person, so the count covers all the
orders of that person and not only the order in the current row.
