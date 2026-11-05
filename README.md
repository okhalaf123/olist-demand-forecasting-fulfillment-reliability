# -olist-demand-forecasting-fulfillment-reliability

## Data Structure & Initial Checks

The original dataset contains **nine tables** and about **1.55 million records** from Olist, a Brazilian online marketplace, covering orders placed between **September 2016 and October 2018**. Each order is a purchase made by one customer from one or more independent sellers. The analysis starts with the nine raw tables, rebuilds them in Power Query into one flat order-item table for analysis, and uses a few small summary tables for forecasting, delivery reliability, and customer experience.

The dataset does **not** include inventory levels, supplier lead times, or product costs. Because of this, the project measures *delivery* reliability (seller to customer) and uses item revenue as a profit proxy. It does not measure procurement lead time or true safety stock.

### Database Schema

![Olist Database Schema](images/olist_schema.png)

### Raw Data Structure

| Table | One row represents | Rows | Key fields |
|---|---|---|---|
| `olist_orders_dataset` | One order | 99,441 | `order_id`, `customer_id`, `order_status`, purchase, approval, carrier, delivery, and estimated delivery timestamps |
| `olist_order_items_dataset` | One item within an order | 112,650 | `order_id`, `order_item_id`, `product_id`, `seller_id`, `price`, `freight_value` |
| `olist_order_payments_dataset` | One payment record for an order | 103,886 | `order_id`, `payment_type`, `payment_installments`, `payment_value` |
| `olist_order_reviews_dataset` | One review record | 99,224 | `review_id`, `order_id`, `review_score`, `review_comment_title`, `review_comment_message` |
| `olist_products_dataset` | One product | 32,951 | `product_id`, `product_category_name`, `product_weight_g` |
| `olist_customers_dataset` | One customer ID (a new ID per order) | 99,441 | `customer_id`, `customer_unique_id`, zip prefix, city, state |
| `olist_sellers_dataset` | One seller | 3,095 | `seller_id`, zip prefix, city, state |
| `olist_geolocation_dataset` | One coordinate point for a zip prefix | 1,000,163 | zip prefix, latitude, longitude, city, state |
| `product_category_name_translation` | One Portuguese category name | 71 | Portuguese name, English name |

The `orders` table is the hub. It connects to customers through `customer_id`, to items, payments, and reviews through `order_id`, and items connect to products and sellers. Zip prefixes in the customers, sellers, and geolocation tables connect locations to coordinates.

### Order Status

| Status | Meaning | Treatment |
|---|---|---|
| `delivered` | Order reached the customer | Counted as demand. The only status eligible for delivery analysis |
| `shipped` | Order is with the carrier | Counted as demand |
| `invoiced` | Order invoiced, not yet shipped | Counted as demand |
| `processing` | Order is being prepared (301 orders) | Counted as demand |
| `approved` | Payment approved (2 orders) | Counted as demand |
| `created` | Order created, not yet approved | Counted as demand |
| `canceled` | Order canceled | Excluded from demand |
| `unavailable` | Items unavailable | Excluded from demand |

### Analysis Tables

The project uses the following tables and queries, all rebuilt from the raw CSV files when the workbook is refreshed:

| Table | Purpose |
|---|---|
| `fact_order_items` | Final flat table, one row per order item, with all joins, flags, and calculated fields |
| `order_review_text` | Review title and comment text for orders with a comment, used for sentiment analysis |
| `stg_orders` | Order timestamps, status flags, delivery times, and delay fields |
| `stg_payments_order` | Payments reduced to one row per order |
| `stg_reviews_pick` / `stg_reviews_order` | Reviews cleaned to one valid review per order |
| `stg_products` | Products with English category names |
| `stg_customers` / `stg_sellers` | Customers and sellers with 5-character zip codes and coordinates |
| `stg_geo_zip` | Geolocation reduced to one row per zip prefix |
| `lst_tier1` | The 17 product categories selected for demand forecasting |
| `params` | Analysis window and coordinate limits used by the cleaning rules |

The SQL checks used to profile the raw data and to validate the cleaned table are in the project repository: [Phase 0 profiling queries](olist_phase0_profiling.sql) and [Phase 1 validation queries](olist_phase1_validation.sql).

### Initial Data Checks

Several profiling checks were completed in SQL before cleaning and modeling the data.

**Row counts and import validation:**  
Row counts were compared with the published counts for each file to confirm the import was complete. Review comments contain line breaks, which are a common cause of broken imports, so the reviews table was checked first. Zip code prefixes were imported as text, because reading them as numbers drops leading zeros and breaks joins between tables.

**Date range check:**  
Orders span **September 2016 to October 2018**. The earliest months of 2016 and the last months of 2018 contain very few orders, so the demand analysis uses **January 2017 to August 2018**, which gives 20 complete months.

**Order status check:**  
Most orders are `delivered`. Orders with `canceled` or `unavailable` status are excluded from demand analysis. The `processing` (301) and `approved` (2) orders are kept because they are real purchases that had not shipped yet.

**Delivery timestamp validation:**  
Only `delivered` orders can have a customer delivery date. Among delivered orders with complete timestamps, delivery time from purchase to customer averaged **12.56 days**, with a median of **10.22 days**, a 95th percentile of **29.27 days**, and a maximum of **209.63 days**. Because the mean is higher than the median and the tail is long, delivery times are **right-skewed**. This is why the delivery buffer uses an empirical percentile rather than a mean plus standard deviations. Orders with missing timestamps, or with a carrier or delivery date earlier than the purchase date, are flagged and excluded from delivery analysis but kept for demand analysis. **[X – check 1.11]** orders had timestamp anomalies.

**Missing values:**  
Missing values were found in the order timestamps, review comments, and product category.

- Review comment titles are missing for about **88%** of reviews and comment messages for about **58–59%**. A missing comment mostly means the customer left a star rating without text, not that no review exists.
- Missing `product_category_name` values were labeled `unknown`. In the demand window these items account for **1,449 orders** and about **1.33%** of revenue.
- Blank text values were standardized as nulls before cleaning.

**Review duplicate checks:**  
There were **[X – check 4.2]** exact duplicate review rows, and all duplicate groups were identical. These were collapsed to one row. **[X – check 4.5]** orders had more than one review, and the most recent answer was kept.

A further **789 `review_id` values** were shared by more than one order. In every case the orders belonged to the same customer, and 748 of them were placed within an hour of each other. These look like one review copied across several orders, so they were treated as errors: they are excluded from review metrics, but the orders are kept and labeled `invalid_shared_id` so they are not mistaken for orders with no review.

**Category coverage and translation:**  
Products fall into **73 product categories**, and the translation file covers **71** of them. Two categories had no English name and were translated manually: `pc_gamer` became `gaming_pc` and `portateis_cozinha_e_preparadores_de_alimentos` became `kitchen_appliances_and_food_prep`. All categories are kept in the data. Only the categories selected below are forecast.

**Forecast category selection:**  
Categories were tested on three criteria over the 20-month window: active in every month, at least 30 orders per month on average, and within the first 80% of cumulative revenue (revenue is used as a profit proxy because item price is the only value available and freight is excluded). **17 categories** met all three and cover about **79.6%** of window revenue, so they are the categories forecast in this project. Two points are worth noting:

- `computers` ranks 17th by revenue but exists in only **13 of the 20 months**, so it was reported but not forecast.
- `pet_shop` sits just under the 80% revenue line (about 79.7% cumulative before it) and is kept because it was active in all 20 months with about 85 orders per month.

The list is frozen on purpose so forecasts do not change category lists when data is refreshed.

**Location and geolocation checks:**  
The geolocation table has about **1 million rows** but only one location is needed per zip prefix. **[X – check 8.2]** exact duplicate rows were removed, and **[X – check 8.3]** points outside Brazil were removed as bad coordinates. The coordinate limits were widened to keep Fernando de Noronha (zip prefix 53990), which lies far east of the mainland. The remaining points were reduced to one row per zip prefix using median latitude and longitude. **[X – checks 8.7/8.8]** customer and seller zip prefixes had no usable coordinates, and these were left blank rather than estimated.

**Price, freight, and payment checks:**  
Zero freight values were kept because free shipping is real. Zero-value payments were kept because they can represent vouchers, while payment records with zero installments were treated as missing installment information. **[X – checks 2.3/2.10/3.3]** records were affected.

**Join integrity:**  
**[X – section 10 orphan checks]**. Orders without any items were not carried into the flat table, because there is nothing to count as demand.

### Data Cleaning

The cleaned dataset was built in Power Query from the raw files with the following steps:

- Read every file as text and converted each column deliberately, to protect zip prefixes and dates.
- Standardized blank values as nulls and trimmed text.
- Converted timestamps to dates and added purchase date and purchase month.
- Collapsed duplicate review rows, kept the latest valid review per order, and excluded reviews with an `id` shared across orders.
- Reduced payments to one row per order, with total value, main payment type, installments, and number of payment methods.
- Reduced geolocation to one row per zip prefix and attached coordinates to customers and sellers.
- Translated product categories to English, applied the two manual translations, and labeled missing categories `unknown`.
- Joined orders, customers, sellers, products, payments, and reviews to the order items without duplicating item rows.
- Created flags for the analysis rules: `demand_eligible`, `in_forecast_window`, `delivery_eligible`, `single_seller_order`, and `is_tier1`.
- Calculated fulfillment time (purchase to customer delivery), handling time, transit time, promised days, delivery delay, and lateness (judged on dates, not timestamps).
- Added freight-to-price ratio, seller-to-customer lane, same-state flag, and straight-line distance.
- Marked the first row of each order (`is_order_first_row`) so order-level metrics are not counted once per item.

The result is a refreshable table with **112,650 rows**, one for each order item.
