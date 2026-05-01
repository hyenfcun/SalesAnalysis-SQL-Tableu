# AtliQ Hardware — Sales Insights Data Analysis

**An end-to-end data analysis project uncovering why AtliQ Hardware's revenue declined 57% in two years — and what management should do about it.**

This project uses pure SQL for data cleaning and analysis, and Tableau for interactive dashboarding. Every insight is paired with a business recommendation.

---

## Approach and methodology

This project follows the AIMS Grid framework:

| Component | Detail |
|---|---|
| **Purpose** | Identify root causes of AtliQ Hardware's revenue decline and provide data-backed recommendations |
| **Stakeholders** | Sales Director, Regional Managers, Data & Analytics Team |
| **End result** | 3 interactive Tableau dashboards backed by clean, well-modeled SQL data |
| **Success criteria** | Dashboards enable 10% cost savings; analysts reclaim 20% of time previously spent on manual reporting |

**Technical pipeline:** Raw MySQL dump → SQL-based audit → SQL-based cleaning → SQL analysis → CSV export → Tableau visualization → Business recommendations.

---

## Dataset

The data comes from `db_dump_v1.sql`, a MySQL dump containing 5 tables in a star schema with 150,283 transactions across 38 customers, 14 markets, and 279 products from 2017–2020.

| Table | Type | Rows | Key columns |
|---|---|---|---|
| `transactions` | Fact | 150,283 | `product_code`, `customer_code`, `market_code`, `order_date`, `sales_qty`, `sales_amount`, `currency` |
| `customers` | Dimension | 38 | `customer_code`, `custmer_name`, `customer_type` |
| `markets` | Dimension | 17 | `markets_code`, `markets_name`, `zone` |
| `products` | Dimension | 279 | `product_code`, `product_type` |
| `date` | Dimension | 1,126 | `date`, `year`, `month_name` |

---

## Data cleaning (SQL)

Before analysis, I audited the raw data and found 4 quality issues. Every cleaning decision is documented with the SQL used and the reasoning behind it.

### Issue 1: Currency field contains carriage return artifacts

The `currency` column contained values like `'INR\r'` instead of `'INR'` — artifacts from a Windows CSV export.

```sql
SELECT DISTINCT currency, LENGTH(currency) AS len FROM transactions;
-- Result: INR (len 3), INR\r (len 4), USD (len 3), USD\r (len 4)

UPDATE transactions SET currency = 'INR' WHERE currency = 'INR\r';
UPDATE transactions SET currency = 'USD' WHERE currency = 'USD\r';
-- 150,000 INR rows + 2 USD rows fixed
```

### Issue 2: Invalid sales amounts

```sql
SELECT sales_amount, COUNT(*) AS cnt
FROM transactions WHERE sales_amount <= 0
GROUP BY sales_amount ORDER BY cnt DESC;
-- Result: 1,609 rows with amount = 0, 2 rows with amount = -1
```

**Decision:** These are data entry errors — there is no `return_flag` column in the schema to suggest legitimate returns. Excluded via a view, not deleted, to preserve the raw data.

```sql
CREATE VIEW clean_transactions AS
SELECT * FROM transactions WHERE sales_amount > 0;
```

### Issue 3: Multi-currency normalization

Only 4 transactions were in USD. Used a fixed conversion rate of ₹75/USD (approximate average for the 2017–2020 data period).

```sql
CREATE VIEW normalized_transactions AS
SELECT *,
  CASE WHEN currency = 'USD' THEN sales_amount * 75
       ELSE sales_amount
  END AS norm_sales_amount
FROM clean_transactions;
```

### Issue 4: Duplicate transactions

Found 277 groups of exact duplicate pairs — every duplicated row appeared exactly twice with identical values across all columns, indicating import artifacts rather than legitimate repeat orders.

```sql
SELECT dup_count, COUNT(*) AS num_groups
FROM (
  SELECT product_code, customer_code, market_code, order_date,
         sales_qty, sales_amount, currency, COUNT(*) AS dup_count
  FROM clean_transactions
  GROUP BY product_code, customer_code, market_code, order_date,
           sales_qty, sales_amount, currency
  HAVING COUNT(*) > 1
) sub
GROUP BY dup_count;
-- Result: dup_count = 2, num_groups = 277
```

---

## Analysis and key findings

### Q1: Revenue breakdown by city

```sql
SELECT m.markets_name,
       SUM(t.norm_sales_amount) AS total_revenue,
       ROUND(SUM(t.norm_sales_amount) * 100.0 /
             (SELECT SUM(norm_sales_amount) FROM normalized_transactions), 2) AS pct_of_total
FROM normalized_transactions t
JOIN markets m ON t.market_code = m.markets_code
GROUP BY m.markets_name
ORDER BY total_revenue DESC;
```

**Finding:** Delhi NCR alone accounts for 52.79% of total revenue. The top 3 cities (Delhi NCR, Mumbai, Ahmedabad) combine for 81.44%, meaning the remaining 11 markets contribute less than 19%. This is a severe geographic concentration risk. If anything disrupts the Delhi NCR market, the company loses over half its revenue. Markets like Bhubaneshwar (0.09%), Surat (0.26%), and Lucknow (0.31%) are essentially negligible and may not justify the cost of maintaining operations there.

### Q2: Revenue trend by year and month

```sql
SELECT d.year, d.month_name,
       SUM(t.norm_sales_amount) AS monthly_revenue
FROM normalized_transactions t
JOIN date d ON t.order_date = d.date
GROUP BY d.year, d.month_name
ORDER BY d.year, FIELD(d.month_name, 'January','February','March','April','May','June',
                       'July','August','September','October','November','December');
```

### Q3: Top 5 customers by revenue

```sql
SELECT c.custmer_name,
       SUM(t.norm_sales_amount) AS total_revenue,
       SUM(t.sales_qty) AS total_qty
FROM normalized_transactions t
JOIN customers c ON t.customer_code = c.customer_code
GROUP BY c.custmer_name
ORDER BY total_revenue DESC
LIMIT 5;
```

### Q4: Top 5 products by revenue

```sql
SELECT t.product_code, p.product_type,
       SUM(t.norm_sales_amount) AS total_revenue
FROM normalized_transactions t
JOIN products p ON t.product_code = p.product_code
GROUP BY t.product_code, p.product_type
ORDER BY total_revenue DESC
LIMIT 5;
```

### Q5: Year-over-year revenue growth by market

```sql
WITH yearly_revenue AS (
  SELECT m.markets_name, d.year,
         SUM(t.norm_sales_amount) AS revenue
  FROM normalized_transactions t
  JOIN markets m ON t.market_code = m.markets_code
  JOIN date d ON t.order_date = d.date
  GROUP BY m.markets_name, d.year
)
SELECT curr.markets_name, curr.year,
       curr.revenue AS current_year_revenue,
       prev.revenue AS previous_year_revenue,
       ROUND((curr.revenue - prev.revenue) * 100.0 / prev.revenue, 2) AS yoy_growth_pct
FROM yearly_revenue curr
LEFT JOIN yearly_revenue prev
  ON curr.markets_name = prev.markets_name
  AND curr.year = prev.year + 1
WHERE prev.revenue IS NOT NULL
ORDER BY curr.year, yoy_growth_pct;
```

**Finding:** The data tells a three-act story. **2018** was the peak; all 14 markets grew over 100%. **2019** marked the turning point; 9 of 14 markets declined, with Surat (-60%), Chennai (-43%), and Patna (-40%) hit hardest. Delhi NCR fell 22%. **2020** was a collapse; every single market declined. Delhi NCR dropped 55%, going from ₹221M (2018) → ₹171M (2019) → ₹77M (2020) with a 65% decline in two years. The markets that resisted in 2019 (Bhubaneshwar, Hyderabad) also collapsed in 2020, confirming a company-wide problem compounded by COVID.

### Q6: Revenue concentration risk — Pareto analysis

```sql
WITH customer_revenue AS (
  SELECT c.custmer_name,
         SUM(t.norm_sales_amount) AS revenue,
         SUM(SUM(t.norm_sales_amount)) OVER (ORDER BY SUM(t.norm_sales_amount) DESC) AS running_total,
         SUM(SUM(t.norm_sales_amount)) OVER () AS grand_total
  FROM normalized_transactions t
  JOIN customers c ON t.customer_code = c.customer_code
  GROUP BY c.custmer_name
)
SELECT custmer_name, revenue,
       ROUND(revenue * 100.0 / grand_total, 2) AS pct_of_total,
       ROUND(running_total * 100.0 / grand_total, 2) AS cumulative_pct
FROM customer_revenue
ORDER BY revenue DESC
LIMIT 10;
```

**Finding:** Electricalsara Stores alone accounts for 41.95% of total revenue. The top 3 customers combine for 51.96%, and the top 5 reach 61.01%. The gap between #1 (41.95%) and #2 (5.03%) is enormous; losing Electricalsara would be more devastating than losing the bottom 30 customers combined. Combined with the geographic finding (52% from Delhi NCR), it's very likely Electricalsara is the Delhi NCR revenue with a double concentration risk in one city and one customer.

### Q7: Seasonality analysis

```sql
SELECT d.month_name,
       ROUND(AVG(monthly_rev), 2) AS avg_monthly_revenue,
       MIN(monthly_rev) AS worst_year,
       MAX(monthly_rev) AS best_year
FROM (
  SELECT d.year, d.month_name,
         SUM(t.norm_sales_amount) AS monthly_rev
  FROM normalized_transactions t
  JOIN date d ON t.order_date = d.date
  GROUP BY d.year, d.month_name
) sub
JOIN date d ON sub.month_name = d.month_name
GROUP BY d.month_name
ORDER BY avg_monthly_revenue DESC;
```

**Finding:** Moderate seasonality exists with no extreme peaks. August (₹35.8M avg) and July (₹35.7M avg) are the strongest months, suggesting a mid-year buying surge tied to fiscal year budgeting. June is the weakest (₹25M avg) and most unpredictable; its worst year was nearly half its best. The seasonal spread is only ~30%, meaning the year-over-year decline overwhelms any seasonal pattern.

### Q8: Customer retention — cohort analysis

```sql
WITH first_purchase AS (
  SELECT customer_code,
         MIN(order_date) AS first_order,
         YEAR(MIN(order_date)) AS cohort_year
  FROM normalized_transactions
  GROUP BY customer_code
),
cohort_activity AS (
  SELECT fp.cohort_year, d.year AS activity_year,
         COUNT(DISTINCT t.customer_code) AS active_customers
  FROM normalized_transactions t
  JOIN first_purchase fp ON t.customer_code = fp.customer_code
  JOIN date d ON t.order_date = d.date
  GROUP BY fp.cohort_year, d.year
)
SELECT cohort_year, activity_year, active_customers,
       activity_year - cohort_year AS years_since_first_purchase
FROM cohort_activity
ORDER BY cohort_year, activity_year;
```

**Finding:** All 38 customers were acquired in 2017 with 100% retention through 2020, zero churn, but also zero new customers in three years. This reframes the entire revenue decline: the problem isn't customer loss, it's stagnant acquisition combined with shrinking order values. Each of the 38 customers is spending significantly less year over year.

### Q9: Product-market matrix

```sql
SELECT m.markets_name, p.product_type,
       COUNT(DISTINCT t.product_code) AS unique_products,
       SUM(t.norm_sales_amount) AS revenue
FROM normalized_transactions t
JOIN markets m ON t.market_code = m.markets_code
JOIN products p ON t.product_code = p.product_code
GROUP BY m.markets_name, p.product_type
ORDER BY m.markets_name, revenue DESC;
```

**Finding:** Own Brand dominates everywhere. In Delhi NCR, it's ₹157M vs ₹86M for Distribution (65/35 split). Bengaluru is the only market where Distribution revenue exceeds Own Brand by nearly 10x, suggesting a completely different customer profile. Chennai has 33 Own-brand products generating only ₹5.8M vs Delhi NCR's 95 products generating ₹157M — same product type, vastly different per-product revenue.

### Q10: Anomaly detection

```sql
WITH stats AS (
  SELECT AVG(norm_sales_amount) AS avg_amount,
         STDDEV(norm_sales_amount) AS std_amount
  FROM normalized_transactions
)
SELECT t.product_code, t.customer_code, t.market_code,
       t.order_date, t.sales_qty, t.norm_sales_amount,
       ROUND((t.norm_sales_amount - s.avg_amount) / s.std_amount, 2) AS z_score
FROM normalized_transactions t
CROSS JOIN stats s
WHERE ABS((t.norm_sales_amount - s.avg_amount) / s.std_amount) > 3
ORDER BY z_score DESC;
```

**Finding:** 1,734 transactions (~1.2%) have z-scores above 3. The top outlier (z-score 50.02) is a ₹1.5M transaction from Cus020. Cus006 in Mark004 dominates the outliers, appearing in 8 of the top 12 rows — these are likely legitimate bulk orders given consistent customer/market pairing. Cus038 in Mark013 (1,798 units, ₹1.3M) is a one-off that warrants investigation.

---

## Key insights and recommendations

| # | Insight | Recommendation |
|---|---------|----------------|
| 1 | Delhi NCR contributes 53% of revenue — extreme geographic concentration | Invest in growth markets like Mumbai and Ahmedabad; allocate dedicated sales resources to tier-2 cities |
| 2 | Revenue collapsed 65% from peak (2018) to 2020 across all markets | Conduct root-cause analysis separating COVID impact from structural issues; the decline started in 2019 before COVID |
| 3 | Electricalsara Stores = 42% of total revenue; top 3 = 52% | Implement a key account retention program for Electricalsara; aggressively prospect new customers to reduce dependency |
| 4 | Zero new customers acquired from 2018–2020 | The most urgent priority is a customer acquisition strategy — the company retained 100% of customers but failed to grow its base |
| 5 | Seasonality is moderate (July–August peak) but irrelevant compared to the structural decline | Don't optimize for seasonality; focus resources on reversing the revenue decline trajectory |

---
## Data Analysis Using Tableau
### Dashboard 1 — [Revenue Overview](https://public.tableau.com/app/profile/huyen.le7695/viz/AtliQHardware-RevenueOverview/RevenueOverview)
 
[![Revenue Overview](images/AtliQ_Hardware_-_Revenue_Overview.png)](https://public.tableau.com/app/profile/huyen.le7695/viz/AtliQHardware-RevenueOverview/RevenueOverview)
<img width="1209" height="559" alt="image" src="https://github.com/user-attachments/assets/b6c5ba8b-7fbd-4c01-8f40-135052837f95" />

 
### Dashboard 2 — [Market & Growth Analysis](https://public.tableau.com/app/profile/huyen.le7695/viz/AtliQHardware-MarketGrowthAnalysis/AtliQHardware-MarketGrowthAnalysis)
 
[![Market & Growth Analysis](images/AtliQ_Hardware_-_Market___Growth_Analysis.png)](https://public.tableau.com/app/profile/huyen.le7695/viz/AtliQHardware-MarketGrowthAnalysis/AtliQHardware-MarketGrowthAnalysis)
<img width="1200" height="602" alt="image" src="https://github.com/user-attachments/assets/51412fac-d56b-4070-a64f-15fa3682d3de" />

 
### Dashboard 3 — [Risk & Customer Intelligence](https://public.tableau.com/app/profile/huyen.le7695/viz/AtliQHardware-RiskCustomerIntelligence/RiskCustomerIntelligence)
 
[![Risk & Customer Intelligence](images/AtliQ_Hardware_-_Risk___Customer_Intelligence.png)](https://public.tableau.com/app/profile/huyen.le7695/viz/AtliQHardware-RiskCustomerIntelligence/RiskCustomerIntelligence)
<img width="1231" height="566" alt="image" src="https://github.com/user-attachments/assets/6b07bc0e-8468-4fd9-aec0-a99d037e8b01" />


---
## Tools used

| Tool | Purpose |
|---|---|
| MySQL 9.7 | Database engine — data storage, cleaning, and all analytical queries |
| VS Code + MySQL Shell extension | SQL development environment |
| Tableau Public 2026.1 | Interactive dashboard creation and publishing |
| Git and GitHub | Version control and portfolio hosting |

---

