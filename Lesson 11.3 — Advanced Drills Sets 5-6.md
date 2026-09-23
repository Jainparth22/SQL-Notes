# **SQL Advanced Interview Drill — Set 5**

**Level:** Advanced OA + SDE interviews\
**Dialect:** PostgreSQL\
**Focus:** Problems where knowing only `SELECT`, `JOIN`, and `GROUP BY` is not enough.

In this set, we’ll cover six important patterns: **relational division, gaps and islands, median calculation, cohort retention, rolling time-window analytics, and recursive CTEs**.

The goal is not just to get the correct output. It is to understand why a solution works, know an alternative approach, and explain the trade-offs when an interviewer asks you to solve it differently.

## **1. Relational division — customers who purchased every product**

**Problem:** Find customers who have purchased every product available in the `products` table at least once.

Assume these tables:

```
customers (
    customer_id,
    customer_name
);

products (
    product_id,
    product_name
);

orders (
    order_id,
    customer_id,
    order_date
);

order_items (
    order_id,
    product_id,
    quantity
);
```

This is called **relational division**: find entities that are related to *all* members of another set.

### **Approach 1: Double&#x20;**`NOT EXISTS`

```
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM products p
    WHERE NOT EXISTS (
        SELECT 1
        FROM orders o
        JOIN order_items oi
            ON oi.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
          AND oi.product_id = p.product_id
    )
);
```

Read it from the inside out:

1. The innermost query checks whether customer `c` purchased product `p`.
2. The inner `NOT EXISTS` finds products that the customer **has not** purchased.
3. The outer `NOT EXISTS` requires that there are **no products** the customer has not purchased.

In plain English:

> Return customers for whom no unpurchased product exists.

### **Approach 2:&#x20;**`GROUP BY`**&#x20;+&#x20;**`HAVING`

```
SELECT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
LEFT JOIN order_items oi
    ON oi.order_id = o.order_id
LEFT JOIN products p
    ON p.product_id = oi.product_id
GROUP BY c.customer_id, c.customer_name
HAVING COUNT(DISTINCT p.product_id) = (
    SELECT COUNT(*)
    FROM products
);
```

Why `COUNT(DISTINCT)`?

A customer may purchase the same product in multiple orders. Counting rows would count repeated purchases, not unique products.

Why use `LEFT JOIN`?

It preserves customers who have made no purchases. This also handles an empty product catalogue: if there are zero products, every customer satisfies the requirement mathematically.

**Interview takeaway:** Double `NOT EXISTS` is a very natural expression of “has every item.” `GROUP BY` + `HAVING` is often easier to explain when the problem is phrased as comparing distinct counts.

## **2. Gaps and islands — find consecutive login days**

**Problem:** Find every period in which a user logged in for at least three consecutive days.

Assume:

```
user_logins (
    user_id,
    login_date
);
```

Example input:

| user_id | login_date |
| ------- | ---------- |
| 101     | 2026-11-01 |
| 101     | 2026-11-02 |
| 101     | 2026-11-03 |
| 101     | 2026-11-05 |
| 101     | 2026-11-06 |
| 101     | 2026-11-07 |
| 102     | 2026-11-01 |
| 102     | 2026-11-03 |

Expected result:

| user_id | streak_start | streak_end | consecutive_days |
| ------- | ------------ | ---------- | ---------------- |
| 101     | 2026-11-01   | 2026-11-03 | 3                |
| 101     | 2026-11-05   | 2026-11-07 | 3                |

This is a classic **gaps and islands** problem.

- An *island* is a consecutive sequence.
- A *gap* is a break in that sequence.

### **Approach 1:&#x20;**`ROW_NUMBER()`**&#x20;grouping trick**

```
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date
    FROM user_logins
),
numbered AS (
    SELECT
        user_id,
        login_date,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY login_date
        ) AS rn
    FROM distinct_logins
),
grouped AS (
    SELECT
        user_id,
        login_date,
        login_date - rn::int AS grp
    FROM numbered
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS consecutive_days
FROM grouped
GROUP BY user_id, grp
HAVING COUNT(*) >= 3
ORDER BY user_id, streak_start;
```

**Why does this work?**

Consider user 101:

| login_date | row_number | login_date − row_number |
| ---------- | ---------- | ----------------------- |
| Nov 1      | 1          | Oct 31                  |
| Nov 2      | 2          | Oct 31                  |
| Nov 3      | 3          | Oct 31                  |
| Nov 5      | 4          | Nov 1                   |
| Nov 6      | 5          | Nov 1                   |
| Nov 7      | 6          | Nov 1                   |

Notice something interesting:

For consecutive dates, subtracting the row number produces the same date.

That constant value becomes the group identifier.

### **Approach 2:&#x20;**`LAG()`**&#x20;+ cumulative&#x20;**`SUM()`

```
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date
    FROM user_logins
),
previous_dates AS (
    SELECT
        user_id,
        login_date,
        LAG(login_date) OVER (
            PARTITION BY user_id
            ORDER BY login_date
        ) AS previous_date
    FROM distinct_logins
),
marked AS (
    SELECT
        user_id,
        login_date,
        CASE
            WHEN previous_date = login_date - 1 THEN 0
            ELSE 1
        END AS new_group
    FROM previous_dates
),
grouped AS (
    SELECT
        user_id,
        login_date,
        SUM(new_group) OVER (
            PARTITION BY user_id
            ORDER BY login_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS grp
    FROM marked
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS consecutive_days
FROM grouped
GROUP BY user_id, grp
HAVING COUNT(*) >= 3
ORDER BY user_id, streak_start;
```

Here, `LAG()` checks the previous login date. Whenever the current date does not immediately follow it, we mark the beginning of a new group.

The cumulative `SUM()` assigns a different group number to each island.

**Important edge cases:**

- Deduplicate `(user_id, login_date)` first. Otherwise, multiple logins on one day can distort the streak.
- `COUNT(*) >= 3` finds streaks of three or more days.
- For exactly three days, use `COUNT(*) = 3`.
- For timestamp-based activity, decide first whether the streak means consecutive calendar days or consecutive 24-hour periods.

**Interview takeaway:** The `ROW_NUMBER()` method is compact; the `LAG()` method makes the boundary between islands more explicit.

## **3. Median salary per department**

**Problem:** Find the median salary in each department.

Assume:

```
employees (
    emp_id,
    name,
    department_id,
    salary
);
```

First, understand the difference between average and median.

For these salaries:

```
10000, 12000, 15000, 18000, 90000
```

Average:

\\[ \frac{10000+12000+15000+18000+90000}{5}=29000 \\]

Median:

\\[ 15000 \\]

The median is the middle value after sorting.

For an even number of values, it is the average of the two middle values.

### **Approach 1:&#x20;**`PERCENTILE_CONT()`

PostgreSQL provides an ordered-set aggregate for this.

```
SELECT
    department_id,
    PERCENTILE_CONT(0.5)
        WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
WHERE salary IS NOT NULL
GROUP BY department_id;
```

`0.5` represents the 50th percentile.

Two related functions:

| Function               | Behaviour                                                            |
| ---------------------- | -------------------------------------------------------------------- |
| `PERCENTILE_CONT(0.5)` | Calculates the continuous percentile; can interpolate between values |
| `PERCENTILE_DISC(0.5)` | Returns an actual value from the ordered data                        |

Example:

```
Salaries: 10, 20, 30, 40
```

The continuous median is:

\\[ \frac{20+30}{2}=25 \\]

The discrete median returns an actual observed salary.

```
SELECT
    department_id,
    PERCENTILE_DISC(0.5)
        WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
WHERE salary IS NOT NULL
GROUP BY department_id;
```

### **Approach 2:&#x20;**`ROW_NUMBER()`**&#x20;+&#x20;**`COUNT()`

This approach is useful when the database does not provide a convenient percentile function.

```
WITH ranked AS (
    SELECT
        department_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary
        ) AS rn,
        COUNT(*) OVER (
            PARTITION BY department_id
        ) AS total_rows
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT
    department_id,
    AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (
    (total_rows + 1) / 2,
    (total_rows + 2) / 2
)
GROUP BY department_id;
```

Understand the formula:

| Number of values | Middle position(s) | Result                      |
| ---------------- | ------------------ | --------------------------- |
| 5                | 3                  | Third value                 |
| 4                | 2 and 3            | Average of second and third |
| 3                | 2                  | Second value                |
| 2                | 1 and 2            | Average of both             |

For integer division:

- When `total_rows = 5`, positions are `3` and `3`.
- When `total_rows = 4`, positions are `2` and `3`.

The `AVG()` therefore calculates the correct median in both cases.

**Interview takeaway:** Know both percentile functions and the ranking-based solution. The latter demonstrates that you understand how median is calculated rather than just knowing a built-in function.

## **4. Customer cohort retention analysis**

**Problem:** Calculate monthly customer retention.

A customer belongs to a cohort based on the month of their first purchase.

For example:

- A customer's first purchase was in January.
- They purchased again in February.
- They purchased again in April.

That customer belongs to the January cohort and is active in January, February and April.

Assume:

```
orders (
    order_id,
    customer_id,
    order_date
);
```

We want:

| cohort_month | activity_month | month_number | retained_customers |
| ------------ | -------------- | ------------ | ------------------ |
| Jan 2026     | Jan 2026       | 0            | 100                |
| Jan 2026     | Feb 2026       | 1            | 70                 |
| Jan 2026     | Mar 2026       | 2            | 55                 |
| Jan 2026     | Apr 2026       | 3            | 40                 |

### **Step 1: Identify each customer's cohort**

```
WITH customer_cohort AS (
    SELECT
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY customer_id
)
SELECT *
FROM customer_cohort;
```

`MIN(order_date)` gives the first purchase date. `DATE_TRUNC()` converts it to the first day of that month.

### **Step 2: Identify customer activity by month**

```
WITH customer_activity AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS activity_month
    FROM orders
)
SELECT *
FROM customer_activity;
```

Why `DISTINCT`?

A customer might place five orders in February. For retention, that normally counts as one active customer in February, not five.

### **Step 3: Build the complete retention calculation**

```
WITH customer_cohort AS (
    SELECT
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY customer_id
),
customer_activity AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS activity_month
    FROM orders
),
cohort_sizes AS (
    SELECT
        cohort_month,
        COUNT(*) AS cohort_size
    FROM customer_cohort
    GROUP BY cohort_month
),
retention AS (
    SELECT
        cc.cohort_month,
        ca.activity_month,
        (
            EXTRACT(YEAR FROM ca.activity_month)
            - EXTRACT(YEAR FROM cc.cohort_month)
        ) * 12
        + (
            EXTRACT(MONTH FROM ca.activity_month)
            - EXTRACT(MONTH FROM cc.cohort_month)
        ) AS month_number,
        COUNT(DISTINCT ca.customer_id) AS retained_customers
    FROM customer_cohort cc
    JOIN customer_activity ca
        ON ca.customer_id = cc.customer_id
       AND ca.activity_month >= cc.cohort_month
    GROUP BY
        cc.cohort_month,
        ca.activity_month
)
SELECT
    r.cohort_month,
    r.activity_month,
    r.month_number,
    r.retained_customers,
    cs.cohort_size,
    ROUND(
        100.0 * r.retained_customers
        / NULLIF(cs.cohort_size, 0),
        2
    ) AS retention_percentage
FROM retention r
JOIN cohort_sizes cs
    ON cs.cohort_month = r.cohort_month
ORDER BY r.cohort_month, r.month_number;
```

### **Understand the calculation**

\\[ \text{Retention Rate}= \frac{\text{Customers active in the period}} {\text{Original cohort size}}\times100 \\]

For a cohort of 100 customers:

- 70 active in month 1 → 70% retention.
- 55 active in month 2 → 55% retention.
- 40 active in month 3 → 40% retention.

`NULLIF(cohort_size, 0)` prevents division by zero.

**Interview clarification:** This query calculates monthly activity retention. It does not calculate whether customers made a purchase in *every* intervening month. That stricter requirement is sometimes called consecutive-month retention.

## **5. Seven-day rolling revenue**

**Problem:** Calculate each store's revenue over the current day and previous six calendar days.

Assume:

```
sales (
    sale_id,
    store_id,
    sale_date,
    amount
);
```

Example:

| store_id | sale_date | amount |
| -------- | --------- | ------ |
| 1        | Nov 1     | 100    |
| 1        | Nov 2     | 200    |
| 1        | Nov 3     | 150    |
| 1        | Nov 5     | 300    |

### **First: Understand&#x20;**`ROWS`**&#x20;versus&#x20;**`RANGE`

This is a very important interview distinction.

```
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
```

Means: current row plus the preceding six rows.

It does **not** necessarily mean seven calendar days.

```
RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
```

Means: include rows within the previous six calendar days, based on the ordering value.

When dates are missing, these produce different results.

### **Approach: Aggregate daily and use&#x20;**`RANGE`

```
WITH daily_revenue AS (
    SELECT
        store_id,
        sale_date::date AS sale_day,
        SUM(amount) AS daily_total
    FROM sales
    GROUP BY store_id, sale_date::date
)
SELECT
    store_id,
    sale_day,
    daily_total,
    SUM(daily_total) OVER (
        PARTITION BY store_id
        ORDER BY sale_day::timestamp
        RANGE BETWEEN INTERVAL '6 days' PRECEDING
                  AND CURRENT ROW
    ) AS rolling_7_day_revenue
FROM daily_revenue
ORDER BY store_id, sale_day;
```

Why aggregate first?

A store can have many transactions on a single day. We first turn them into one daily revenue row, then calculate the rolling total.

Why `PARTITION BY store_id`?

Each store must have an independent rolling total.

### **When should you use&#x20;**`ROWS`**?**

For a rolling total over the last seven records, regardless of calendar gaps:

```
SUM(daily_total) OVER (
    PARTITION BY store_id
    ORDER BY sale_day
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
)
```

For a seven-calendar-day report that must display dates with no sales, build a calendar date series and left join the daily totals, replacing missing revenue with zero using `COALESCE()`.

**Interview takeaway:** Whenever you see “last seven days,” clarify whether the interviewer means seven calendar days or seven records. This one distinction changes the SQL.

## **6. Recursive CTE — employee hierarchy**

**Problem:** Given employees and their managers, find every subordinate under a particular manager, including each employee's hierarchy depth.

Assume:

```
employees (
    employee_id,
    name,
    manager_id
);
```

Example:

| employee_id | name        | manager_id |
| ----------- | ----------- | ---------- |
| 1           | CEO         | NULL       |
| 2           | Director A  | 1          |
| 3           | Director B  | 1          |
| 4           | Manager A1  | 2          |
| 5           | Developer A | 4          |
| 6           | Developer B | 4          |
| 7           | Manager B1  | 3          |

Hierarchy:

**CEO (1)**

**Director A (2)**

**Manager A1 (4)**

Developer A (5)

Developer B (6)

**Director B (3)**

**Manager B1 (7)**

Simplified reporting hierarchy. Arrows represent manager-to-subordinate relationships.

### **Approach:&#x20;**`WITH RECURSIVE`

Suppose we want everyone reporting under employee 1.

```
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor: start with the selected manager
    SELECT
        employee_id,
        name,
        manager_id,
        0 AS depth,
        ARRAY[employee_id] AS path
    FROM employees
    WHERE employee_id = 1

    UNION ALL

    -- Recursive member: find direct reports
    SELECT
        e.employee_id,
        e.name,
        e.manager_id,
        eh.depth + 1,
        eh.path || e.employee_id
    FROM employees e
    JOIN employee_hierarchy eh
        ON e.manager_id = eh.employee_id
    WHERE NOT (e.employee_id = ANY(eh.path))
)
SELECT
    employee_id,
    name,
    manager_id,
    depth,
    path
FROM employee_hierarchy
ORDER BY path;
```

### **Understand the two parts**

**Part 1 — Anchor query**

```
SELECT ...
FROM employees
WHERE employee_id = 1
```

This establishes the starting point.

**Part 2 — Recursive query**

```
SELECT ...
FROM employees e
JOIN employee_hierarchy eh
    ON e.manager_id = eh.employee_id
```

It finds employees whose manager is already in the hierarchy.

The process repeats until no more employees are found.

### **Important columns**

| Column      | Purpose                                                   |
| ----------- | --------------------------------------------------------- |
| `depth`     | Distance from the starting manager                        |
| `path`      | Sequence of employee IDs traversed                        |
| `UNION ALL` | Adds the next level to the recursive result               |
| `ANY(path)` | Checks whether an employee ID already appears in the path |

The condition:

```
WHERE NOT (e.employee_id = ANY(eh.path))
```

prevents revisiting an employee already in the current path. This is a useful cycle safeguard for malformed hierarchy data.

To return only subordinates and exclude the starting manager:

```
SELECT *
FROM employee_hierarchy
WHERE depth > 0;
```

**Interview takeaway:** Recursive CTEs are used for org charts, category trees, folder structures, bill-of-materials expansion, and graph-like traversal.

# **Set 5 — Interview revision sheet**

| Pattern             | Main technique                  | What it solves                 |
| ------------------- | ------------------------------- | ------------------------------ |
| Relational division | Double `NOT EXISTS`             | Must have all required items   |
| Gaps and islands    | `ROW_NUMBER()` or `LAG()`       | Consecutive dates or sequences |
| Median              | `PERCENTILE_CONT()` or ranking  | Middle value in a group        |
| Cohort retention    | CTEs + `DATE_TRUNC()`           | Customer activity over time    |
| Rolling analytics   | Window frames: `ROWS` / `RANGE` | Moving totals and averages     |
| Hierarchies         | `WITH RECURSIVE`                | Parent-child traversal         |

## **Set 5 — Try these without looking at the solutions**

These are deliberately designed to test whether you can adapt a pattern rather than simply remember a query.

**Q1. Relational division — Medium**

Find customers who purchased every product in category `'Electronics'`, not every product in the complete catalogue.

**Q2. Gaps and islands — Medium**

Find the longest consecutive login streak for every user. Return the user ID, starting date, ending date and streak length.

**Q3. Median — Medium**

Find the department whose median salary is highest. Return the department ID and median salary.

**Q4. Cohort analysis — Hard**

Find the percentage of customers from each cohort who returned in the immediately following calendar month. A customer who returns two months later must not count as retained for the next month.

**Q5. Rolling analytics — Hard**

Find the highest seven-calendar-day revenue for each store and the date on which that seven-day period ends.

**Q6. Recursive CTE — Hard**

Find the number of descendants under every manager, excluding the manager themselves.

## **What comes next: Set 6**

The next installment will move from individual SQL patterns to **complex multi-step interview problems**:

1. Detect consecutive transactions that cross a monetary threshold.
2. Find users who performed event A before event B within a specified time window.
3. Calculate session durations from event logs.
4. Find the first and last transaction in each user's activity period.
5. Solve retention and churn problems with window functions.
6. Debug queries that produce incorrect results because of joins, NULLs, duplicates, and window frames.

One important rule for your interviews: when you solve a problem, always be ready to explain the **assumptions, edge cases, and why your query returns the intended rows**. That is what separates memorising SQL syntax from being able to solve SQL problems independently.


# **SQL Advanced Interview Drill — Set 6**

**Level:** Advanced SQL for SDE interviews and placement OAs\
**Dialect:** PostgreSQL\
**Focus:** Multi-step problems involving event data, transaction sequences, sessionisation, retention, and debugging.

This set is about combining techniques. In real interviews, a question may require `LAG()`, a window function, a CTE, aggregation and a join in the same solution.

We'll solve six problems and examine the reasoning behind them.

## **1. Detect consecutive transactions crossing a threshold**

**Problem:** Find users whose latest three consecutive transactions have a combined amount of at least ₹1,000.

Assume:

```
transactions (
    txn_id,
    user_id,
    txn_time,
    amount
);
```

Example:

| user_id | txn_id | amount |
| ------- | ------ | ------ |
| 101     | 1      | 200    |
| 101     | 2      | 300    |
| 101     | 3      | 600    |
| 101     | 4      | 100    |
| 102     | 5      | 200    |
| 102     | 6      | 200    |
| 102     | 7      | 300    |

For user 101, the first three transactions total ₹1,100. The next three total ₹1,000.

### **Approach 1: Rolling window with&#x20;**`SUM()`

```
WITH transaction_windows AS (
    SELECT
        user_id,
        txn_id,
        txn_time,
        amount,
        SUM(amount) OVER (
            PARTITION BY user_id
            ORDER BY txn_time, txn_id
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS last_three_total,
        COUNT(*) OVER (
            PARTITION BY user_id
            ORDER BY txn_time, txn_id
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ) AS transaction_count
    FROM transactions
)
SELECT
    user_id,
    txn_id,
    txn_time,
    last_three_total
FROM transaction_windows
WHERE transaction_count = 3
  AND last_three_total >= 1000
ORDER BY user_id, txn_time;
```

**Understand the key part:**

```
ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
```

For every transaction, SQL considers:

- The current transaction.
- The immediately previous transaction.
- The transaction before that.

This is a three-row sliding window.

Why do we need `transaction_count = 3`? Without it, the first and second rows for each user would represent incomplete windows.

### **Approach 2: Self-join**

```
SELECT
    t1.user_id,
    t3.txn_id AS ending_txn_id,
    t1.amount + t2.amount + t3.amount AS total_amount
FROM transactions t1
JOIN transactions t2
    ON t2.user_id = t1.user_id
JOIN transactions t3
    ON t3.user_id = t1.user_id
WHERE (t1.txn_time, t1.txn_id)
        < (t2.txn_time, t2.txn_id)
  AND (t2.txn_time, t2.txn_id)
        < (t3.txn_time, t3.txn_id)
  AND NOT EXISTS (
      SELECT 1
      FROM transactions middle
      WHERE middle.user_id = t1.user_id
        AND (middle.txn_time, middle.txn_id)
              > (t1.txn_time, t1.txn_id)
        AND (middle.txn_time, middle.txn_id)
              < (t2.txn_time, t2.txn_id)
  )
  AND NOT EXISTS (
      SELECT 1
      FROM transactions middle
      WHERE middle.user_id = t1.user_id
        AND (middle.txn_time, middle.txn_id)
              > (t2.txn_time, t2.txn_id)
        AND (middle.txn_time, middle.txn_id)
              < (t3.txn_time, t3.txn_id)
  )
  AND t1.amount + t2.amount + t3.amount >= 1000;
```

This checks that no other transaction exists between the three selected transactions. It is more complicated and is generally less attractive than the window solution.

**Interview takeaway:** When the requirement says “previous N records,” think window frames. When it says “previous N calendar days,” think date-based `RANGE` or a calendar table.

## **2. Find users who performed event A before event B within 30 minutes**

**Problem:** Find users who performed `login` and then `purchase` within 30 minutes of that login.

Assume:

```
events (
    event_id,
    user_id,
    event_name,
    event_time
);
```

### **Approach 1:&#x20;**`EXISTS`

```
SELECT DISTINCT
    a.user_id
FROM events a
WHERE a.event_name = 'login'
  AND EXISTS (
      SELECT 1
      FROM events b
      WHERE b.user_id = a.user_id
        AND b.event_name = 'purchase'
        AND b.event_time >= a.event_time
        AND b.event_time <= a.event_time + INTERVAL '30 minutes'
        AND (b.event_time, b.event_id)
              > (a.event_time, a.event_id)
  );
```

The event ID is used to break timestamp ties and establish a deterministic ordering.

Why `EXISTS`?

We only need to establish that at least one qualifying purchase exists. We do not need to retrieve every matching purchase.

### **Approach 2:&#x20;**`JOIN`**&#x20;+ time condition**

```
SELECT DISTINCT
    a.user_id
FROM events a
JOIN events b
    ON b.user_id = a.user_id
   AND b.event_name = 'purchase'
   AND b.event_time >= a.event_time
   AND b.event_time <= a.event_time + INTERVAL '30 minutes'
   AND (b.event_time, b.event_id)
         > (a.event_time, a.event_id)
WHERE a.event_name = 'login';
```

Both solutions return users who satisfy the condition.

### **A harder variation**

What if the interviewer asks:

> Return each login and its first subsequent purchase within 30 minutes.

Use PostgreSQL's `LEFT JOIN LATERAL`:

```
SELECT
    a.user_id,
    a.event_id AS login_id,
    a.event_time AS login_time,
    b.event_id AS purchase_id,
    b.event_time AS purchase_time
FROM events a
LEFT JOIN LATERAL (
    SELECT
        event_id,
        event_time
    FROM events b
    WHERE b.user_id = a.user_id
      AND b.event_name = 'purchase'
      AND (b.event_time, b.event_id)
            > (a.event_time, a.event_id)
      AND b.event_time <= a.event_time + INTERVAL '30 minutes'
    ORDER BY b.event_time, b.event_id
    LIMIT 1
) b ON TRUE
WHERE a.event_name = 'login';
```

`LATERAL` lets the subquery refer to values from the preceding row in the `FROM` clause. Here, it finds the first eligible purchase for each login.

**Interview takeaway:** Know when to use `EXISTS` (existence), `JOIN` (matching rows), and `LATERAL` (a correlated subquery returning rows).

## **3. Identify user sessions from event logs**

**Problem:** A user can perform many events. If the gap between consecutive events is more than 30 minutes, consider the next event to be the beginning of a new session.

Find each user's session start, session end and event count.

Assume the same `events` table as above.

Example:

| user_id | event_time |
| ------- | ---------- |
| 101     | 10:00      |
| 101     | 10:10      |
| 101     | 10:25      |
| 101     | 11:10      |
| 101     | 11:20      |
| 102     | 10:00      |
| 102     | 11:00      |

For user 101, the first three events belong to one session. The 11:10 event begins another session because the gap is 45 minutes.

### **Step 1: Find the previous event time**

```
WITH previous_events AS (
    SELECT
        user_id,
        event_id,
        event_time,
        LAG(event_time) OVER (
            PARTITION BY user_id
            ORDER BY event_time, event_id
        ) AS previous_event_time
    FROM events
)
SELECT *
FROM previous_events;
```

`LAG()` looks at the previous row within the same user's ordered events.

### **Step 2: Mark the beginning of each session**

```
WITH previous_events AS (
    SELECT
        user_id,
        event_id,
        event_time,
        LAG(event_time) OVER (
            PARTITION BY user_id
            ORDER BY event_time, event_id
        ) AS previous_event_time
    FROM events
),
marked AS (
    SELECT *,
        CASE
            WHEN previous_event_time IS NULL THEN 1
            WHEN event_time - previous_event_time
                    > INTERVAL '30 minutes' THEN 1
            ELSE 0
        END AS new_session
    FROM previous_events
)
SELECT *
FROM marked;
```

A `1` means a new session begins.

### **Step 3: Assign session numbers**

```
WITH previous_events AS (
    SELECT
        user_id,
        event_id,
        event_time,
        LAG(event_time) OVER (
            PARTITION BY user_id
            ORDER BY event_time, event_id
        ) AS previous_event_time
    FROM events
),
marked AS (
    SELECT *,
        CASE
            WHEN previous_event_time IS NULL THEN 1
            WHEN event_time - previous_event_time
                    > INTERVAL '30 minutes' THEN 1
            ELSE 0
        END AS new_session
    FROM previous_events
),
sessionized AS (
    SELECT *,
        SUM(new_session) OVER (
            PARTITION BY user_id
            ORDER BY event_time, event_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS session_number
    FROM marked
)
SELECT
    user_id,
    session_number,
    MIN(event_time) AS session_start,
    MAX(event_time) AS session_end,
    COUNT(*) AS total_events,
    MAX(event_time) - MIN(event_time) AS session_duration
FROM sessionized
GROUP BY user_id, session_number
ORDER BY user_id, session_number;
```

### **Understand the pattern**

**Sessionisation logic**

**10:00**

Event 1

**10:10**

Event 2

**10:25**

Event 3

Session 1 — gaps are no more than 30 minutes

**11:10**

Event 4

**11:20**

Event 5

Session 2 — the 45-minute gap started a new session

**Important edge case:** We used `> INTERVAL '30 minutes'`. If the rule says a gap of 30 minutes or more starts a new session, use `>=`.

**Interview takeaway:** This is one of the most reusable applications of `LAG()` + cumulative `SUM()`. It appears in website analytics, app usage, clickstream analysis and customer behaviour tracking.

## **4. Find the first and last transaction for every customer in each month**

**Problem:** For every customer and calendar month, return their first and last transaction, including the transaction IDs and amounts.

Assume:

```
transactions (
    txn_id,
    user_id,
    txn_time,
    amount
);
```

### **Approach 1: Two&#x20;**`ROW_NUMBER()`**&#x20;rankings**

```
WITH ranked AS (
    SELECT
        txn_id,
        user_id,
        txn_time,
        amount,
        DATE_TRUNC('month', txn_time) AS txn_month,

        ROW_NUMBER() OVER (
            PARTITION BY
                user_id,
                DATE_TRUNC('month', txn_time)
            ORDER BY txn_time, txn_id
        ) AS first_rank,

        ROW_NUMBER() OVER (
            PARTITION BY
                user_id,
                DATE_TRUNC('month', txn_time)
            ORDER BY txn_time DESC, txn_id DESC
        ) AS last_rank
    FROM transactions
)
SELECT
    user_id,
    txn_month,

    MAX(txn_id) FILTER (
        WHERE first_rank = 1
    ) AS first_txn_id,

    MAX(amount) FILTER (
        WHERE first_rank = 1
    ) AS first_txn_amount,

    MAX(txn_id) FILTER (
        WHERE last_rank = 1
    ) AS last_txn_id,

    MAX(amount) FILTER (
        WHERE last_rank = 1
    ) AS last_txn_amount

FROM ranked
GROUP BY user_id, txn_month
ORDER BY user_id, txn_month;
```

Why use `FILTER`?

It allows an aggregate to consider only rows matching a condition.

For example:

```
MAX(amount) FILTER (WHERE first_rank = 1)
```

means calculate the maximum amount only among rows marked as the first transaction.

Since there is exactly one such row per customer-month, this retrieves its amount.

### **Approach 2: PostgreSQL&#x20;**`DISTINCT ON`

PostgreSQL has a convenient feature called `DISTINCT ON`.

```
WITH first_txns AS (
    SELECT DISTINCT ON (
        user_id,
        DATE_TRUNC('month', txn_time)
    )
        user_id,
        DATE_TRUNC('month', txn_time) AS txn_month,
        txn_id AS first_txn_id,
        amount AS first_amount
    FROM transactions
    ORDER BY
        user_id,
        DATE_TRUNC('month', txn_time),
        txn_time,
        txn_id
),
last_txns AS (
    SELECT DISTINCT ON (
        user_id,
        DATE_TRUNC('month', txn_time)
    )
        user_id,
        DATE_TRUNC('month', txn_time) AS txn_month,
        txn_id AS last_txn_id,
        amount AS last_amount
    FROM transactions
    ORDER BY
        user_id,
        DATE_TRUNC('month', txn_time),
        txn_time DESC,
        txn_id DESC
)
SELECT
    f.user_id,
    f.txn_month,
    f.first_txn_id,
    f.first_amount,
    l.last_txn_id,
    l.last_amount
FROM first_txns f
JOIN last_txns l
    USING (user_id, txn_month)
ORDER BY f.user_id, f.txn_month;
```

**Which is better?**

| Method            | Advantage                                                                        |
| ----------------- | -------------------------------------------------------------------------------- |
| `ROW_NUMBER()`    | Portable concept, flexible, one scan in this formulation                         |
| `DISTINCT ON`     | Concise and useful in PostgreSQL                                                 |
| `MIN()` / `MAX()` | Simple for dates, but can return multiple tied transaction rows when joined back |

**Interview takeaway:** Whenever the question says first, last, earliest, latest, top or bottom record, think ranking functions.

## **5. Calculate monthly retention and next-month churn**

**Problem:** Find how many customers active in each month returned in the immediately following month.

A customer who returns after two months does **not** count as next-month retained.

Assume:

```
orders (
    order_id,
    customer_id,
    order_date
);
```

### **Step 1: Create unique customer-month activity**

```
WITH monthly_activity AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS activity_month
    FROM orders
)
SELECT *
FROM monthly_activity;
```

This creates one row per customer per month, regardless of the number of orders.

### **Step 2: Find each customer's next active month**

```
WITH monthly_activity AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS activity_month
    FROM orders
),
activity_with_next AS (
    SELECT
        customer_id,
        activity_month,
        LEAD(activity_month) OVER (
            PARTITION BY customer_id
            ORDER BY activity_month
        ) AS next_active_month
    FROM monthly_activity
)
SELECT *
FROM activity_with_next;
```

### **Step 3: Calculate retention and churn**

```
WITH monthly_activity AS (
    SELECT DISTINCT
        customer_id,
        DATE_TRUNC('month', order_date) AS activity_month
    FROM orders
),
activity_with_next AS (
    SELECT
        customer_id,
        activity_month,
        LEAD(activity_month) OVER (
            PARTITION BY customer_id
            ORDER BY activity_month
        ) AS next_active_month
    FROM monthly_activity
)
SELECT
    activity_month,
    COUNT(*) AS active_customers,

    COUNT(*) FILTER (
        WHERE next_active_month =
              activity_month + INTERVAL '1 month'
    ) AS retained_next_month,

    COUNT(*) FILTER (
        WHERE next_active_month IS DISTINCT FROM
              activity_month + INTERVAL '1 month'
    ) AS not_retained_next_month,

    ROUND(
        100.0 * COUNT(*) FILTER (
            WHERE next_active_month =
                  activity_month + INTERVAL '1 month'
        ) / COUNT(*),
        2
    ) AS next_month_retention_pct

FROM activity_with_next
GROUP BY activity_month
ORDER BY activity_month;
```

Notice this expression:

```
next_active_month IS DISTINCT FROM
    activity_month + INTERVAL '1 month'
```

Unlike ordinary `<>`, `IS DISTINCT FROM` handles `NULL` explicitly. A customer with no later activity is counted as not retained.

### **Critical business distinction**

Not retained next month does not necessarily mean permanently churned.

A customer could be inactive in February but return in March. They failed next-month retention in February, but might not be considered permanently churned under a business's definition.

**Interview takeaway:** Always define churn before writing SQL. Some companies define it as one inactive month; others require two or more months of inactivity.

## **6. Debug SQL — inflated revenue caused by joins**

**Problem:** A company has orders, items and payments.

```
orders (
    order_id,
    customer_id
);

order_items (
    item_id,
    order_id,
    quantity,
    price
);

payments (
    payment_id,
    order_id,
    amount
);
```

A developer writes:

```
SELECT
    o.customer_id,
    SUM(oi.quantity * oi.price) AS item_revenue,
    SUM(p.amount) AS payment_total
FROM orders o
JOIN order_items oi
    ON oi.order_id = o.order_id
JOIN payments p
    ON p.order_id = o.order_id
GROUP BY o.customer_id;
```

It looks correct. But it can produce **inflated totals**.

### **Why?**

Suppose order 1 has:

`order_items`

| item_id | order_id | item revenue |
| ------- | -------- | ------------ |
| 1       | 1        | 100          |
| 2       | 1        | 200          |

`payments`

| payment_id | order_id | amount |
| ---------- | -------- | ------ |
| 1          | 1        | 150    |
| 2          | 1        | 150    |

Joining both tables produces:

| item revenue | payment amount |
| ------------ | -------------- |
| 100          | 150            |
| 100          | 150            |
| 200          | 150            |
| 200          | 150            |

Actual item revenue = ₹300.

Actual payments = ₹300.

But the query calculates:

- Item revenue = ₹600.
- Payment total = ₹600.

This is the classic **join multiplication** problem.

### **Correct solution: Aggregate before joining**

```
WITH item_totals AS (
    SELECT
        order_id,
        SUM(quantity * price) AS item_revenue
    FROM order_items
    GROUP BY order_id
),
payment_totals AS (
    SELECT
        order_id,
        SUM(amount) AS payment_total
    FROM payments
    GROUP BY order_id
)
SELECT
    o.customer_id,
    SUM(COALESCE(i.item_revenue, 0)) AS total_item_revenue,
    SUM(COALESCE(p.payment_total, 0)) AS total_payments
FROM orders o
LEFT JOIN item_totals i
    ON i.order_id = o.order_id
LEFT JOIN payment_totals p
    ON p.order_id = o.order_id
GROUP BY o.customer_id;
```

Now each order has at most one row in each preaggregated table.

This prevents the many-to-many multiplication.

**Interview takeaway:** When aggregating across multiple one-to-many relationships, aggregate each child table to the required join grain first.

## **7. Seven SQL debugging traps you must recognise**

These are common reasons for queries passing basic tests but failing hidden OA test cases.

| Trap                     | Incorrect approach                                   | Correct thinking                                      |
| ------------------------ | ---------------------------------------------------- | ----------------------------------------------------- |
| NULL in `NOT IN`         | `WHERE id NOT IN (subquery)`                         | Prefer `NOT EXISTS` when subquery can contain NULL    |
| LEFT JOIN filtering      | Put right-table condition in `WHERE` unintentionally | Use `ON` when unmatched left rows must remain         |
| Duplicate multiplication | Join multiple one-to-many tables before summing      | Preaggregate child tables                             |
| Wrong count              | `COUNT(*)` after a LEFT JOIN                         | Use `COUNT(child.id)` to exclude unmatched rows       |
| Ranking ties             | `ROW_NUMBER()` when all tied winners are required    | Use `RANK()` or `DENSE_RANK()`                        |
| Nondeterministic results | `ORDER BY date` when dates can tie                   | Add a unique tie-breaker                              |
| Wrong window frame       | Assume seven rows means seven days                   | Choose `ROWS` or `RANGE` according to the requirement |

### **Three particularly important examples**

**Trap A —&#x20;**`NOT IN`**&#x20;and NULL**

```
-- Potentially incorrect if the subquery returns NULL
SELECT *
FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id
    FROM orders
);
```

Safer:

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

**Trap B — LEFT JOIN condition placement**

```
-- Removes customers without completed orders
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
WHERE o.status = 'Completed';
```

To retain every customer, put the condition in `ON`:

```
SELECT c.customer_id, o.order_id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
   AND o.status = 'Completed';
```

**Trap C — COUNT after LEFT JOIN**

```
-- Incorrect when customers have no orders
SELECT
    c.customer_id,
    COUNT(*) AS order_count
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
GROUP BY c.customer_id;
```

A customer with no orders still produces one NULL-extended joined row.

Correct:

```
COUNT(o.order_id)
```

# **Set 6 — Practice assessment**

Try these without referring to the solutions above.

**Five-question checkpoint**

Self-test

**Q1. A user generates 10 events. A gap greater than 45 minutes starts a new session. Which function helps identify the previous event?**

LEAD()

LAG()

NTILE()

FIRST_VALUE()

**Q2. You need the first purchase after each login, within 20 minutes. Which PostgreSQL feature is especially useful?**

CROSS JOIN

DISTINCT ON only

LEFT JOIN LATERAL

UNION

**Q3. Two order items and three payment rows belong to the same order. How many rows can their direct join produce?**

2

3

5

6

**Q4. You need the previous seven calendar days, including the current date. What matters most?**

Only ROWS

A date-based RANGE or calendar table

Only DISTINCT

GROUP BY alone

**Q5. A customer is active in January and March but not February. Are they retained in January's next-month retention?**

Yes

No

Check answers

## **Where we are now**

You have completed the advanced pattern-building stage up to Set 6.

**Next: Set 7 — Timed SQL OA Simulation.**

It will contain three difficulty levels:

- **Easy:** 10 questions covering filtering, aggregation, joins and basic subqueries.
- **Medium:** 10 questions covering ranking, CTEs, correlated subqueries and analytical functions.
- **Hard:** 5 questions involving sessionisation, retention, gaps and islands, multi-table aggregation and query constraints.

For every question, we'll evaluate correctness, edge cases, efficiency and whether you can produce an alternative solution. The aim now is to turn the material you've studied into independent interview-level problem-solving ability.


