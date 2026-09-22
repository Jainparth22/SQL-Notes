# **Advanced SQL Interview Drill — Set 3 Solutions + Set 4**

Before introducing another set of questions, let's complete the five challenges from Set 3. These are particularly useful because they combine multiple SQL concepts.

# **1. Review of Set 3 — Reference solutions**

## **Problem 1: Second-highest distinct salary in each department**

**Approach A — Correlated subquery without window functions**

```
SELECT
    e1.emp_id,
    e1.name,
    e1.department_id,
    e1.salary
FROM employees e1
WHERE e1.salary IS NOT NULL
  AND (
      SELECT COUNT(DISTINCT e2.salary)
      FROM employees e2
      WHERE e2.department_id
            IS NOT DISTINCT FROM e1.department_id
        AND e2.salary > e1.salary
  ) = 1;
```

**Approach B — GROUP BY + JOIN**

```
WITH department_max AS (
    SELECT
        department_id,
        MAX(salary) AS highest_salary
    FROM employees
    GROUP BY department_id
),

second_salary AS (
    SELECT
        e.department_id,
        MAX(e.salary) AS second_highest
    FROM employees e
    JOIN department_max d
      ON e.department_id
         IS NOT DISTINCT FROM d.department_id
    WHERE e.salary < d.highest_salary
    GROUP BY e.department_id
)

SELECT
    e.name,
    e.department_id,
    e.salary
FROM employees e
JOIN second_salary s
  ON e.department_id
     IS NOT DISTINCT FROM s.department_id
 AND e.salary = s.second_highest;
```

**Why two approaches?**

The first counts higher distinct salaries for each candidate employee. The second calculates the highest and then the highest salary below it for every department.

Both preserve ties.

## **Problem 2: Customers with at least three completed orders in the same month**

```
SELECT
    customer_id,
    DATE_TRUNC('month', order_date) AS order_month,
    COUNT(DISTINCT order_id) AS completed_orders
FROM orders
WHERE status = 'Completed'
GROUP BY
    customer_id,
    DATE_TRUNC('month', order_date)
HAVING COUNT(DISTINCT order_id) >= 3
ORDER BY customer_id, order_month;
```

**Important:** Three orders across three different months do not qualify. We group by both customer and month.

## **Problem 3: Longest-tenured employee in each department, including ties**

Assume `employees` has a `hire_date` column.

**Approach A — RANK**

```
WITH ranked AS (
    SELECT
        emp_id,
        name,
        department_id,
        hire_date,

        RANK() OVER (
            PARTITION BY department_id
            ORDER BY hire_date ASC
        ) AS rnk

    FROM employees
    WHERE hire_date IS NOT NULL
)

SELECT *
FROM ranked
WHERE rnk = 1;
```

The earliest hiring date represents the longest tenure, assuming all employees are evaluated at the same reference date.

**Approach B — MIN + JOIN**

```
SELECT
    e.emp_id,
    e.name,
    e.department_id,
    e.hire_date
FROM employees e
JOIN (
    SELECT
        department_id,
        MIN(hire_date) AS earliest_hire
    FROM employees
    WHERE hire_date IS NOT NULL
    GROUP BY department_id
) d
  ON e.department_id = d.department_id
 AND e.hire_date = d.earliest_hire;
```

Both return all tied employees.

## **Problem 4: Three consecutive login days, but not the fourth**

There are two possible interpretations.

**Interpretation A:** The user logged in on three consecutive days and did not log in on the following day.

**Interpretation B:** The user had an exact three-day streak, with neither a preceding nor following login day.

The distinction matters.

Assume the requirement is an exact three-day streak.

```
WITH dates AS (
    SELECT DISTINCT
        user_id,
        login_date::DATE AS login_date
    FROM logins
),

triples AS (
    SELECT
        d.user_id,
        d.login_date AS start_date,
        d.login_date + 2 AS end_date

    FROM dates d

    WHERE EXISTS (
        SELECT 1
        FROM dates d2
        WHERE d2.user_id = d.user_id
          AND d2.login_date = d.login_date + 1
    )

    AND EXISTS (
        SELECT 1
        FROM dates d3
        WHERE d3.user_id = d.user_id
          AND d3.login_date = d.login_date + 2
    )

    AND NOT EXISTS (
        SELECT 1
        FROM dates d4
        WHERE d4.user_id = d.user_id
          AND d4.login_date = d.login_date - 1
    )

    AND NOT EXISTS (
        SELECT 1
        FROM dates d5
        WHERE d5.user_id = d.user_id
          AND d5.login_date = d.login_date + 3
    )
)

SELECT *
FROM triples;
```

This finds maximal streaks of exactly three days.

## **Problem 5: Percentage of customers making a second purchase within 30 days**

Assumption: The denominator is customers who made at least one purchase, not all registered customers.

```
WITH ranked_orders AS (
    SELECT
        customer_id,
        order_id,
        order_date,

        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date, order_id
        ) AS rn

    FROM orders
),

customer_purchases AS (
    SELECT
        customer_id,

        MAX(order_date) FILTER (
            WHERE rn = 1
        ) AS first_purchase,

        MAX(order_date) FILTER (
            WHERE rn = 2
        ) AS second_purchase

    FROM ranked_orders
    GROUP BY customer_id
)

SELECT
    ROUND(
        100.0 *
        COUNT(*) FILTER (
            WHERE second_purchase <=
                  first_purchase + INTERVAL '30 days'
        )
        / NULLIF(COUNT(*), 0),
        2
    ) AS second_purchase_rate
FROM customer_purchases;
```

The `ROW_NUMBER()` selects the first and second purchases. The outer query measures how many customers made their second purchase within 30 days.

If the interviewer wants the percentage of all registered customers, change the denominator to the total number of rows in `customers` and start from that table using a LEFT JOIN.

# **Advanced SQL Interview Drill — Set 4: Query Rewriting and Alternative Solutions**



Medium → Hard

Interviewer constraint-based round



This set focuses on a common interview situation: the first query works, but the interviewer changes the requirement or imposes a restriction.

We'll cover six problems.

## **Problem 1 — Find the latest order of every customer**

**Schema:** `orders(order_id, customer_id, order_date, amount)`

The interviewer asks:

"Find each customer's most recent order."

### **Method 1 — ROW_NUMBER()**

```
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY customer_id
            ORDER BY order_date DESC, order_id DESC
        ) AS rn
    FROM orders
)
SELECT *
FROM ranked
WHERE rn = 1;
```

### **Method 2 — MAX() + JOIN**

```
SELECT o.*
FROM orders o
JOIN (
    SELECT
        customer_id,
        MAX(order_date) AS latest_date
    FROM orders
    GROUP BY customer_id
) latest
  ON o.customer_id = latest.customer_id
 AND o.order_date = latest.latest_date;
```

This returns all orders tied at the latest timestamp.

### **Method 3 — PostgreSQL DISTINCT ON**

```
SELECT DISTINCT ON (customer_id)
    *
FROM orders
ORDER BY customer_id, order_date DESC, order_id DESC;
```

PostgreSQL-specific

`DISTINCT ON` selects the first row in each customer group according to the specified ordering.

**Interviewer follow-up:** What if two orders have the same timestamp?

- ROW_NUMBER with order_id chooses one deterministically.
- MAX + JOIN returns both.
- DISTINCT ON with order_id chooses one deterministically.

Always clarify whether the interviewer wants one order or every tied latest order.

## **Problem 2 — Find customers whose completed spending exceeds ₹1,000**

### **Method 1 — GROUP BY + HAVING**

```
SELECT
    customer_id,
    SUM(amount) AS total_spent
FROM orders
WHERE status = 'Completed'
GROUP BY customer_id
HAVING SUM(amount) > 1000;
```

### **Method 2 — Correlated subquery**

```
SELECT *
FROM customers c
WHERE (
    SELECT SUM(o.amount)
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'Completed'
) > 1000;
```

### **Method 3 — CTE + JOIN**

```
WITH customer_spending AS (
    SELECT
        customer_id,
        SUM(amount) AS total_spent
    FROM orders
    WHERE status = 'Completed'
    GROUP BY customer_id
)
SELECT
    c.customer_id,
    c.customer_name,
    cs.total_spent
FROM customers c
JOIN customer_spending cs
    ON c.customer_id = cs.customer_id
WHERE cs.total_spent > 1000;
```

**What to explain:** The first approach is shortest. The second expresses the condition independently for each customer. The third makes the intermediate aggregation reusable and provides customer names.

## **Problem 3 — Find employees earning more than every employee in department 102**

This question tests ANY, ALL and aggregate subqueries.

### **Method 1 — ALL**

```
SELECT *
FROM employees
WHERE salary > ALL (
    SELECT salary
    FROM employees
    WHERE department_id = 102
      AND salary IS NOT NULL
);
```

### **Method 2 — MAX()**

```
SELECT *
FROM employees
WHERE salary > (
    SELECT MAX(salary)
    FROM employees
    WHERE department_id = 102
);
```

Both represent the requirement when department 102 has at least one non-NULL salary.

**Interview trap:** NULL values and empty subqueries affect ALL semantics. Adding `salary IS NOT NULL` makes the intended comparison explicit.

### **Variation**

Find employees earning more than at least one employee in department 102.

```
SELECT *
FROM employees
WHERE salary > ANY (
    SELECT salary
    FROM employees
    WHERE department_id = 102
      AND salary IS NOT NULL
);
```

| Requirement                      | Logic |
| -------------------------------- | ----- |
| Greater than every salary        | > ALL |
| Greater than at least one salary | > ANY |
| Greater than highest salary      | > MAX |
| Greater than lowest salary       | > MIN |

## **Problem 4 — Find customers who have exactly two completed orders**

### **Method 1 — GROUP BY + HAVING**

```
SELECT
    customer_id,
    COUNT(*) AS completed_orders
FROM orders
WHERE status = 'Completed'
GROUP BY customer_id
HAVING COUNT(*) = 2;
```

### **Method 2 — Correlated subquery**

```
SELECT *
FROM customers c
WHERE (
    SELECT COUNT(*)
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'Completed'
) = 2;
```

### **Method 3 — Window function**

```
WITH customer_counts AS (
    SELECT
        customer_id,
        order_id,

        COUNT(*) OVER (
            PARTITION BY customer_id
        ) AS order_count

    FROM orders
    WHERE status = 'Completed'
)
SELECT DISTINCT customer_id
FROM customer_counts
WHERE order_count = 2;
```

**Follow-up:** Find customers with at least two orders.

Change:

```
HAVING COUNT(*) >= 2;
```

**Follow-up:** Customers with exactly two orders, including customers with zero orders?

Start from `customers`, LEFT JOIN orders, and use `COUNT(o.order_id) = 2`.

## **Problem 5 — Remove duplicate emails while retaining the most recent record**

Advanced

Schema:

`users(user_id, email, created_at)`

Assume email duplicates should be identified without considering case or surrounding spaces.

### **Step 1 — Identify duplicates**

```
WITH ranked AS (
    SELECT
        user_id,
        email,
        created_at,

        ROW_NUMBER() OVER (
            PARTITION BY LOWER(TRIM(email))
            ORDER BY created_at DESC, user_id DESC
        ) AS rn

    FROM users
)
SELECT *
FROM ranked
WHERE rn > 1;
```

This identifies all records except the most recent one for each normalised email.

### **Step 2 — Delete duplicates safely**

PostgreSQL:

```
BEGIN;

WITH ranked AS (
    SELECT
        user_id,

        ROW_NUMBER() OVER (
            PARTITION BY LOWER(TRIM(email))
            ORDER BY created_at DESC, user_id DESC
        ) AS rn

    FROM users
)

DELETE FROM users u
USING ranked r
WHERE u.user_id = r.user_id
  AND r.rn > 1;

-- Inspect results before committing.

COMMIT;
```

During practice, replace COMMIT with ROLLBACK until you have verified the result.

**Interview concepts:**

- ROW_NUMBER.
- PARTITION BY.
- Data cleaning.
- Deterministic tie-breaking.
- DELETE USING.
- Transactions.

## **Problem 6 — Avoid incorrect totals after joining multiple tables**

Important real-world debugging problem

Suppose:

**orders**

| order_id | customer_id | order_total |
| -------- | ----------- | ----------- |
| 101      | 1           | 1500        |
| 102      | 1           | 800         |

**order_items**

| order_id | product_id | quantity |
| -------- | ---------- | -------- |
| 101      | 1          | 2        |
| 101      | 2          | 3        |
| 101      | 3          | 1        |
| 102      | 4          | 2        |

The interviewer asks:

"Calculate total order value per customer."

### **Incorrect solution**

```
SELECT
    o.customer_id,
    SUM(o.order_total) AS total
FROM orders o
JOIN order_items oi
    ON o.order_id = oi.order_id
GROUP BY o.customer_id;
```

Output:

```
Customer 1 = 5300
```

But the correct answer is:

```
1500 + 800 = 2300
```

Why?

Order 101 is repeated three times after joining with its three order-item records.

### **Solution 1 — Aggregate the orders independently**

```
SELECT
    customer_id,
    SUM(order_total) AS total_order_value
FROM orders
GROUP BY customer_id;
```

No JOIN is required.

### **Solution 2 — Pre-aggregate order items**

Suppose the requirement also asks for item quantity.

```
WITH item_summary AS (
    SELECT
        order_id,
        SUM(quantity) AS total_items
    FROM order_items
    GROUP BY order_id
)
SELECT
    o.customer_id,
    SUM(o.order_total) AS total_order_value,
    SUM(i.total_items) AS total_items
FROM orders o
JOIN item_summary i
    ON o.order_id = i.order_id
GROUP BY o.customer_id;
```

This prevents the order-level amount from being multiplied by the number of item rows.

**Core lesson:** Always understand the grain of each table before joining and aggregating.

Grain means what one row represents.

# **7. Interviewer Constraint Challenge**

Try solving these without looking at the hints.

| # | Requirement                                | Constraint             |
| - | ------------------------------------------ | ---------------------- |
| 1 | Latest order per customer                  | No window functions    |
| 2 | Exactly two completed orders               | No GROUP BY            |
| 3 | Highest salary per department              | No MAX                 |
| 4 | Second-highest distinct salary             | No LIMIT or DENSE_RANK |
| 5 | Customers with no orders                   | No NOT EXISTS          |
| 6 | Employees earning above department average | No correlated subquery |
| 7 | Find duplicates                            | No GROUP BY            |
| 8 | Running total                              | No SUM window function |

Hints:

| # | Approach                            |
| - | ----------------------------------- |
| 1 | MAX + JOIN                          |
| 2 | Correlated COUNT                    |
| 3 | NOT EXISTS                          |
| 4 | Correlated COUNT(DISTINCT)          |
| 5 | LEFT JOIN + IS NULL                 |
| 6 | Window AVG or CTE + JOIN            |
| 7 | ROW_NUMBER                          |
| 8 | Correlated aggregation or self-join |

# **8. Your SQL Interview Mastery Scorecard**

The following tracker measures your actual ability, not just lesson completion.

**Interview readiness**

# **0%**

0 of 12 skills independently demonstrated

Solve second and Nth-highest salary using three methods

Solve highest-per-group problems with ties

Use anti-joins and EXISTS correctly

Solve aggregate questions using correlated subqueries

Use JOINs without multiplying aggregates accidentally

Solve top N per group using ranking

Solve duplicate detection and deduplication

Solve running totals and LAG/LEAD problems

Solve gaps-and-islands questions

Handle NULL, empty result and tie cases

Explain execution plans and index choices

Explain ACID and isolation levels

## **Next: Advanced SQL Interview Drill — Set 5**

We'll move into harder problems involving:

1. Relational division and universal conditions.
2. Gaps and islands.
3. Running totals with complex partitions.
4. Median and percentile calculations.
5. Customer retention cohorts.
6. Consecutive transaction sequences.
7. Recursive CTE interview problems.
8. Query optimisation under memory and indexing constraints.

**Your preparation rule:** For every problem, write one correct solution, one alternative solution, and identify the edge case most likely to break a memorised query.

