# olist-demand-forecasting-fulfillment-reliability

## Data Structure & Initial Checks

The original dataset contains **nine tables** and about **1.55 million records** (1,554,239 rows) from Olist, a Brazilian online marketplace, covering orders placed between **September 2016 and October 2018**. Each order is a purchase made by one customer from one or more independent sellers. The analysis starts with the nine raw tables, rebuilds them in Power Query into one flat order-item table for analysis, and uses a few small summary tables for forecasting, delivery reliability, and customer experience.

The dataset does **not** include inventory levels, supplier lead times, or product costs. Because of this, the project measures *delivery* reliability (seller to customer) and uses item revenue as a profit proxy. It does not measure procurement lead time or true safety stock.

### Database Schema

![Olist Database Schema](images/olist_schema.png)

### Raw Data Structure

| Table | One row represents | Rows | Key fields |
|---|---|---|---|
| `olist_orders_dataset` | One order | 99,441 | `order_id`, `customer_id`, `order_status`, purchase, approval, carrier, delivery, and estimated delivery timestamps |
| `olist_order_items_dataset` | One item within an order | 112,650 | `order_id`, `order_item_id`, `product_id`, `seller_id`, `price`, `freight_value` |
| `olist_order_payments_dataset` | One payment record for an order | 103,886 | `order_id`, `payment_type`, `payment_installments`, `payment_value` |
| `olist_order_reviews_dataset` | One review record | 102,541* | `review_id`, `order_id`, `review_score`, `review_comment_title`, `review_comment_message` |
| `olist_products_dataset` | One product | 32,951 | `product_id`, `product_category_name`, `product_weight_g` |
| `olist_customers_dataset` | One customer ID (a new ID per order) | 99,441 | `customer_id`, `customer_unique_id`, zip prefix, city, state |
| `olist_sellers_dataset` | One seller | 3,095 | `seller_id`, zip prefix, city, state |
| `olist_geolocation_dataset` | One coordinate point for a zip prefix | 1,000,163 | zip prefix, latitude, longitude, city, state |
| `product_category_name_translation` | One Portuguese category name | 71 | Portuguese name, English name |

\*The reviews file contains 3,317 exact duplicate rows. After removing them, 99,224 rows remain, which matches the published row count.

The `orders` table is the hub. It connects to customers through `customer_id`, to items, payments, and reviews through `order_id`, and items connect to products and sellers. Zip prefixes in the customers, sellers, and geolocation tables connect locations to coordinates.

### Order Status

| Status | Orders | Meaning | Treatment |
|---|---|---|---|
| `delivered` | 96,478 | Order reached the customer | Counted as demand. The only status eligible for delivery analysis |
| `shipped` | 1,107 | Order is with the carrier | Counted as demand |
| `canceled` | 625 | Order canceled | Excluded from demand |
| `unavailable` | 609 | Items unavailable | Excluded from demand |
| `invoiced` | 314 | Order invoiced, not yet shipped | Counted as demand |
| `processing` | 301 | Order is being prepared | Counted as demand |
| `created` | 5 | Order created, not yet approved | Counted as demand, but none of these orders have items, so they do not appear in the item-level analysis |
| `approved` | 2 | Payment approved | Counted as demand |

Of the 99,441 orders, **775 have no order items** (603 `unavailable`, 164 `canceled`, 5 `created`, 2 `invoiced`, and 1 `shipped`). They cannot contribute demand or revenue, so they are not in the flat table, which leaves **98,666 orders** with at least one item.

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
Row counts were compared with the published counts for each file to confirm the import was complete. Every file matched except the reviews file, which has 102,541 rows because it contains 3,317 exact duplicate rows; removing them gives exactly the published 99,224. Zip code prefixes were imported as text, because reading them as numbers drops leading zeros and breaks joins between tables.

**Date range check:**  
Orders span **September 2016 to October 2018**. The earliest months of 2016 and the last months of 2018 contain very few orders, so the demand analysis uses **January 2017 to August 2018**, which gives 20 complete months.

**Order status check:**  
**97.02%** of orders are `delivered`. Orders with `canceled` or `unavailable` status are excluded from demand analysis. The `processing` (301) and `approved` (2) orders are kept because they are real purchases that had not shipped yet.

**Delivery timestamp validation:**  
Only `delivered` orders can have a customer delivery date. Timestamp order was tested for six rules. No order was approved before purchase, delivered before purchase, or had an estimated delivery date before purchase. Three problems were found:

| Issue | Orders |
|---|---|
| Carrier date before purchase | 166 |
| Delivered before carrier date | 23 |
| Carrier date before approval | 1,359 |

Orders where the carrier date is before purchase or delivery is before the carrier date, along with orders missing a delivery or carrier date, are excluded from delivery analysis but kept for demand analysis. The 1,359 orders where the carrier date is before the approval timestamp are **not** excluded, because the approval timestamp is not used in any delivery measure and these orders keep a valid purchase, carrier, and delivery sequence.

Of the **96,478 delivered orders**, **96,281** passed all delivery rules (197 were excluded). Of those, **95,029** come from a single seller and **1,252** involve more than one seller. Delivery reliability statistics use only the 95,029 single-seller orders, because a multi-seller order has one delivery time but several sellers. **6,531** of the 96,281 eligible orders were delivered after the estimated delivery date, judged by date rather than timestamp.

In the Phase 0 profile of delivered orders with a delivery date (**96,470 orders**), delivery time from purchase to customer averaged **12.56 days**, with a median of **10.22 days**, a 95th percentile of **29.27 days**, and a maximum of **209.63 days**. Because the mean is higher than the median and the tail is long, delivery times are **right-skewed**. This is why the delivery buffer uses an empirical percentile rather than a mean plus standard deviations.

**Missing values:**  
Missing values were found in the order timestamps, review comments, and product category.

- Review comment titles are missing for about **88%** of reviews and comment messages for about **58–59%**. A missing comment mostly means the customer left a star rating without text, not that no review exists.
- Missing `product_category_name` values were labeled `unknown`. In the demand window these items account for **1,449 orders** and about **1.33%** of revenue (before removing canceled and unavailable orders).
- Blank text values were standardized as nulls before cleaning.

**Review duplicate and coverage checks:**  
There were **3,317 duplicate review groups** (6,634 rows), and every group was fully identical, so one row per group was kept and **3,317 rows were removed**. The remaining **99,224 review rows** cover **98,673 orders**, so a small number of orders (551 more review rows than orders) have more than one distinct review. For these orders the most recent answer was kept. Another **768 orders have no review record at all** and form the "no review" group used in the customer experience analysis.

A further **789 `review_id` values** were shared by more than one order. In every case the orders belonged to the same customer, and 748 of them were placed within an hour of each other. These look like one review copied across several orders, so they were treated as errors: they are excluded from review metrics, but the orders are kept and labeled `invalid_shared_id` so they are not mistaken for orders with no review.

**Category coverage and translation:**  
Products fall into **73 product categories**, and the translation file covers **71** of them. Two categories had no English name and were translated manually: `pc_gamer` became `gaming_pc` and `portateis_cozinha_e_preparadores_de_alimentos` became `kitchen_appliances_and_food_prep`. All categories are kept in the data. Only the categories selected below are forecast.

**Forecast category selection:**  
Categories were tested on three criteria over the 20-month window: active in every month, at least 30 orders per month on average, and within the first 80% of cumulative revenue (revenue is used as a profit proxy because item price is the only value available and freight is excluded). **17 categories** met all three and cover about **79.6%** of window revenue, so they are the categories forecast in this project. Two points are worth noting:

- `computers` ranks 17th by revenue but exists in only **13 of the 20 months**, so it was reported but not forecast.
- `pet_shop` sits just under the 80% revenue line and is kept because it was active in all 20 months with about 85 orders per month.

The list is frozen on purpose so forecasts do not change category lists when data is refreshed.

**Location and geolocation checks:**  
The geolocation table has **1,000,163 rows** but only one location is needed per zip prefix. There were **128,174 groups of exact duplicate rows**, and **261,831 duplicate rows were removed**, leaving 738,332 rows. In the profiling check, **31 points across 20 zip prefixes** fell outside Brazil's mainland bounding box. The coordinate limits used in the pipeline were widened to keep Fernando de Noronha (zip prefix 53990), which lies far east of the mainland. The remaining points were reduced to one row per zip prefix using median latitude and longitude, producing a lookup of **19,011 zip prefixes** out of 19,015 in the raw table (4 prefixes lost all their points as out-of-Brazil coordinates).

In the raw data, **278 customer rows (0.28%, 157 distinct zip prefixes)** and **7 seller rows (0.23%)** had no matching geolocation record. Where coordinates are missing they are left blank rather than estimated, and the state is kept.

**Price, freight, and payment checks:**  
No order item has a missing product, seller, shipping limit date, price, or freight value. There are no zero or negative prices and no negative freight values. **383 items have zero freight**, which was kept because free shipping is real.

Freight is a meaningful share of item price: for **38.4%** of items freight is 10–24% of the price and for **29.2%** it is 25–49%. For **4,124 items (3.7%)** freight is equal to or above the item price (3,593 items at 100–199% and 531 at 200% or more). These were kept because they can be real low-price items with high shipping cost, and they matter for the customer experience analysis.

In payments, there are **9 zero-value records** and **3 records with an undefined payment type** (`not_defined`), all kept as-is. There are **2 records with zero installments**, which were treated as missing installment information.

**Join integrity:**  
There are **no orphan records**: every item, payment, and review points to an existing order, every item points to an existing product and seller, and every order points to an existing customer. Every customer has at least one order. Some orders are missing child records: 775 orders have no items (see order status), 1 order has no payment record, and 768 orders have no review record.

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

The result is a refreshable table with **112,650 rows**, one for each order item, covering **98,666 orders**. Of these orders, **96,281** are eligible for delivery analysis and **95,029** of those come from a single seller.

The cleaned table was validated by rebuilding the same rules independently in SQL. Row counts, the demand, forecast-window, and delivery flags, review status, category, geolocation, and delivery-time statistics all matched between Power Query and SQL.

### Category A: Demand & Category Performance

**Supporting materials:**
- Excel workbook: [olist_pipeline_v3.xlsx](olist_pipeline_v3.xlsx)
  - `fc_accuracy`: backtest accuracy by series and model
  - `fc_drivers`: correlation tests and error breakdown by series
  - `fc_viewer`: actual vs backtest vs forecast chart for any series

This section answers:

**Which product categories and regions have the most volatile or hardest-to-predict demand, and why?**

Weekly item demand was forecast for the 17 Tier 1 categories and Brazil's 5 regions. Three methods were tested: a 4-week moving average (MA4) as the baseline, simple exponential smoothing (SES), and Excel's FORECAST.ETS. Each method was fitted on 74 weeks (January 2017 to May 2018) and scored on the 11 held-out weeks that followed (4 June to 19 August 2018). Accuracy is measured with MAPE (average weekly % error), bias (total over- or under-forecast) and RMSE (typical miss in items per week).

<p align="center">
  <img src="images/forecast_volume_vs_error.png" width="700"><br>
</p>

#### Insight 1: Simple exponential smoothing was the most reliable method, and FORECAST.ETS did not beat the baseline.

| Measure | FORECAST.ETS | SES | MA4 |
|---|---|---|---|
| Median MAPE, 17 categories | 33.3% | **29.8%** | 32.0% |
| Median MAPE, 5 regions | 33.0% | **18.8%** | 22.1% |
| Series where the method is best (of 24) | 4 | **17** | 3 |
| Median bias, 17 categories | +14.8% | **+7.7%** | +14.3% |

- SES had the lowest error for both categories and regions, the smallest bias, and was the best method for **17 of 24 series**.
- FORECAST.ETS did worse because it projects a trend. It learned the fast growth of 2017 and kept projecting it forward, but by mid-2018 demand had stopped growing and then dipped after the truckers' strike.
- That left FORECAST.ETS consistently too high, with a median over-forecast of **14.8%**, about twice that of SES.
- With only 1.4 years of history and no repeated seasonal pattern to learn from, a method that tracks the current level is safer than one that assumes past growth will continue.

#### Insight 2: Regions forecast better than categories, but Southeast and North were harder to predict for different reasons.

- Three of five regions had MAPE under **20%** ("good" on the Lewis scale): **Northeast (15.5%)**, **South (17.3%)** and **Center-West (18.8%)**. Only one category, **health_beauty (14.6%)**, reached that level.
- Regions forecast better because each one combines many categories, so the ups and downs of individual categories partly cancel out.

**Southeast (24.9% MAPE): large, but hit by a sharp temporary drop**
- Southeast is the largest region, with **69%** of all demand. Normally, a series this large should be among the easiest to forecast, because random week-to-week noise is small relative to its size.
- Its error came instead from one sharp swing. In the week of 9 July, Southeast demand fell to about **half** its normal pre-strike level, a much deeper drop than any other region.
- Two weeks later it had fully recovered. A forecasting model cannot anticipate a sudden one-week collapse and rebound like this, so the forecast was far too high during the drop.
- The drop was concentrated in São Paulo, Rio de Janeiro and Minas Gerais, the region's three largest states. Because Southeast is most of the business, this swing also drove the national total.

**North (30.6% MAPE): small, and moving against the other regions**
- North is the smallest region, at about **31 items a week**. At that size, a handful of orders more or less swings the weekly total by a large percentage, which pushes MAPE up.
- North also behaved differently from the rest of the country. It recovered quickly after the strike, barely dipped in July, and then had a short spike in late July before dropping off.
- The models expected North to follow its own past level. It ended up as the only region they under-forecast (bias **-7%**), because its demand returned to normal while the other regions stayed lower after the strike.

#### Insight 3: Volume is the strongest predictor of forecast accuracy.

- Higher-volume series were consistently easier to forecast. Across the 17 categories, the rank correlation between weekly volume and MAPE is **-0.70** (p = **0.002**), so the pattern is very unlikely to be chance.
- The pattern holds when regions are included (**-0.73**) and when the two worst categories are removed (**-0.59**), so it isn't driven by a few extreme cases.
- Part of this is unavoidable. Orders arrive somewhat randomly, so even a perfect forecast of a category's average will miss in individual weeks. That random miss is much larger, as a percentage, for small categories.
- For example, randomness alone would cause about a **20%** weekly error for office_furniture (16 items a week), compared with about **6%** for health_beauty (208 items a week).
- How noisy a category was before the test did **not** predict its error (correlation **0.09**). Low volume and changes in demand level made series hard to forecast, not past volatility.

#### Insight 4: Categories whose demand stayed down after the strike were over-forecast the most.

| Category | Level shift after strike | Bias (best model) | MAPE |
|---|---|---|---|
| office_furniture | -45% | +48% | 86.1% |
| cool_stuff | -45% | +23% | 36.2% |
| garden_tools | -38% | +56% | 72.3% |
| furniture_decor | -27% | +21% | 41.1% |
| pet_shop | +18% | -4% | 25.2% |
| stationery | +25% | 0% | 29.0% |

- The models treated the May 2018 strike as a temporary shock and expected each category to return to its pre-strike level afterwards.
- For some categories, mostly home and garden goods, demand did not come back. It settled at a permanently lower level, so the forecasts stayed too high for the whole test period.
- The table shows this directly: categories that fell the most after the strike (left column) were over-forecast the most (middle column). Across all 17 categories, the correlation is **-0.76** (p < **0.001**).
- Categories that grew after the strike, such as pet_shop and stationery, had almost no bias.
- The hardest categories to predict were those whose demand level changed, not those with the most week-to-week noise. Planning for these categories should allow wider safety margins, or use monthly rather than weekly forecasts.

#### Insight 5: office_furniture's high MAPE comes from small volumes and a steady over-forecast, not large misses.

- office_furniture had the worst MAPE (**86.1%**), but its forecasts were usually off by only about **9 to 10 items a week**.
- The misses were consistent rather than driven by a few large outliers. The model over-forecast in **10 of 11 weeks**, because demand settled at about half its pre-strike level.
- MAPE makes this look worse than it is, because each miss is divided by a small number. In the final test week only **5 items** sold, so that one week alone added about **35 points** to the MAPE.
- For low-volume categories, RMSE and bias describe forecast quality better than MAPE, and monthly planning would smooth out these small-count swings.

#### Insight 6: The World Cup lowered orders during Brazil's matches, but explains little of the summer dip.

- The 2018 World Cup (14 June to 15 July) overlapped the test period, so it is a possible explanation for the weak June and July demand.
- People did stop shopping to watch Brazil play. Orders during Brazil's five matches were about **half** their normal level for the same day and hour. A national study by Konduto, an anti-fraud firm, found a similar pattern across Brazilian e-commerce, with sales falling **up to 28%** during the games.
- However, the matches lasted only a few hours each, so the lost orders add up to only about **6%** of one normal week.
- The biggest drop (7 to 12 July) came after Brazil was knocked out on 6 July, when the team was no longer playing.
- The World Cup explains the short dips during matches, but not the larger July drop. The data does not show what caused it, so it is treated as an unexplained shock.
