# The Complete DBMS Handbook — Part 4

## 📘 MODULE 14: INDUSTRY CASE STUDIES

> **Continuing from Module 13 (Common SQL Pitfalls).** This module applies everything from Modules 1–13 to realistic, domain-specific problems across eight industries. Each case study includes a schema, the business question, and — where instructive — **multiple solution approaches** with an explicit discussion of their tradeoffs, exactly as you'd be expected to reason in a senior interview.

---

## 14.0 Chapter Roadmap

```
Module 14: Industry Case Studies
│
├── 14.1 Banking
├── 14.2 HR Analytics
├── 14.3 Sales Analytics
├── 14.4 Inventory
├── 14.5 Healthcare
├── 14.6 Social Media
├── 14.7 Recommendation Systems
├── 14.8 Finance
└── 14.9 Exercises
```

---

## 14.1 Banking

**Schema:**
```sql
Accounts(account_id, customer_id, account_type, balance, opened_date)
Transactions(txn_id, account_id, txn_type, amount, txn_date)  -- txn_type: 'DEPOSIT'/'WITHDRAWAL'
```

### Problem 1: Detect potentially fraudulent accounts — 3+ withdrawals exceeding 10,000 within any 24-hour period

**Approach A — Window Function (Preferred):**
```sql
WITH FlaggedWithdrawals AS (
    SELECT
        account_id,
        txn_date,
        amount,
        COUNT(*) OVER (
            PARTITION BY account_id
            ORDER BY txn_date
            RANGE BETWEEN INTERVAL '24 hours' PRECEDING AND CURRENT ROW
        ) AS withdrawals_in_24h
    FROM Transactions
    WHERE txn_type = 'WITHDRAWAL' AND amount > 10000
)
SELECT DISTINCT account_id
FROM FlaggedWithdrawals
WHERE withdrawals_in_24h >= 3;
```

**Approach B — Self-Join (Illustrating the pre-window-function way, Module 3 style):**
```sql
SELECT DISTINCT t1.account_id
FROM Transactions t1
JOIN Transactions t2
    ON t1.account_id = t2.account_id
    AND t2.txn_date BETWEEN t1.txn_date - INTERVAL '24 hours' AND t1.txn_date
WHERE t1.txn_type = 'WITHDRAWAL' AND t1.amount > 10000
  AND t2.txn_type = 'WITHDRAWAL' AND t2.amount > 10000
GROUP BY t1.account_id, t1.txn_id
HAVING COUNT(*) >= 3;
```

**Tradeoff Discussion:** Approach A (window function, Module 5) is more readable and typically far more performant — the self-join in Approach B creates a combinatorial join that scales poorly on large transaction volumes (recall Module 13, Section 13.6's Cartesian-explosion-adjacent risk on self-joins with wide date ranges). **Always prefer the window-function approach for this "N events within a sliding time window" pattern.**

### Problem 2: Running account balance reconciliation

```sql
SELECT
    account_id,
    txn_date,
    amount,
    SUM(CASE WHEN txn_type = 'DEPOSIT' THEN amount ELSE -amount END)
        OVER (PARTITION BY account_id ORDER BY txn_date
              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
FROM Transactions
ORDER BY account_id, txn_date;
```
> Note the explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — a deliberate, defensive frame clause (Module 5 §5.7, Module 12 §12.7) rather than relying on the default, since transaction timestamps could plausibly tie.

**Industry Note:** Banking systems demand strict auditability — recall Module 10's **audit trigger pattern** as the standard mechanism for capturing every balance-affecting change with full before/after context, and Module 9's **transaction + error handling discipline** for atomic multi-step transfers.

---

## 14.2 HR Analytics

**Schema:**
```sql
Employees(emp_id, name, dept_id, manager_id, salary, hire_date, performance_rating)
Departments(dept_id, dept_name, budget)
```

### Problem 1: Employees earning more than their manager

```sql
SELECT e.name AS employee, e.salary AS employee_salary,
       m.name AS manager, m.salary AS manager_salary
FROM Employees e
JOIN Employees m ON e.manager_id = m.emp_id   -- self-join, recall Module 1 §1.7 (Rename/ρ)
WHERE e.salary > m.salary;
```

### Problem 2: Attrition-risk flag — employees with below-average performance AND no raise (salary unchanged) in the last 2 years

*(Assuming a `SalaryChangeLog` table from Module 10's audit trigger pattern.)*
```sql
WITH DeptAvgPerformance AS (
    SELECT dept_id, AVG(performance_rating) AS avg_rating
    FROM Employees
    GROUP BY dept_id
),
RecentlyRaised AS (
    SELECT DISTINCT emp_id
    FROM SalaryChangeLog
    WHERE changed_at > CURRENT_DATE - INTERVAL '2 years'
)
SELECT e.emp_id, e.name
FROM Employees e
JOIN DeptAvgPerformance d ON e.dept_id = d.dept_id
WHERE e.performance_rating < d.avg_rating
  AND e.emp_id NOT IN (SELECT emp_id FROM RecentlyRaised);
-- Safe here: RecentlyRaised.emp_id is a non-nullable FK-sourced column — verify this assumption
-- before using NOT IN in production (Module 1 §1.8.6); NOT EXISTS remains the safer default.
```

### Problem 3: Org-wide headcount and total compensation rollup by department, with a grand total

```sql
SELECT
    COALESCE(d.dept_name, 'ALL DEPARTMENTS') AS dept_name,
    COUNT(e.emp_id) AS headcount,
    SUM(e.salary) AS total_compensation
FROM Employees e
JOIN Departments d ON e.dept_id = d.dept_id
GROUP BY ROLLUP (d.dept_name);
```
> Direct application of **Module 6's `ROLLUP`** — a textbook HR reporting use case.

**Industry Note:** This domain heavily exercises Module 4's **recursive CTEs** (org chart depth, span of control analysis) and Module 6's **advanced aggregation** (compensation bands, headcount rollups).

---

## 14.3 Sales Analytics

**Schema:**
```sql
Sales(sale_id, salesperson_id, region, product_id, sale_date, amount)
Products(product_id, product_name, category)
```

### Problem 1: Month-over-month revenue growth percentage, by region

```sql
WITH MonthlyRevenue AS (
    SELECT region, DATE_TRUNC('month', sale_date) AS sale_month, SUM(amount) AS revenue
    FROM Sales
    GROUP BY region, DATE_TRUNC('month', sale_date)
)
SELECT
    region, sale_month, revenue,
    LAG(revenue) OVER (PARTITION BY region ORDER BY sale_month) AS prev_month_revenue,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (PARTITION BY region ORDER BY sale_month))
        / NULLIF(LAG(revenue) OVER (PARTITION BY region ORDER BY sale_month), 0), 2
    ) AS mom_growth_pct
FROM MonthlyRevenue
ORDER BY region, sale_month;
```
> Notice the **`NULLIF(..., 0)` division-by-zero guard** (Module 12, Section 12.6/12.7) protecting against a region with zero revenue in the prior month.

### Problem 2: Top 3 salespeople per region, per quarter

**Approach A — Window Function:**
```sql
WITH QuarterlySales AS (
    SELECT salesperson_id, region, DATE_TRUNC('quarter', sale_date) AS qtr, SUM(amount) AS total_sales
    FROM Sales
    GROUP BY salesperson_id, region, DATE_TRUNC('quarter', sale_date)
)
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (PARTITION BY region, qtr ORDER BY total_sales DESC) AS rnk
    FROM QuarterlySales
) ranked
WHERE rnk <= 3;
```

**Approach B — LATERAL (Module 3, Section 3.6):**
```sql
SELECT r.region, top_sellers.*
FROM (SELECT DISTINCT region FROM Sales) r
CROSS JOIN LATERAL (
    SELECT salesperson_id, SUM(amount) AS total_sales
    FROM Sales s
    WHERE s.region = r.region
    GROUP BY salesperson_id
    ORDER BY total_sales DESC
    LIMIT 3
) AS top_sellers;
```

**Tradeoff Discussion:** Approach A is generally preferred for its single-pass, set-based clarity; Approach B (`LATERAL`) becomes advantageous if the per-group computation is itself complex/multi-column beyond what a simple window function ranking can express cleanly (Module 3, Section 3.6).

### Problem 3: Product category sales report with subtotals and grand total

```sql
SELECT
    p.category, r.region, SUM(s.amount) AS total_sales,
    GROUPING(p.category) AS is_cat_rollup,
    GROUPING(r.region) AS is_region_rollup
FROM Sales s
JOIN Products p ON s.product_id = p.product_id
CROSS JOIN LATERAL (SELECT s.region AS region) r
GROUP BY CUBE (p.category, r.region);
```
> Direct application of **Module 6's `CUBE` + `GROUPING()`** for a full "every angle" sales dashboard.

---

## 14.4 Inventory

**Schema:**
```sql
Products(product_id, name, category, reorder_threshold)
Warehouses(warehouse_id, name, location)
Stock(product_id, warehouse_id, quantity, last_updated)
StockMovements(movement_id, product_id, warehouse_id, movement_type, quantity, movement_date)
-- movement_type: 'INBOUND' / 'OUTBOUND'
```

### Problem 1: Products below reorder threshold, aggregated across all warehouses

```sql
SELECT p.product_id, p.name, SUM(st.quantity) AS total_stock, p.reorder_threshold
FROM Products p
JOIN Stock st ON p.product_id = st.product_id
GROUP BY p.product_id, p.name, p.reorder_threshold
HAVING SUM(st.quantity) < p.reorder_threshold;
```

### Problem 2: Days of stock remaining, based on trailing 30-day average daily outbound movement

```sql
WITH DailyOutbound AS (
    SELECT product_id, movement_date::date AS day, SUM(quantity) AS daily_out
    FROM StockMovements
    WHERE movement_type = 'OUTBOUND'
      AND movement_date > CURRENT_DATE - INTERVAL '30 days'
    GROUP BY product_id, movement_date::date
),
AvgDailyUsage AS (
    SELECT product_id, AVG(daily_out) AS avg_daily_usage
    FROM DailyOutbound
    GROUP BY product_id
)
SELECT
    p.product_id, p.name,
    SUM(st.quantity) AS current_stock,
    a.avg_daily_usage,
    ROUND(SUM(st.quantity) / NULLIF(a.avg_daily_usage, 0), 1) AS days_remaining
FROM Products p
JOIN Stock st ON p.product_id = st.product_id
JOIN AvgDailyUsage a ON p.product_id = a.product_id
GROUP BY p.product_id, p.name, a.avg_daily_usage;
```
> A multi-CTE decomposition (Module 12, Section 12.2) that would be genuinely unreadable as one nested query — a textbook example of when to decompose.

### Problem 3: Warehouse-to-warehouse stock imbalance (products where one warehouse is overstocked and another is understocked)

```sql
WITH ProductAvgStock AS (
    SELECT product_id, AVG(quantity) AS avg_qty, STDDEV(quantity) AS stddev_qty
    FROM Stock
    GROUP BY product_id
)
SELECT s.product_id, s.warehouse_id, s.quantity, a.avg_qty,
       (s.quantity - a.avg_qty) / NULLIF(a.stddev_qty, 0) AS z_score
FROM Stock s
JOIN ProductAvgStock a ON s.product_id = a.product_id
WHERE ABS((s.quantity - a.avg_qty) / NULLIF(a.stddev_qty, 0)) > 2
ORDER BY z_score DESC;
```

---

## 14.5 Healthcare

**Schema:**
```sql
Patients(patient_id, name, dob, gender)
Appointments(appointment_id, patient_id, doctor_id, appt_date, status)
Prescriptions(prescription_id, patient_id, drug_name, prescribed_date, dosage)
```

### Problem 1: Patients with overlapping/conflicting drug prescriptions on the same date

```sql
SELECT p1.patient_id, p1.drug_name AS drug_1, p2.drug_name AS drug_2, p1.prescribed_date
FROM Prescriptions p1
JOIN Prescriptions p2
    ON p1.patient_id = p2.patient_id
    AND p1.prescribed_date = p2.prescribed_date
    AND p1.prescription_id < p2.prescription_id   -- ◀ avoids duplicate pairs & self-pairing
WHERE p1.drug_name <> p2.drug_name;
```
> The `p1.prescription_id < p2.prescription_id` trick is a standard pattern for **avoiding both self-pairs and mirror-duplicate pairs** in a self-join — worth memorizing as a reusable idiom.

### Problem 2: No-show rate per doctor over the last 6 months

```sql
SELECT
    doctor_id,
    COUNT(*) FILTER (WHERE status = 'NO_SHOW') AS no_shows,   -- Module 6 §6.6, FILTER clause
    COUNT(*) AS total_appointments,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status = 'NO_SHOW') / COUNT(*), 2) AS no_show_rate_pct
FROM Appointments
WHERE appt_date > CURRENT_DATE - INTERVAL '6 months'
GROUP BY doctor_id
ORDER BY no_show_rate_pct DESC;
```

### Problem 3: Patients overdue for a follow-up (last appointment > 12 months ago, but had 2+ appointments historically)

```sql
WITH PatientAppointmentStats AS (
    SELECT patient_id, MAX(appt_date) AS last_visit, COUNT(*) AS total_visits
    FROM Appointments
    WHERE status = 'COMPLETED'
    GROUP BY patient_id
)
SELECT patient_id, last_visit, total_visits
FROM PatientAppointmentStats
WHERE last_visit < CURRENT_DATE - INTERVAL '12 months'
  AND total_visits >= 2;
```

> ⚠️ **Healthcare Data Note (Ethical/Privacy Reminder):** Real healthcare schemas involve PHI (Protected Health Information) subject to strict regulatory requirements (e.g., HIPAA in the US). Access-controlling queries via **Views** (Module 8) that expose only de-identified/necessary columns, and **audit triggers** (Module 10) logging every access to sensitive records, are standard, expected practice in this domain — not optional extras.

---

## 14.6 Social Media

**Schema:**
```sql
Users(user_id, username, signup_date)
Posts(post_id, user_id, content, posted_at)
Follows(follower_id, followee_id, followed_at)
Likes(like_id, post_id, user_id, liked_at)
```

### Problem 1: Mutual followers ("friends" — both follow each other)

```sql
SELECT f1.follower_id AS user_a, f1.followee_id AS user_b
FROM Follows f1
JOIN Follows f2
    ON f1.follower_id = f2.followee_id
    AND f1.followee_id = f2.follower_id
WHERE f1.follower_id < f1.followee_id;   -- avoids duplicate mirror pairs (same idiom as 14.5)
```

### Problem 2: Users with the highest engagement rate (likes received per post, last 30 days)

```sql
WITH RecentPosts AS (
    SELECT post_id, user_id
    FROM Posts
    WHERE posted_at > CURRENT_DATE - INTERVAL '30 days'
),
PostLikeCounts AS (
    SELECT rp.user_id, rp.post_id, COUNT(l.like_id) AS like_count
    FROM RecentPosts rp
    LEFT JOIN Likes l ON rp.post_id = l.post_id
    GROUP BY rp.user_id, rp.post_id
)
SELECT user_id, AVG(like_count) AS avg_likes_per_post, COUNT(*) AS post_count
FROM PostLikeCounts
GROUP BY user_id
ORDER BY avg_likes_per_post DESC
LIMIT 10;
```

### Problem 3: "Friends of friends" recommendation (users followed by people you follow, but not followed by you, and not yourself) — Recursive/Graph Pattern

```sql
WITH DirectFollows AS (
    SELECT followee_id FROM Follows WHERE follower_id = 123   -- target user_id = 123
),
FriendsOfFriends AS (
    SELECT DISTINCT f2.followee_id
    FROM Follows f1
    JOIN Follows f2 ON f1.followee_id = f2.follower_id
    WHERE f1.follower_id = 123
)
SELECT followee_id AS recommended_user_id
FROM FriendsOfFriends
WHERE followee_id <> 123
  AND followee_id NOT IN (SELECT followee_id FROM DirectFollows);
```
> Note this is a **fixed 2-hop traversal** (via a self-join, Module 1 §1.7), not a full recursive CTE (Module 4) — appropriate since "friends of friends" is a fixed depth-2 relationship, not an unbounded-depth traversal. Contrast with Module 4's flight-routes example, which genuinely needed unbounded-depth recursion.

---

## 14.7 Recommendation Systems

**Schema:**
```sql
Users(user_id, name)
Products(product_id, name, category)
Purchases(purchase_id, user_id, product_id, purchase_date)
```

### Problem 1: "Customers who bought X also bought Y" (Market Basket Analysis)

```sql
WITH TargetProductBuyers AS (
    SELECT DISTINCT user_id FROM Purchases WHERE product_id = 501   -- target product
)
SELECT p.product_id, pr.name, COUNT(DISTINCT p.user_id) AS co_purchase_count
FROM Purchases p
JOIN TargetProductBuyers t ON p.user_id = t.user_id
JOIN Products pr ON p.product_id = pr.product_id
WHERE p.product_id <> 501
GROUP BY p.product_id, pr.name
ORDER BY co_purchase_count DESC
LIMIT 10;
```

### Problem 2: Products purchased by EVERY customer in a given segment (Division pattern, callback to Module 1 §1.9)

```sql
WITH VIPCustomers AS (
    SELECT user_id FROM Users WHERE user_id IN (SELECT user_id FROM CustomerSegments WHERE segment = 'VIP')
)
SELECT product_id
FROM Purchases
WHERE user_id IN (SELECT user_id FROM VIPCustomers)
GROUP BY product_id
HAVING COUNT(DISTINCT user_id) = (SELECT COUNT(*) FROM VIPCustomers);
```
> Direct, real-world application of **Module 1's Division operator pattern** — "products bought by ALL VIP customers."

### Problem 3: Personalized "trending in your category" — top products in categories the user has purchased from before, excluding products they already own

```sql
WITH UserCategories AS (
    SELECT DISTINCT pr.category
    FROM Purchases p JOIN Products pr ON p.product_id = pr.product_id
    WHERE p.user_id = 42
),
UserOwnedProducts AS (
    SELECT DISTINCT product_id FROM Purchases WHERE user_id = 42
),
CategoryTrending AS (
    SELECT pr.product_id, pr.category, COUNT(*) AS purchase_count,
           RANK() OVER (PARTITION BY pr.category ORDER BY COUNT(*) DESC) AS rnk
    FROM Purchases p
    JOIN Products pr ON p.product_id = pr.product_id
    WHERE pr.category IN (SELECT category FROM UserCategories)
    GROUP BY pr.product_id, pr.category
)
SELECT product_id, category, purchase_count
FROM CategoryTrending
WHERE rnk <= 5
  AND product_id NOT IN (SELECT product_id FROM UserOwnedProducts);
```

---

## 14.8 Finance

**Schema:**
```sql
Stocks(symbol, company_name, sector)
DailyPrices(symbol, trade_date, close_price, volume)
Portfolios(portfolio_id, user_id, symbol, shares, purchase_price, purchase_date)
```

### Problem 1: 7-day moving average closing price per stock

```sql
SELECT symbol, trade_date, close_price,
       AVG(close_price) OVER (
           PARTITION BY symbol ORDER BY trade_date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS moving_avg_7d
FROM DailyPrices
ORDER BY symbol, trade_date;
```
> A canonical **window-frame** application (Module 5, Section 5.7) — `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` gives exactly 7 rows (today + 6 prior), a very common finance/time-series pattern.

### Problem 2: Portfolio unrealized gain/loss, ranked by percentage return

```sql
WITH LatestPrices AS (
    SELECT DISTINCT ON (symbol) symbol, close_price
    FROM DailyPrices
    ORDER BY symbol, trade_date DESC       -- PostgreSQL DISTINCT ON idiom for "latest per group"
)
SELECT
    pf.portfolio_id, pf.symbol, pf.shares, pf.purchase_price, lp.close_price AS current_price,
    (lp.close_price - pf.purchase_price) * pf.shares AS unrealized_gain_loss,
    ROUND(100.0 * (lp.close_price - pf.purchase_price) / NULLIF(pf.purchase_price, 0), 2) AS pct_return,
    RANK() OVER (ORDER BY (lp.close_price - pf.purchase_price) / NULLIF(pf.purchase_price, 0) DESC) AS return_rank
FROM Portfolios pf
JOIN LatestPrices lp ON pf.symbol = lp.symbol;
```
> **`DISTINCT ON`** is a PostgreSQL-specific extension for "one row per group, by some order" — functionally similar to a `ROW_NUMBER() = 1` filter (Module 5), but a useful, engine-specific idiom to know exists (MySQL requires the window-function approach instead, since it lacks `DISTINCT ON`).

### Problem 3: Sector-level portfolio concentration risk (percentage of total portfolio value per sector)

```sql
WITH LatestPrices AS (
    SELECT DISTINCT ON (symbol) symbol, close_price FROM DailyPrices ORDER BY symbol, trade_date DESC
),
PortfolioValues AS (
    SELECT pf.portfolio_id, s.sector, pf.shares * lp.close_price AS position_value
    FROM Portfolios pf
    JOIN Stocks s ON pf.symbol = s.symbol
    JOIN LatestPrices lp ON pf.symbol = lp.symbol
)
SELECT
    portfolio_id, sector, SUM(position_value) AS sector_value,
    ROUND(100.0 * SUM(position_value) / SUM(SUM(position_value)) OVER (PARTITION BY portfolio_id), 2) AS pct_of_portfolio
FROM PortfolioValues
GROUP BY portfolio_id, sector
ORDER BY portfolio_id, pct_of_portfolio DESC;
```
> Notice the **nested aggregate + window function** (`SUM(SUM(...)) OVER (...)`) — a genuinely advanced pattern combining `GROUP BY`-level aggregation (Module 6) with a window function computed *on top of* the grouped result (Module 5), used here to compute "this sector's share of the total portfolio."

---

## 14.9 Exercises

### Mini Projects (One Per Domain — Apply Multiple Modules Together)
1. **Banking:** Design and query a fraud-detection system flagging accounts with 5+ transactions in any rolling 1-hour window, using a window function.
2. **HR Analytics:** Build a full `ROLLUP`-based compensation report by department and job level, with `GROUPING()`-labeled subtotal rows.
3. **Sales Analytics:** Write a recursive CTE-based territory hierarchy report (region → sub-region → sales rep) showing rolled-up sales at each level.
4. **Inventory:** Design a `LATERAL`-based query finding the top 2 fastest-moving (highest outbound volume) products per warehouse.
5. **Healthcare:** Write an audit-trigger design (Module 10 pattern) for tracking every access to a `Prescriptions` table, and a query to detect unusual read-access frequency spikes.
6. **Social Media:** Write a recursive CTE to find all users within 3 degrees of separation from a given user (true unbounded-depth graph traversal, contrasting with Section 14.6 Problem 3's fixed 2-hop version).
7. **Recommendation Systems:** Extend the "bought X also bought Y" query to compute a simple co-purchase strength score (co-purchase count / total purchases of X), and rank recommendations by that score.
8. **Finance:** Write a query detecting a "golden cross" event (a stock's 50-day moving average crossing above its 200-day moving average) using two window functions and a `LAG()`-based day-over-day comparison.

### Practice Links
- [LeetCode Database Problems](https://leetcode.com/studyplan/top-sql-50/)
- [DataLemur — SQL Interview Questions](https://datalemur.com/questions?category=SQL)
- [StrataScratch — SQL Interview Questions](https://www.stratascratch.com/)

---

## Chapter Summary

Every case study in this module deliberately **reuses constructs from Modules 1–13** in realistic combinations: window functions for time-series and ranking patterns (Banking, Sales, Finance), CT[Es for multi-stage decomposition (Inventory, HR), the Division pattern for "bought by everyone" logic (Recommendation Systems), self-join idioms for pairwise relationships (Healthcare, Social Media), `ROLLUP`/`CUBE`/`GROUPING()` for reporting (HR, Sales), and defensive `NULLIF`/`NOT EXISTS` patterns throughout. **The real skill this module builds is pattern recognition** — mapping a business question to the right combination of SQL tools, the single most valuable skill tested in senior SQL interviews.

---

## Cross-References for Later Parts / Modules
- **Interview Mastery — timed, structured practice applying these same patterns under interview conditions** → Module 15 (next)
- **Indexing strategy for the window-function and self-join-heavy queries in this module** → Part 8
- **Data Warehousing / star schema design for the reporting-heavy queries (Sales, HR)** → later Parts

---

> ✅ **Module 14 Complete.**
> Reply **"next"**, **"continue"**, or **"Module 15"** to proceed to **Module 15: Interview Mastery** — frequently asked Advanced SQL questions, window function/CTE/ranking/recursive-SQL question banks, query optimization basics, output prediction, scenario-based questions, and real interview coding rounds.
