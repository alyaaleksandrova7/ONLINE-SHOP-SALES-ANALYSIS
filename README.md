# ONLINE SHOP SALES ANALYSIS (SQL)
SQL-based analytics project focused on e-commerce sales and customer behavior. Covers revenue and order trends, AOV, top products and users, retention and churn, cohort analysis, ARPU/ARPPU and LTV, providing actionable insights to identify funnel bottlenecks and high-value customers.

This section analyzes the **TOTAL REVENUE** of the online store and its monthly dynamics based on completed orders.

SELECT	STRFTIME('%Y-%m', order_date) AS month,
		ROUND(AVG(total_price), 2) AS revenue FROM orders
GROUP BY month

month  |revenue|
-------+-------+
2023-11|2796.76|
2023-12| 2750.3|
2024-01|2856.27|
2024-02|2813.49|
2024-03|2876.09|
2024-04|2880.35|
2024-05|2833.59|
2024-06|2819.79|
2024-07|2848.84|
2024-08|2857.66|
2024-09|2839.19|
2024-10|2784.09|
2024-11|3115.29|

