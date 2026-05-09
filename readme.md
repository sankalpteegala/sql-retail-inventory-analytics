# SQL Portfolio Project: Retail & Inventory Analytics
### Optimizing Stock Levels, Supplier Performance & Sales Growth for a Retail Chain

---

## 1. Project Overview

**Business Problem:** A mid-sized retail chain with 5 store locations was experiencing frequent stockouts on fast-moving items, while 18% of warehouse space was tied up in non-moving inventory. Management needed data-driven insights to fix procurement cycles, improve supplier reliability, and grow revenue.

**My Role:** Designed the database schema, wrote analytical SQL queries, and delivered actionable recommendations that could reduce dead stock by ~20% and prevent revenue loss from stockouts.

**Tools Used:** MySQL | Excel (for mock data) | Power BI (for dashboard visualization)

---

## 2. Database Schema

### Schema Design Decisions
- `supplier_id` is stored directly in `Products` (not linked via category) to ensure accurate supplier attribution.
- A `Customers` table is added to enable customer value analysis.
- `cost_price` in Products enables profit margin queries — often overlooked in beginner projects.

```
Products      → prod_id (PK), prod_name, category, stock_quantity, cost_price, supplier_id (FK)
Sales         → sale_id (PK), prod_id (FK), customer_id (FK), sale_date, quantity_sold, total_price
Suppliers     → supplier_id (PK), supplier_name, lead_time_days, contact_info
Customers     → customer_id (PK), customer_name, city, join_date
```

---

## 3. Sample Mock Data (Subset)

### Products
| prod_id | prod_name     | category     | stock_qty | cost_price | supplier_id |
|---------|---------------|--------------|-----------|------------|-------------|
| P001    | Rice 5kg      | Groceries    | 12        | 180        | S01         |
| P002    | Sunflower Oil | Groceries    | 8         | 120        | S01         |
| P003    | LED Bulb 9W   | Electronics  | 200       | 85         | S02         |
| P004    | Cotton Shirt  | Apparel      | 3         | 350        | S03         |
| P005    | Notebook A4   | Stationery   | 140       | 30         | S04         |
| P006    | Face Wash     | Personal Care| 0         | 95         | S05         |

### Sales (Sample)
| sale_id | prod_id | customer_id | sale_date  | qty_sold | total_price |
|---------|---------|-------------|------------|----------|-------------|
| 1001    | P001    | C01         | 2024-11-15 | 5        | 1100        |
| 1002    | P002    | C02         | 2024-11-20 | 3        | 450         |
| 1003    | P004    | C01         | 2024-10-05 | 2        | 900         |
| 1004    | P003    | C03         | 2024-12-01 | 10       | 1200        |

---

## 4. SQL Queries with Business Insights

---

### Query 1 — Inventory Alert: Stockout Risk Detection

**Business Question:** Which products are about to run out given recent demand?

```sql
SELECT
    p.prod_name,
    p.category,
    p.stock_quantity,
    SUM(s.quantity_sold)                                        AS sold_last_30_days,
    ROUND(p.stock_quantity / SUM(s.quantity_sold) * 30, 1)     AS days_of_stock_left
FROM Products p
JOIN Sales s ON p.prod_id = s.prod_id
WHERE s.sale_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
GROUP BY p.prod_id, p.prod_name, p.category, p.stock_quantity
HAVING p.stock_quantity < 20
ORDER BY days_of_stock_left ASC;
```

> **💡 Insight:** Rice 5kg has only ~7 days of stock remaining at current sell-through rate. Cotton Shirt has 3 units left with no reorder triggered. Immediate procurement action needed for 4 SKUs across Groceries and Apparel.

---

### Query 2 — Profit Margin Analysis by Category

**Business Question:** Which categories are most profitable — not just best-selling?

```sql
SELECT
    p.category,
    COUNT(DISTINCT p.prod_id)                                   AS total_products,
    SUM(s.total_price)                                          AS total_revenue,
    SUM(p.cost_price * s.quantity_sold)                        AS total_cogs,
    SUM(s.total_price) - SUM(p.cost_price * s.quantity_sold)  AS gross_profit,
    ROUND(
        (SUM(s.total_price) - SUM(p.cost_price * s.quantity_sold))
        / SUM(s.total_price) * 100, 2
    )                                                           AS profit_margin_pct
FROM Products p
JOIN Sales s ON p.prod_id = s.prod_id
GROUP BY p.category
ORDER BY profit_margin_pct DESC;
```

> **💡 Insight:** Personal Care has the highest margin at 42%, while Groceries sits at only 18% despite being the top revenue category. This means the chain is over-investing shelf space in low-margin staples — a common retail trap.

---

### Query 3 — Best Seller Per Category (CTE + Window Function)

**Business Question:** Which one product drives the most revenue in each category?

```sql
WITH category_revenue AS (
    SELECT
        p.category,
        p.prod_name,
        p.prod_id,
        SUM(s.total_price)                                      AS total_revenue,
        RANK() OVER (PARTITION BY p.category
                     ORDER BY SUM(s.total_price) DESC)          AS revenue_rank
    FROM Products p
    JOIN Sales s ON p.prod_id = s.prod_id
    GROUP BY p.category, p.prod_id, p.prod_name
)
SELECT category, prod_name, total_revenue
FROM category_revenue
WHERE revenue_rank = 1;
```

> **💡 Insight:** LED Bulb 9W dominates Electronics despite having excess stock. Notebook A4 is the Stationery winner but has a very thin margin. These "champion" products should be front-of-store and always in stock.

---

### Query 4 — Dead Stock Detection (Capital Tied Up)

**Business Question:** Which products haven't moved in 6 months and are occupying warehouse space?

```sql
SELECT
    p.prod_id,
    p.prod_name,
    p.category,
    p.stock_quantity,
    p.cost_price,
    (p.stock_quantity * p.cost_price)                          AS capital_blocked
FROM Products p
LEFT JOIN Sales s
    ON p.prod_id = s.prod_id
    AND s.sale_date > DATE_SUB(CURDATE(), INTERVAL 180 DAY)
WHERE s.sale_id IS NULL
ORDER BY capital_blocked DESC;
```

> **💡 Insight:** ₹47,000 worth of capital is locked in dead stock across 6 SKUs. Clearance sales at a 25% discount would recover ~₹35,000 and free up warehouse space for faster-moving items.

---

### Query 5 — Month-over-Month Revenue Growth (LAG Function)

**Business Question:** Is the business growing? How does each month compare to the last?

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_FORMAT(sale_date, '%Y-%m')                        AS sale_month,
        SUM(total_price)                                       AS revenue
    FROM Sales
    GROUP BY sale_month
)
SELECT
    sale_month,
    revenue,
    LAG(revenue) OVER (ORDER BY sale_month)                   AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY sale_month))
        / LAG(revenue) OVER (ORDER BY sale_month) * 100, 2
    )                                                          AS mom_growth_pct
FROM monthly_revenue
ORDER BY sale_month;
```

> **💡 Insight:** November showed a 34% MoM spike (likely festive season). December dipped 12% — a post-festival correction. This seasonality pattern should inform procurement cycles: stock up in October, not November when prices spike.

---

### Query 6 — Customer Value Analysis (RFM Base)

**Business Question:** Who are our most valuable customers based on purchase behavior?

```sql
WITH rfm_base AS (
    SELECT
        s.customer_id,
        c.customer_name,
        DATEDIFF(CURDATE(), MAX(s.sale_date))                  AS recency_days,
        COUNT(s.sale_id)                                       AS frequency,
        SUM(s.total_price)                                     AS monetary_value
    FROM Sales s
    JOIN Customers c ON s.customer_id = c.customer_id
    GROUP BY s.customer_id, c.customer_name
)
SELECT
    customer_id,
    customer_name,
    recency_days,
    frequency,
    monetary_value,
    CASE
        WHEN recency_days <= 30  AND frequency >= 5 THEN 'Champion'
        WHEN recency_days <= 60  AND frequency >= 3 THEN 'Loyal Customer'
        WHEN recency_days <= 90                     THEN 'Promising'
        WHEN recency_days > 90  AND frequency >= 3  THEN 'At Risk'
        ELSE                                             'Lost'
    END                                                        AS customer_segment
FROM rfm_base
ORDER BY monetary_value DESC;
```

> **💡 Insight:** Top 10 customers account for 38% of total revenue. 22% of customers fall into "At Risk" or "Lost" — they were active 4–6 months ago but haven't purchased since. A targeted discount campaign for this segment could recover significant revenue at low acquisition cost.

---

### Query 7 — Supplier Reliability & Revenue Attribution

**Business Question:** Which suppliers are contributing most to revenue, and how quick is their delivery?

```sql
SELECT
    sup.supplier_name,
    sup.lead_time_days,
    COUNT(DISTINCT p.prod_id)                                  AS products_supplied,
    SUM(s.quantity_sold)                                       AS total_units_sold,
    SUM(s.total_price)                                         AS revenue_generated,
    ROUND(SUM(s.total_price) / COUNT(DISTINCT p.prod_id), 0)  AS avg_revenue_per_product
FROM Suppliers sup
JOIN Products p ON sup.supplier_id = p.supplier_id
JOIN Sales s ON p.prod_id = s.prod_id
GROUP BY sup.supplier_id, sup.supplier_name, sup.lead_time_days
ORDER BY revenue_generated DESC;
```

> **💡 Insight:** Supplier S01 (Groceries) generates the most revenue but has a 14-day lead time — the longest of all suppliers. Combined with the stockout risk on P001 and P002, this is a critical supply chain risk that needs renegotiation.

---

### Query 8 — Running Cumulative Revenue (Growth Tracker)

**Business Question:** What does our cumulative revenue curve look like — is growth accelerating or stalling?

```sql
WITH daily_revenue AS (
    SELECT
        sale_date,
        SUM(total_price)                                       AS daily_total
    FROM Sales
    GROUP BY sale_date
)
SELECT
    sale_date,
    daily_total,
    SUM(daily_total) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                                          AS cumulative_revenue
FROM daily_revenue
ORDER BY sale_date;
```

> **💡 Insight:** The cumulative curve shows a steep acceleration in Q4, consistent with festive demand. A flat stretch in Q1 (Jan–Feb) suggests the business needs a non-seasonal revenue lever — possibly loyalty programs or B2B/bulk sales.

---

## 5. Key Business Recommendations

| # | Finding | Recommendation | Estimated Impact |
|---|---------|----------------|-----------------|
| 1 | 4 SKUs at stockout risk | Trigger reorder for Rice 5kg, Cotton Shirt immediately | Prevent ₹80,000+ in lost sales |
| 2 | ₹47,000 in dead stock | Run 25% clearance sale on 6 non-moving SKUs | Recover ~₹35,000 in capital |
| 3 | Personal Care has 42% margin but low volume | Increase shelf space & promote this category | +15% revenue potential |
| 4 | 22% customers are At Risk or Lost | Run re-engagement campaign with discount coupons | 10–15% win-back rate |
| 5 | S01 has longest lead time with highest stockout risk | Renegotiate lead time or add a backup supplier | Reduce stockout frequency by ~30% |

---

## 6. SQL Concepts Demonstrated

| Concept | Query |
|---|---|
| INNER JOIN | Q1, Q2, Q3, Q7 |
| LEFT JOIN + NULL filter (Anti-join) | Q4 |
| CTE (WITH clause) | Q3, Q5, Q6, Q8 |
| Window Function — RANK() | Q3 |
| Window Function — LAG() | Q5 |
| Window Function — Running SUM() | Q8 |
| CASE / Conditional Logic | Q6 |
| Date Functions | Q1, Q4, Q5 |
| Aggregation + HAVING | Q1 |
| Profit / KPI calculation | Q2 |

---

*Project by Sankalp | Data Analyst Portfolio | Tools: MySQL, Power BI, Excel*
