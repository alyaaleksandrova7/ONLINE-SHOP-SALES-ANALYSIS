# Online Shop Sales SQL Analysis 

SQL-based analytics project focused on e-commerce sales and customer behavior during 2023 and 2024.  
The analysis covers revenue and order trends, AOV, top products and customers, retention and churn, cohort analysis, ARPU/ARPPU and LTV to identify funnel bottlenecks and high-value users.

---

## Revenue & Orders

1.  Total revenue and yearly order quantity.
```sql
SELECT
    STRFTIME('%Y', order_date) AS year,
    COUNT(*) AS orders_count,
    ROUND(SUM(total_price), 2) AS revenue
FROM orders
GROUP BY year

year|orders_count|revenue    |
----+------------+-----------+
2023|        2210| 6124071.86|
2024|       12790|36371632.95|
```

2. This section analyzes orders of the online store and monthly AOV (average order value) based on completed orders.
```sql
SELECT
    STRFTIME('%Y-%m', order_date) AS month,
    COUNT(*) AS orders_count,
    ROUND(AVG(total_price), 2) AS avg_order_value
FROM orders
GROUP BY month
ORDER BY month;
month  |orders_count|avg_order_value|
-------+------------+---------------+
2023-11|         988|        2796.76|
2023-12|        1222|         2750.3|
2024-01|        1284|        2856.27|
2024-02|        1173|        2813.49|
2024-03|        1306|        2876.09|
2024-04|        1213|        2880.35|
2024-05|        1267|        2833.59|
2024-06|        1190|        2819.79|
2024-07|        1320|        2848.84|
2024-08|        1308|        2857.66|
2024-09|        1242|        2839.19|
2024-10|        1355|        2784.09|
2024-11|         132|        3115.29|
```
AOV remains stable across most months, with a noticeable peak in November, suggesting possible seasonal or promotional effects. Order volume is mostly stable with slight seasonal variations. October seems slightly higher; November 2024 is abnormal, likely due to missing data or more expensive items.

3. Top-5 products by revenue
```sql
WITH cte AS (
    SELECT 
        ot.product_id,
        SUM(ot.quantity) AS quantity,
        ROUND(AVG(ot.price_at_purchase), 2) AS avg_price,
        ROUND(SUM(ot.price_at_purchase * ot.quantity) 
        	/ SUM(ot.quantity), 2) AS weighted_price,
        ROUND(SUM(ot.quantity * ot.price_at_purchase), 2) AS revenue
    FROM order_items ot
    GROUP BY ot.product_id
)
SELECT 
    p.product_id,
    p.name,
    p.category,
    cte.quantity,
    cte.avg_price,
    cte.weighted_price,
    ROUND(
    (cte.avg_price - cte.weighted_price)
    	/ NULLIF(cte.weighted_price, 0) * 100,
    	2
		) AS price_bias_pct,
    cte.revenue 
FROM Products p
JOIN cte ON cte.product_id = p.product_id
ORDER BY revenue DESC
LIMIT 10;
product_id|name      |category   |quantity|avg_price|weighted_price|price_bias_pct|revenue |
----------+----------+-----------+--------+---------+--------------+--------------+--------+
       174|Product174|Accessories|      78|   474.77|        596.28|        -20.38| 46509.7|
       131|Product131|Stationery |      94|   391.47|        453.68|        -13.71|42646.32|
       187|Product187|Accessories|      95|   343.14|        424.63|        -19.19|40339.69|
        40|Product40 |Clothing   |      83|   458.29|        479.09|         -4.34|39764.29|
       104|Product104|Stationery |      75|   441.04|        499.29|        -11.67|37446.49|
        98|Product98 |Stationery |      79|   422.19|        473.03|        -10.75|37369.76|
       172|Product172|Electronics|      67|    523.8|         554.6|         -5.55|37157.87|
       125|Product125|Home       |      57|   536.95|        646.82|        -16.99|36868.67|
        85|Product85 |Electronics|      83|   358.95|        444.17|        -19.19|36865.75|
        65|Product65 |Accessories|      68|   425.43|        539.91|         -21.2|36713.97|
```
Weighted average price is calculated as revenue divided by total quantity sold, reflecting the actual price level at which customers most frequently purchase the product. The deviation shows how much the simple average price differs from the weighted average.
For top-revenue products, the volume-weighted average price exceeds the simple average price, indicating that higher-priced transactions account for a larger share of total units sold. As a result, simple averaging understates effective monetization.

4. Orders by day of the week
```sql
SELECT
    CASE CAST(STRFTIME('%w', order_date) AS INTEGER)
        WHEN 0 THEN 'Sunday'
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
    END AS day_of_week,
    COUNT(order_id) AS order_count
FROM orders
GROUP BY day_of_week
ORDER BY order_count DESC;
day_of_week|order_count|
-----------+-----------+
Tuesday    |       2172|
Thursday   |       2159|
Saturday   |       2159|
Sunday     |       2141|
Wednesday  |       2131|
Monday     |       2123|
Friday     |       2115|
```
Order volume is evenly distributed across all days of the week, with no significant day-of-week effect. Differences between the highest (Tuesday) and lowest (Friday) order counts are minimal, indicating stable customer purchasing behavior throughout the week rather than reliance on specific peak days.

---

## Customers 
This section could also include a deeper analysis of individual customers, identifying specific behaviors to inform personalized loyalty programs or targeted campaigns. However, for privacy and clarity, the portfolio focuses on aggregated metrics and segments, which convey the key insights without exposing individual-level data.

1. Customer Base and Engagement Overview

The table summarizes the customer base structure, including activation, engagement, and inactivity levels.
Metrics are calculated as of the last available order date in the dataset.

Customer base
- total_customers
- customers_without_orders
- active_customers
- one_time_customers

Engagement
- repeat_customers (more than 2 orders)
- loyal_customers (more than 3 orders)

Inactivity
- inactive_1m
- inactive_2m
- inactive_3m
- churned_customers (inactive 6 months)
```sql
WITH orders_per_customer AS (
    SELECT
        c.customer_id,
        COUNT(o.order_id) AS order_count
    FROM Customers_new c
    LEFT JOIN orders o
        ON c.customer_id = o.customer_id
    GROUP BY c.customer_id
),
monthly_activity AS (
    SELECT DISTINCT
        customer_id,
        STRFTIME('%Y-%m', order_date) AS ym
    FROM orders
),
activity_gaps AS (
    SELECT
        customer_id,
        (
            CAST(SUBSTR(ym,1,4) AS INT) * 12 + CAST(SUBSTR(ym,6,2) AS INT)
          - CAST(SUBSTR(LAG(ym) OVER (
                PARTITION BY customer_id ORDER BY ym
            ),1,4) AS INT) * 12
          - CAST(SUBSTR(LAG(ym) OVER (
                PARTITION BY customer_id ORDER BY ym
            ),6,2) AS INT)
        ) AS month_diff
    FROM monthly_activity
),
base AS (
    SELECT
        COUNT(*) AS total_customers,
        COUNT(CASE WHEN opc.order_count = 0 THEN 1 END) AS customers_without_orders,
        COUNT(CASE WHEN opc.order_count >= 1 THEN 1 END) AS active_customers,
        COUNT(CASE WHEN opc.order_count = 1 THEN 1 END) AS one_time_customers,
        COUNT(CASE WHEN opc.order_count >= 2 THEN 1 END) AS repeat_customers,
        COUNT(CASE WHEN opc.order_count >= 3 THEN 1 END) AS loyal_customers,
        COUNT(DISTINCT CASE WHEN ag.month_diff = 2 THEN opc.customer_id END) AS inactive_1m,
        COUNT(DISTINCT CASE WHEN ag.month_diff = 3 THEN opc.customer_id END) AS inactive_2m,
        COUNT(DISTINCT CASE WHEN ag.month_diff = 4 THEN opc.customer_id END) AS inactive_3m,
        COUNT(DISTINCT CASE WHEN ag.month_diff >= 7 THEN opc.customer_id END) AS churned_customers
    FROM orders_per_customer opc
    LEFT JOIN activity_gaps ag
        ON opc.customer_id = ag.customer_id
)
-- absolute values
SELECT
    'count' AS metric_type,
    total_customers,
    customers_without_orders,
    active_customers,
    one_time_customers,
    repeat_customers,
    loyal_customers,
    inactive_1m,
    inactive_2m,
    inactive_3m,
    churned_customers
FROM base
UNION ALL
-- percentages
SELECT
    'percentage' AS metric_type,
    100.0,
    ROUND(customers_without_orders * 100.0 / total_customers, 2),
    ROUND(active_customers * 100.0 / total_customers, 2),
    ROUND(one_time_customers * 100.0 / total_customers, 2),
    ROUND(repeat_customers * 100.0 / total_customers, 2),
    ROUND(loyal_customers * 100.0 / total_customers, 2),
    ROUND(inactive_1m * 100.0 / total_customers, 2),
    ROUND(inactive_2m * 100.0 / total_customers, 2),
    ROUND(inactive_3m * 100.0 / total_customers, 2),
    ROUND(churned_customers * 100.0 / total_customers, 2)
FROM base;
metric_type|total_customers|customers_without_orders|active_customers|one_time_customers|repeat_customers|loyal_customers|inactive_1m|inactive_2m|inactive_3m|churned_customers|
-----------+---------------+------------------------+----------------+------------------+----------------+---------------+-----------+-----------+-----------+-----------------+
count      |          14562|                       0|           14562|              5000|            9562|              0|        685|        619|        548|             1025|
percentage |          100.0|                     0.0|           100.0|             34.34|           65.66|            0.0|        4.7|       4.25|       3.76|             7.04|
```
The dataset contains 14,562 registered customers, all of whom have placed at least one completed order. Approximately 34.3% of customers placed exactly one order, while 65.7% of customers placed two orders, indicating a customer base with a strong share of repeat purchasing behavior.

Customer inactivity analysis shows a gradual decline over time: 4.7% of customers experienced a one-month purchase gap, 4.3% a two-month gap, and 3.8% a three-month gap between consecutive orders. Overall, 7.0% of customers can be classified as churned, defined as having a purchase gap of six months or longer.

2. Monthly Customer Retention

Monthly cohort analysis shows a sharp decline in retention after the first purchase, with less than half of customers returning in the following month. Retention continues to decrease significantly in subsequent months, indicating that customer engagement drops quickly after the initial transaction.
```sql
WITH first_order AS (
    SELECT
        customer_id,
        STRFTIME('%Y-%m', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY customer_id
)
, monthly_orders AS (
    SELECT DISTINCT
        customer_id,
        STRFTIME('%Y-%m', order_date) AS order_month
    FROM orders
)
, retention_base AS (
    SELECT
        mo.customer_id,
        fo.cohort_month,
        (
            CAST(SUBSTR(mo.order_month,1,4) AS INT) * 12
          + CAST(SUBSTR(mo.order_month,6,2) AS INT)
          -
            CAST(SUBSTR(fo.cohort_month,1,4) AS INT) * 12
          - CAST(SUBSTR(fo.cohort_month,6,2) AS INT)
        ) AS month_number
    FROM monthly_orders mo
    JOIN first_order fo
        ON mo.customer_id = fo.customer_id
)
, cohort_size AS (
    SELECT
        cohort_month,
        COUNT(DISTINCT customer_id) AS customers
    FROM retention_base
    WHERE month_number = 0
    GROUP BY cohort_month
)
SELECT
    rb.cohort_month,
    cs.customers AS cohort_size,
    ROUND(
        COUNT(DISTINCT CASE WHEN rb.month_number = 0 THEN rb.customer_id END)
        * 100.0 / cs.customers, 2
    ) AS month_0,
    ROUND(
        COUNT(DISTINCT CASE WHEN rb.month_number = 1 THEN rb.customer_id END)
        * 100.0 / cs.customers, 2
    ) AS month_1,
    ROUND(
        COUNT(DISTINCT CASE WHEN rb.month_number = 2 THEN rb.customer_id END)
        * 100.0 / cs.customers, 2
    ) AS month_2,
    ROUND(
        COUNT(DISTINCT CASE WHEN rb.month_number = 3 THEN rb.customer_id END)
        * 100.0 / cs.customers, 2
    ) AS month_3
FROM retention_base rb
JOIN cohort_size cs
    ON rb.cohort_month = cs.cohort_month
GROUP BY rb.cohort_month, cs.customers
ORDER BY rb.cohort_month;
cohort_month|cohort_size|month_0|month_1|month_2|month_3|
------------+-----------+-------+-------+-------+-------+
2023-11     |        967|  100.0|   5.79|   4.65|   6.41|
2023-12     |       1131|  100.0|   6.63|   5.04|   5.57|
2024-01     |       1117|  100.0|   5.55|   6.89|   5.91|
2024-02     |        953|  100.0|   6.93|   6.51|   6.09|
2024-03     |       1009|  100.0|   7.04|   6.84|   7.04|
2024-04     |        847|  100.0|   9.09|   7.79|   8.38|
2024-05     |        847|  100.0|   7.56|  10.98|   9.33|
2024-06     |        703|  100.0|   10.1|   8.11|   9.53|
2024-07     |        698|  100.0|   9.46|  11.32|   10.6|
2024-08     |        628|  100.0|  11.62|  11.31|   1.27|
2024-09     |        549|  100.0|  15.85|   1.64|    0.0|
2024-10     |        503|  100.0|   1.59|    0.0|    0.0|
2024-11     |         48|  100.0|    0.0|    0.0|    0.0|
```
Monthly cohort analysis based on customers’ first purchase month shows that a significant share of customers return for at least one additional purchase, with 66% of the overall customer base placing multiple orders. Retention over the first three months after the initial purchase varies between 5–15% per cohort, reflecting typical patterns of customer engagement in e-commerce. Some months (e.g., July–August 2024) show slightly higher retention in month 2, which may indicate seasonal or promotional effects, while later cohorts (October–November 2024) have low repeat purchase rates, likely due to dataset truncation or insufficient observation time.

These insights highlight opportunities to further increase engagement, suggesting that targeted retention initiatives — such as personalized promotions, loyalty programs, or follow-up campaigns — could help sustain repeat purchases and strengthen long-term customer loyalty.

While overall 66% of customers have made repeat purchases, the monthly cohort retention table shows lower percentages (5–15%) for the first 1–3 months. This discrepancy is expected: many repeat purchases occur after the initial 3-month window, and later cohorts may not yet have had time to make a second purchase. Therefore, low short-term retention does not contradict the strong overall repeat purchase rate.

3. Early Customer Retention & Repeat Purchase Analysis

While monthly cohort analysis provides a long-term view of customer retention, it is also important to examine early-stage customer behavior immediately after the first purchase. This section focuses on understanding how quickly customers disengage or re-engage, as early post-purchase behavior is a strong indicator of future customer lifetime value.

Specifically, this analysis examines:

- customers who made only one purchase and did not return within three months, treated as early churn
- customers who returned for a second purchase, with particular attention to those who reordered within 30 days of their first purchase as a sign of strong early retention
- the time elapsed between the first and second purchase, providing additional insight into repeat purchase dynamics beyond fixed retention windows
  
Together, these metrics complement monthly cohort retention by capturing short-term churn, early re-engagement, and repeat purchase timing, helping identify opportunities to accelerate customer retention and improve early lifecycle performance.
```sql
WITH CustomerOrders AS (
    SELECT 
        customer_id,
        order_date,
        MIN(order_date) OVER (PARTITION BY customer_id) AS first_order,
        LEAD(order_date) OVER (
            PARTITION BY customer_id 
            ORDER BY order_date
        ) AS second_order,
        COUNT(order_id) OVER (PARTITION BY customer_id) AS total_orders
    FROM orders
),
Metrics AS (
    SELECT
        COUNT(DISTINCT CASE 
            WHEN total_orders = 1 
             AND julianday('now') - julianday(first_order) > 90 
            THEN customer_id 
        END) AS churned_after_1_order,
        COUNT(DISTINCT CASE 
            WHEN second_order IS NOT NULL 
             AND julianday(second_order) - julianday(first_order) <= 30 
            THEN customer_id 
        END) AS retained_under_30_days,
        ROUND(
            AVG(
                CASE 
                    WHEN second_order IS NOT NULL
                    THEN julianday(second_order) - julianday(first_order)
                END
            ), 2
        ) AS avg_days_to_second_purchase,
        COUNT(DISTINCT customer_id) AS total_customers
    FROM CustomerOrders
)
SELECT
    'Count' AS metric_type,
    churned_after_1_order,
    retained_under_30_days,
    avg_days_to_second_purchase 
FROM Metrics
UNION ALL
SELECT
    'Rate (%)' AS metric_type,
    ROUND(churned_after_1_order * 100.0 / total_customers, 2),
    ROUND(retained_under_30_days * 100.0 / total_customers, 2),
    NULL
FROM Metrics;
metric_type|churned_after_1_order|retained_under_30_days|avg_days_to_second_purchase|
-----------+---------------------+----------------------+---------------------------+
Count      |                 5000|                   820|                     120.79|
Rate (%)   |                 50.0|                   8.2|                           |
```
The results show that 50% of customers churn after their first purchase, indicating a substantial early drop-off in the customer lifecycle. This suggests that half of newly acquired customers do not re-engage within three months, highlighting the importance of improving post-purchase onboarding and early engagement strategies.

Only 8.2% of customers place a second order within 30 days, pointing to weak short-term retention. While early repeat purchasing is limited, it does not necessarily imply poor overall retention, as many customers return later.

This is supported by the average time to second purchase of approximately 121 days, which indicates that repeat purchases often occur well beyond the initial 30-day window. As a result, low short-term retention rates are consistent with a longer repurchase cycle rather than a complete lack of customer loyalty.
From a business perspective, these insights suggest several opportunities:

- introducing early follow-up and reminder campaigns to shorten the time to second purchase
- offering time-limited incentives to encourage earlier repeat orders
- improving the first-time customer experience to reduce immediate churn and accelerate re-engagement
  
Overall, combining early churn, fast retention, and repeat purchase timing provides a more complete view of early customer behavior and complements the monthly cohort analysis.

---

## ARPU / ARPPU / LTV
While overall revenue and average order metrics provide a high-level view of business performance, understanding customer behavior at a more granular level is essential for targeted marketing and retention strategies. This analysis estimates Customer Lifetime Value (LTV) for each address using the formula:

LTV ≈ ARPU × average number of purchases per customer

In addition, we measure average order value (AOV), purchase frequency, and early repeat behavior (average days between the first and second purchase) to identify patterns in customer loyalty. By grouping data by customer address, we can identify geographic trends and areas with higher-value customers, which is useful for optimizing promotions, loyalty programs, and targeted campaigns.

Note on ARPPU: We did not include ARPPU (average revenue per paying user) in this section because the dataset contains only paying customers. Including ARPPU here would not provide additional insight beyond ARPU, and our focus is on total customer value and repeat behavior across all addresses, which is more actionable for marketing and retention strategies.
```sql
WITH customer_orders AS (
    SELECT
        c.address,
        o.customer_id,
        o.total_price,
        o.order_date,
        ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.order_date) AS rn,
        MIN(o.order_date) OVER (PARTITION BY o.customer_id) AS first_order
    FROM Customers_new c
    JOIN orders o
        ON c.customer_id = o.customer_id
),
customer_metrics AS (
    SELECT
        address,
        customer_id,
        SUM(total_price) AS customer_revenue,
        COUNT(*) AS orders_count,
        MAX(CASE WHEN rn = 2 THEN order_date END) AS second_order,
        MIN(first_order) AS first_order
    FROM customer_orders
    GROUP BY address, customer_id
)
SELECT
    address,
    ROUND(SUM(customer_revenue) / COUNT(DISTINCT customer_id), 2) AS ARPU,
    ROUND(SUM(customer_revenue) / SUM(orders_count), 2) AS AOV,
    ROUND(SUM(orders_count) * 1.0 / COUNT(DISTINCT customer_id), 2) AS frequency,
    ROUND(
        AVG(
            CASE
                WHEN second_order IS NOT NULL
                THEN JULIANDAY(second_order) - JULIANDAY(first_order)
            END
        ),
        2
    ) AS lifespan,
    ROUND(
        (SUM(customer_revenue) / COUNT(DISTINCT customer_id)) *
        (SUM(orders_count) * 1.0 / COUNT(DISTINCT customer_id)) *
        AVG(
            CASE
                WHEN second_order IS NOT NULL
                THEN JULIANDAY(second_order) - JULIANDAY(first_order)
            END
        ),
        2
    ) AS LTV
FROM customer_metrics
GROUP BY address;
address                             |ARPU   |AOV    |frequency|lifespan|LTV    |
------------------------------------+-------+-------+---------+--------+-------+
828 Shell Rd, Beach Haven, WV       |4897.64|3328.49|     1.47|  139.65|7206.52|
616 Boardwalk St, Canal City, VI    |4584.61| 2930.8|     1.56|  106.32|7171.64|
929 Harbor Rd, Bay View, AK         |4701.15| 3134.1|      1.5|  128.83|7051.73|
838 Marina Rd, Lake Haven, AS       |4563.45|2971.55|     1.54|  117.53|7008.16|
565 Summit Rd, High Point, NC       |4651.53|3111.39|      1.5|  127.94|6954.04|
343 Lake Ave, Harbor City, VA       |4465.29|2885.27|     1.55|  116.78|6910.58|
121 Forest Ln, Emerald Heights, WI  |4467.72|2895.75|     1.54|  108.24|6893.06|
696 Anchor St, Port Town, MO        |4505.41| 2961.3|     1.52|  133.22|6854.65|
454 Valley Dr, Bridge Town, CT      |4508.53| 2968.0|     1.52|  123.94|6848.67|
444 Magnolia Blvd, Mountain View, AZ|4432.56|2872.95|     1.54|  120.71| 6838.8|
353 Brook Ave, Fairview, NH         |4331.25|2743.78|     1.58|  117.91| 6837.2|
456 Elm St, Anytown, CA             |4513.15|2989.78|     1.51|  130.05| 6812.7|
131 Park Rd, Twin Peaks, MT         |4457.89|2930.07|     1.52|  124.07|6782.36|
919 Meadow Dr, Valley Falls, RI     |4372.92| 2834.3|     1.54|  131.08|6746.79|
717 Wave Dr, Coral Bay, OK          |4451.28|2939.53|     1.51|  103.99|6740.51|
898 Spring Ave, Blue Ridge, TN      |4414.98|2901.86|     1.52|  128.37|6717.07|
789 Oak Ave, Metropolis, NY         | 4509.9|3055.09|     1.48|  124.18|6657.47|
484 Dune Rd, Salt Lake, DC          |4355.05|2862.47|     1.52|  106.62| 6625.9|
686 Canyon St, Desert Springs, NM   |4416.67|2958.53|     1.49|  117.26|6593.45|
232 River St, Crystal Lake, IN      |4393.38|2928.92|      1.5|  127.78|6590.06|
939 Sand Ln, Gulf Shore, NE         |4441.07|3003.62|     1.48|  135.81|6566.44|
707 Aspen Way, Fawcett City, OH     |4280.04|2808.78|     1.52|  129.35|6521.97|
909 Dogwood Ln, Steel City, PA      |4153.34|2667.28|     1.56|   91.88|6467.35|
141 Shore Ln, Island City, MA       |4163.19|2685.93|     1.55|  121.53|6452.94|
676 Hill St, Pleasant Valley, UT    |4260.32|2813.42|     1.51|  117.17|6451.35|
818 Beach Dr, Port City, ME         |4272.78|2835.02|     1.51|  131.79|6439.69|
575 Vista Rd, Eagle Point, ID       |4266.23|2830.67|     1.51|   120.9|6429.82|
161 Harbor Ln, Bay Point, FM        | 4393.1|3014.87|     1.46|  125.72|6401.37|
272 Coast Rd, Sea Cliff, MH         |4407.24|3039.47|     1.45|  123.44|6390.49|
787 Creek Ln, Green Valley, AR      |4075.62|2605.42|     1.56|  105.03|6375.43|
333 Hickory Rd, River City, TX      |4295.89|2900.76|     1.48|  111.56| 6362.0|
585 Lighthouse Rd, Sea Haven, LA    |4158.49|2720.51|     1.53|  111.22|6356.55|
383 Shore Dr, Beach City, PW        |4198.91| 2786.0|     1.51|  104.24|6328.35|
101 Maple Dr, Smallville, KS        |4239.39|2844.32|     1.49|  120.34| 6318.7|
606 Cherry Cir, Bludhaven, DE       |4274.53|2895.65|     1.48|  135.19|6310.02|
404 Birch Blvd, Central City, CO    |4136.52|2731.67|     1.51|  121.05|6263.88|
474 Marina Ave, Harbor View, VT     |4137.41| 2745.2|     1.51|  120.32|6235.67|
464 Mountain Dr, Shadow Vale, WY    |4235.22| 2878.3|     1.47|  113.09|6231.82|
373 Seashell Dr, Ocean Grove, KY    |4172.89|2795.24|     1.49|  125.67|6229.53|
151 Pearl St, Seaside, SD           |4209.59|2874.84|     1.46|  118.58|6164.05|
949 Port Ave, Sound View, MP        |4065.44|2684.72|     1.51|  119.24|6156.23|
202 Pine Ln, Gotham, NJ             |4128.58|2769.98|     1.49|  125.65|6153.56|
242 Garden St, Liberty City, AL     |4102.31|2747.96|     1.49|  126.14|6124.16|
363 Bay St, Lakeside, IA            |4044.55|2683.59|     1.51|  114.76|6095.71|
727 Pier Dr, River Valley, GU       |4281.17|3011.88|     1.42|  120.54|6085.38|
123 Main St, Springfield, IL        |4104.57|2780.52|     1.48|  124.29|6059.13|
797 Ocean Ave, Coastal Haven, HI    |4240.74|2968.52|     1.43|  109.72| 6058.2|
111 Redwood Ave, Bay Harbor, MI     |4046.77|2732.55|     1.48|  125.41|5993.08|
888 Beech Rd, Sun City, GA          |3979.56|2644.64|      1.5|  122.48|5988.29|
777 Juniper Ave, Golden Valley, NV  |3996.27|2672.67|      1.5|  118.17|5975.38|
808 Willow St, Paradise City, FL    |4083.72|2802.55|     1.46|  125.38|5950.57|
303 Cedar Rd, Star City, WA         |3940.38|2618.61|      1.5|  110.25|5929.33|
666 Sycamore St, Silver Springs, MD |4077.44|2807.42|     1.45|  117.78| 5922.0|
252 Coast Dr, Sunset Beach, MS      | 3953.3|2648.14|     1.49|  117.87|5901.71|
222 Sequoia Dr, Lake Town, MN       |3885.61|2582.21|      1.5|  139.53|5846.92|
505 Walnut St, Coast City, OR       |3957.33|2706.97|     1.46|  123.11|5785.24|
595 Surf Ave, Bayshore, PR          |3787.61|2525.07|      1.5|  132.97|5681.41|
262 Coral Ave, Palm Beach, ND       |3948.62|2750.28|     1.44|  132.62|5669.09|
555 Palm Dr, Ocean Side, SC         |3814.58|2643.77|     1.44|  109.76|5503.89|
```
The LTV analysis by address reveals meaningful patterns in customer behavior and value:
- High-Value Customers:
The highest LTVs (e.g., 828 Shell Rd, Beach Haven, WV – 7,206; 616 Boardwalk St, Canal City, VI – 7,171) are driven by both high ARPU and above-average purchase frequency.
These addresses represent the most profitable customers and could be prioritized for VIP loyalty programs or personalized offers.
- ARPU vs. AOV:
While ARPU (average revenue per customer) varies across addresses, the AOV (average order value) is slightly lower, indicating that some high-LTV addresses achieve their value through repeat purchases rather than exceptionally large single orders.
- Frequency Patterns:
Frequency ranges from 1.42 to 1.58 purchases per customer, showing that even small increases in repeat purchases have a noticeable effect on LTV.
Addresses with lower frequency but high ARPU still achieve moderate LTV, suggesting strategies to increase repeat purchases could further boost lifetime value.
- Lifespan Insights:
Average days to second purchase (“lifespan”) varies widely (e.g., 91.88 days to 139.65 days).
Shorter lifespans combined with high frequency result in faster revenue accumulation, while longer lifespans indicate slower but steady engagement.
- Geographic Insights:
LTV differences by address show regional trends: certain areas consistently appear in the top LTV range, which could guide geo-targeted marketing or expansion strategies.
The LTV analysis by customer address shows that West Virginia (WV) and Alaska (AK) consistently rank among the highest in terms of ARPU, purchase frequency, and overall LTV, suggesting strong customer engagement and higher spending potential in these areas. On the other hand, South Carolina (SC) and Puerto Rico (PR) show the lowest values, which may reflect lower solvency, less demand for the product, or a smaller customer base in these regions. These geographic trends provide actionable insights for prioritizing marketing efforts, targeted promotions, and loyalty initiatives.
