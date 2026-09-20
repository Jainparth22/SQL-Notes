# **Lesson 10 — SQL Company OAs, LeetCode Practice and Mock Interviews**



Final course module

SDE / Data Engineer / Backend interviews



This is the practical culmination of Lessons 1–9.

You have studied SQL syntax, JOINs, aggregation, subqueries, CTEs, window functions, database normalisation, indexes, transactions and query optimisation.

Now we'll combine them to solve unfamiliar problems under interview and OA conditions.

**Our objective:** Given a problem statement and a database schema, you should be able to identify the required SQL pattern, write the query, explain its correctness, and understand its performance implications.

## **1. Complete structure of Lesson 10**

A

**Company OA problem patterns**

Top N, ranking, duplicate detection, anti-joins, consecutive dates, retention, aggregation and analytical queries.

B

**LeetCode SQL 50 preparation**

Problem categories, difficulty progression, reusable patterns and a structured practice plan.

C

**Technical interview preparation**

Query-writing rounds, DBMS concepts, optimisation, debugging and schema-design questions.

D

**Mock assessments**

Easy, medium and hard timed assessments, with answer keys and explanations.

# **SECTION A — COMPANY OA PROBLEM PATTERNS**

The biggest mistake in SQL preparation is memorising individual queries instead of recognising patterns.

For example, these questions look different:

- Find the second-highest salary.
- Find the third-highest product price.
- Find the second-largest transaction amount.

But they may use the same underlying pattern: `DENSE_RANK()` or a maximum-value subquery.

We'll master the reusable patterns.

## **A1. First understand the common database schema**

We'll use these tables throughout the lesson.

**employees**

| emp_id | name   | department_id | salary | manager_id |
| ------ | ------ | ------------- | ------ | ---------- |
| 1      | Aman   | 101           | 60000  | NULL       |
| 2      | Rahul  | 102           | 80000  | 1          |
| 3      | Priya  | 101           | 75000  | 1          |
| 4      | Neha   | 103           | 90000  | 2          |
| 5      | Karan  | 102           | 50000  | 2          |
| 6      | Simran | 101           | 85000  | 3          |
| 7      | Riya   | 104           | 70000  | 4          |

**departments**

| department_id | department_name |
| ------------- | --------------- |
| 101           | Engineering     |
| 102           | HR              |
| 103           | Finance         |
| 104           | Marketing       |
| 105           | Operations      |

**customers**

| customer_id | customer_name | city   |
| ----------- | ------------- | ------ |
| 1           | Aman          | Delhi  |
| 2           | Rahul         | Mumbai |
| 3           | Priya         | Delhi  |
| 4           | Neha          | Pune   |

**orders**

| order_id | customer_id | order_date | amount | status    |
| -------- | ----------- | ---------- | ------ | --------- |
| 101      | 1           | 2026-01-05 | 500    | Completed |
| 102      | 2           | 2026-01-10 | 800    | Completed |
| 103      | 1           | 2026-02-05 | 300    | Pending   |
| 104      | 3           | 2026-02-10 | 1200   | Completed |
| 105      | 2           | 2026-03-05 | 700    | Completed |

All examples below use PostgreSQL.

## **A2. Pattern 1 — Second-highest salary**

Very common OA question

**Problem:** Find the second-highest distinct salary.

Think:

1. Highest salary = maximum salary.
2. Remove the highest salary.
3. Find the maximum among the remaining salaries.

### **Approach 1: MAX + subquery**

```
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```

### **Approach 2: DENSE_RANK**

```
WITH ranked_salaries AS (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
)
SELECT DISTINCT salary AS second_highest_salary
FROM ranked_salaries
WHERE salary_rank = 2;
```

Both return NULL or no row depending on the query shape when fewer than two distinct non-NULL salaries exist. Here, the aggregate MAX approach returns one row with NULL; the ranking approach returns no row.

**Expected result:**

# **85,000**

### **Pattern variations**

| Question                                 | Approach                  |
| ---------------------------------------- | ------------------------- |
| Highest salary                           | MAX()                     |
| Second-highest distinct salary           | MAX + subquery            |
| Nth-highest distinct salary              | DENSE_RANK                |
| Second employee after sorting            | ROW_NUMBER                |
| All employees tied at highest salary     | MAX + JOIN or subquery    |
| Second-highest salary in each department | PARTITION BY + DENSE_RANK |

**Interview trap:** Always determine whether the question asks for a distinct salary or the second employee after sorting.

## **A3. Pattern 2 — Nth-highest salary per department**

**Problem:** Find the second-highest distinct salary in every department.

```
WITH ranked AS (
    SELECT
        name,
        department_id,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees
)
SELECT name, department_id, salary, salary_rank
FROM ranked
WHERE salary_rank = 2;
```

A global ranking is incorrect because each department needs its own ranking.

Expected result:

| name  | department_id | salary | salary_rank |
| ----- | ------------- | ------ | ----------- |
| Priya | 101           | 75000  | 2           |
| Karan | 102           | 50000  | 2           |

Finance and Marketing have only one employee each, so they have no second distinct salary.

**Recognition rule:** Whenever the problem says *within each group*, consider `PARTITION BY`.

## **A4. Pattern 3 — Top N employees in each department**

**Problem:** Find the top two employees in each department.

```
WITH ranked AS (
    SELECT
        emp_id,
        name,
        department_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC, emp_id
        ) AS rn
    FROM employees
)
SELECT emp_id, name, department_id, salary
FROM ranked
WHERE rn <= 2;
```

Why include `emp_id` in ORDER BY?

It provides deterministic ordering when two employees have equal salaries.

**Choose the ranking function according to the requirement:**

| Requirement                          | Function   |
| ------------------------------------ | ---------- |
| Exactly two employees per department | ROW_NUMBER |
| Top two distinct salary levels       | DENSE_RANK |
| Competition ranking, including ties  | RANK       |

This pattern is used in employee rankings, product rankings, customer spending rankings and sales leaderboards.

## **A5. Pattern 4 — Employees earning more than department average**

**Problem:** Find employees earning above their department's average salary.

Approach using a window function:

```
-- Does NOT work: window functions are not allowed in WHERE
SELECT name, department_id, salary
FROM employees
WHERE salary > AVG(salary) OVER (PARTITION BY department_id);
```

Dialect warning

Window functions cannot appear directly in WHERE (PostgreSQL rejects the query above, and most databases do the same). Calculate the window value first in a CTE, then filter:

```
WITH with_avg AS (
    SELECT
        name,
        department_id,
        salary,
        AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
    FROM employees
)
SELECT name, department_id, salary, ROUND(dept_avg, 2) AS dept_avg
FROM with_avg
WHERE salary > dept_avg;
```

Alternative using a correlated subquery:

```
SELECT e.name, e.department_id, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

You should understand both approaches.

## **A6. Pattern 5 — Duplicate detection**

Frequently tested data-cleaning question

Consider:

**users**

| user_id | email        |
| ------- | ------------ |
| 1       | a\@gmail.com |
| 2       | b\@gmail.com |
| 3       | a\@gmail.com |
| 4       | c\@gmail.com |
| 5       | b\@gmail.com |
| 6       | a\@gmail.com |

**Problem 1: Find duplicate emails.**

```
SELECT email, COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Problem 2: Find all duplicate rows except the first occurrence.**

```
WITH numbered AS (
    SELECT
        user_id,
        email,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY user_id) AS rn
    FROM users
)
SELECT user_id, email
FROM numbered
WHERE rn > 1;
```

**Problem 3: Keep only one occurrence.**

```
WITH numbered AS (
    SELECT
        user_id,
        email,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY user_id) AS rn
    FROM users
)
SELECT user_id, email
FROM numbered
WHERE rn = 1;
```

The last query identifies which rows to keep; it does not delete duplicates.

**Recognition rule:**

- Duplicate values: GROUP BY + HAVING.
- Individual duplicate records: ROW_NUMBER.
- Keep one occurrence: ROW_NUMBER = 1.

## **A7. Pattern 6 — Find records without matches (Anti-Join)**

**Problem:** Find customers who have never placed an order.

Approach 1: NOT EXISTS

```
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

Approach 2: LEFT JOIN

```
SELECT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

Both work.

**Which is preferable?**

For existence and non-existence questions, EXISTS and NOT EXISTS often express intent particularly clearly.

They also avoid row multiplication when customers have multiple matching orders.

**OA recognition keywords:**

- Never placed an order.
- No matching record.
- No transactions.
- Without a corresponding entry.
- Missing relationship.

Think `NOT EXISTS` or `LEFT JOIN ... IS NULL`.

## **A8. Pattern 7 — Customers with at least one qualifying order**

**Problem:** Find customers who have at least one completed order above ₹700.

```
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'Completed'
      AND o.amount > 700
);
```

Why EXISTS?

We only care whether a qualifying order exists, not how many there are.

Alternative with JOIN:

```
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
JOIN orders o
    ON o.customer_id = c.customer_id
WHERE o.status = 'Completed'
  AND o.amount > 700;
```

The EXISTS approach avoids needing DISTINCT to remove customers repeated by multiple matching orders.

## **A9. Pattern 8 — Running total and cumulative sales**

**Problem:** Calculate cumulative revenue by date.

```
WITH daily_revenue AS (
    SELECT
        order_date,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'Completed'
    GROUP BY order_date
)
SELECT
    order_date,
    revenue,
    SUM(revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_revenue
FROM daily_revenue
ORDER BY order_date;
```

Important: The inner SUM aggregates orders by date. The outer SUM window function calculates the cumulative total across those date groups.

This is a useful pattern when the source table contains multiple transactions per day.

## **A10. Pattern 9 — Month-over-month growth**

**Problem:** Calculate monthly completed revenue and its percentage change.

```
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month_start,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'Completed'
    GROUP BY DATE_TRUNC('month', order_date)
),
compared AS (
    SELECT
        month_start,
        revenue,
        LAG(revenue) OVER (ORDER BY month_start) AS previous_revenue
    FROM monthly
)
SELECT
    month_start,
    revenue,
    previous_revenue,
    ROUND(
        100.0 * (revenue - previous_revenue)
        / NULLIF(previous_revenue, 0),
        2
    ) AS growth_percentage
FROM compared
ORDER BY month_start;
```

**Important subtlety:** This compares with the preceding available month in the data, not necessarily the preceding calendar month. For calendar-complete reporting, use a calendar table or generate a series and fill missing months with zero where appropriate.

## **A11. Pattern 10 — Consecutive login dates**

**Problem:** Find users who logged in on consecutive dates.

**logins**

| user_id | login_date |
| ------- | ---------- |
| 1       | Jan 01     |
| 1       | Jan 02     |
| 1       | Jan 03     |
| 1       | Jan 05     |
| 2       | Jan 01     |
| 2       | Jan 03     |

Solution:

```
WITH login_comparison AS (
    SELECT
        user_id,
        login_date,
        LAG(login_date) OVER (
            PARTITION BY user_id
            ORDER BY login_date
        ) AS previous_login
    FROM logins
)
SELECT user_id, login_date, previous_login
FROM login_comparison
WHERE login_date = previous_login + 1;
```

This identifies consecutive pairs.

### **Advanced variation: Find users with at least three consecutive login days**

First remove duplicate user/date records, if they can exist.

```
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date
    FROM logins
),
numbered AS (
    SELECT
        user_id,
        login_date,
        login_date - (
            ROW_NUMBER() OVER (
                PARTITION BY user_id
                ORDER BY login_date
            )
        )::INT AS grp
    FROM distinct_logins
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS streak_days
FROM numbered
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

This technique is called **Gaps and Islands**.

The central insight is that consecutive dates and their increasing row numbers produce the same date-minus-row-number value within a consecutive sequence.

## **A12. Pattern 11 — Gaps and Islands**

Advanced OA pattern

Gaps and Islands is a general problem-solving technique.

- Islands are consecutive values or dates.
- Gaps are missing values between sequences.

Example:

Using the `logins` table from the previous pattern, remove the `HAVING` clause to list every island:

```
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date FROM logins
),
numbered AS (
    SELECT
        user_id,
        login_date,
        login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::INT AS grp
    FROM distinct_logins
)
SELECT user_id, MIN(login_date) AS island_start, MAX(login_date) AS island_end, COUNT(*) AS days
FROM numbered
GROUP BY user_id, grp
ORDER BY user_id, island_start;
```

The islands are:

| user_id | island_start | island_end | days |
| ------- | ------------ | ---------- | ---- |
| 1       | Jan 01       | Jan 03     | 3    |
| 1       | Jan 05       | Jan 05     | 1    |
| 2       | Jan 01       | Jan 01     | 1    |
| 2       | Jan 03       | Jan 03     | 1    |

The gap between Jan 03 and Jan 05 is what splits user 1 into two islands.

### **Another example: Consecutive order IDs**

```
SELECT
    MIN(order_id) AS range_start,
    MAX(order_id) AS range_end
FROM (
    SELECT
        order_id,
        order_id - ROW_NUMBER() OVER (ORDER BY order_id) AS grp
    FROM orders
) t
GROUP BY grp
ORDER BY range_start;
```

This identifies continuous ranges of consecutive integer IDs.

The same principle works for dates, activity streaks and consecutive transaction sequences after appropriate handling of duplicates and ordering.

## **A13. Pattern 12 — Customer retention and churn**

Important for Data Engineer and Analytics OAs

These questions are common in business analytics and data engineering assessments.

### **Understand retention**

Retention measures whether customers continue using a service after their initial activity.

Consider:

**user_activity**

| user_id | activity_date |
| ------- | ------------- |
| 1       | Jan 01        |
| 1       | Jan 02        |
| 1       | Jan 15        |
| 2       | Jan 01        |
| 2       | Feb 01        |
| 3       | Jan 10        |

**Problem:** Find users who returned on the day immediately after their first activity.

Users 1 and 2? No. User 1 returned on January 2, while user 2 returned on February 1. Only user 1 qualifies.

```
WITH first_activity AS (
    SELECT user_id, MIN(activity_date) AS first_date
    FROM user_activity
    GROUP BY user_id
)
SELECT DISTINCT f.user_id
FROM first_activity f
JOIN user_activity a
    ON a.user_id = f.user_id
   AND a.activity_date = f.first_date + 1;
```

Expected output:

| user_id |
| ------- |
| 1       |

Calculate the Day 1 retention rate:

```
WITH first_activity AS (
    SELECT user_id, MIN(activity_date) AS first_date
    FROM user_activity
    GROUP BY user_id
),
retained AS (
    SELECT DISTINCT f.user_id
    FROM first_activity f
    JOIN user_activity a
        ON a.user_id = f.user_id
       AND a.activity_date = f.first_date + 1
)
SELECT ROUND(
    100.0 * (SELECT COUNT(*) FROM retained)
          / (SELECT COUNT(*) FROM first_activity),
    2
) AS day1_retention_rate;
```

Result: 33.33%.

### **Understand churn**

Churn refers to customers who stop using or paying for a service during a defined period.

Suppose you have:

**subscriptions**

| customer_id | start_date | end_date |
| ----------- | ---------- | -------- |
| 1           | Jan 01     | NULL     |
| 2           | Jan 01     | Feb 01   |
| 3           | Jan 10     | Jan 30   |
| 4           | Jan 15     | NULL     |

Find customers whose subscriptions ended during February 2026:

```
SELECT customer_id, end_date
FROM subscriptions
WHERE end_date >= DATE '2026-02-01'
  AND end_date <  DATE '2026-03-01';
```

**Important:** An ended subscription is not automatically the same as a churned customer in every business model. You need to understand cancellation, reactivation, trial expiration and billing rules.

## **A14. Pattern 13 — Find the highest-selling product**

**Problem:** Find the product generating the highest total sales.

Assume:

**order_items**

| order_id | product_id | quantity | unit_price |
| -------- | ---------- | -------- | ---------- |
| 101      | 1          | 2        | 500        |
| 101      | 2          | 1        | 800        |
| 102      | 1          | 3        | 500        |
| 103      | 3          | 2        | 400        |

Solution:

```
SELECT
    product_id,
    SUM(quantity * unit_price) AS total_sales
FROM order_items
GROUP BY product_id
ORDER BY total_sales DESC
LIMIT 1;
```

If the interviewer asks for all products tied for the highest revenue:

```
WITH sales AS (
    SELECT product_id, SUM(quantity * unit_price) AS total_sales
    FROM order_items
    GROUP BY product_id
),
ranked AS (
    SELECT
        product_id,
        total_sales,
        RANK() OVER (ORDER BY total_sales DESC) AS rnk
    FROM sales
)
SELECT product_id, total_sales
FROM ranked
WHERE rnk = 1;
```

### **Pattern recognition summary**

| Problem wording                 | Think about                      |
| ------------------------------- | -------------------------------- |
| Second-highest distinct value   | MAX + subquery / DENSE_RANK      |
| Top N per group                 | ROW_NUMBER / RANK / DENSE_RANK   |
| Above group average             | Window AVG / correlated subquery |
| Duplicates                      | GROUP BY + HAVING                |
| Remove duplicate rows           | ROW_NUMBER                       |
| Never has a match               | NOT EXISTS                       |
| Has at least one match          | EXISTS                           |
| Previous month's value          | LAG                              |
| Next month's value              | LEAD                             |
| Cumulative amount               | SUM OVER                         |
| Consecutive dates               | LAG / gaps and islands           |
| Monthly growth                  | DATE_TRUNC + LAG                 |
| Customer returning after signup | First activity + self-join       |
| All tied highest values         | MAX or DENSE_RANK                |
| Highest sales product           | GROUP BY + SUM + ranking         |

# **SECTION B — LEETCODE SQL 50 PREPARATION**

Structured coding practice

LeetCode SQL 50 is a curated SQL practice collection. We'll use it as a problem-solving curriculum rather than treating it as a list of queries to memorise.

Official practice collection: [LeetCode SQL 50 Study Plan](https://leetcode.com/studyplan/top-sql-50/).

## **B1. The five practice categories**

**1. Basic SELECT and filtering**

Foundation

SELECT, WHERE, DISTINCT, ORDER BY, LIMIT, LIKE, NULL handling

Suggested practice: 10 problems

**2. Basic JOINs**

INNER JOIN, LEFT JOIN, multiple tables, anti-joins

Suggested practice: 10 problems

**3. Aggregation and grouping**

COUNT, SUM, AVG, GROUP BY, HAVING, conditional aggregation

Suggested practice: 10 problems

**4. Advanced SELECT and subqueries**

NOT EXISTS, nested queries, date calculations, ranking logic

Suggested practice: 10 problems

**5. Advanced SQL**

OA priority

Window functions, CTEs, LAG, RANK, DENSE_RANK, running totals

Suggested practice: 10 problems

## **B2. Recommended problem progression**

Here are representative SQL 50 problems and the patterns to learn from them. The list below is a curated practice map, not a claim that these are the exact current ordering of all 50 official exercises.

| Problem                                                | Main concept                       |
| ------------------------------------------------------ | ---------------------------------- |
| Recyclable and Low Fat Products                        | Multiple WHERE conditions          |
| Find Customer Referee                                  | NULL and inequality                |
| Big Countries                                          | OR conditions                      |
| Article Views I                                        | DISTINCT                           |
| Invalid Tweets                                         | LENGTH                             |
| Replace Employee ID With Unique Identifier             | LEFT JOIN                          |
| Product Sales Analysis I                               | JOIN                               |
| Customer Who Visited but Did Not Make Any Transactions | LEFT JOIN + anti-join              |
| Rising Temperature                                     | Self join / date comparison        |
| Average Time of Process per Machine                    | Conditional aggregation            |
| Employee Bonus                                         | LEFT JOIN + NULL                   |
| Students and Examinations                              | CROSS JOIN + LEFT JOIN             |
| Managers with at Least 5 Direct Reports                | GROUP BY + HAVING                  |
| Confirmation Rate                                      | JOIN + conditional aggregation     |
| Not Boring Movies                                      | Filtering + ordering               |
| Average Selling Price                                  | JOIN + weighted average            |
| Project Employees I                                    | JOIN + AVG                         |
| Percentage of Users Attended a Contest                 | JOIN + percentage calculation      |
| Queries Quality and Percentage                         | Aggregation + arithmetic           |
| Monthly Transactions I                                 | DATE_TRUNC equivalent + GROUP BY   |
| Immediate Food Delivery II                             | First-order analysis               |
| Game Play Analysis IV                                  | First activity + retention         |
| Product Sales Analysis III                             | First year per product             |
| Last Person to Fit in the Bus                          | Running total                      |
| Count Salary Categories                                | Conditional aggregation            |
| Group Sold Products By The Date                        | STRING_AGG                         |
| List the Products Ordered in a Period                  | GROUP BY + HAVING                  |
| Find Users With Valid E-Mails                          | Pattern matching                   |
| Second Highest Salary                                  | Scalar subquery                    |
| Department Top Three Salaries                          | DENSE_RANK + PARTITION             |
| Nth Highest Salary                                     | Function / ranking                 |
| Rank Scores                                            | RANK                               |
| Consecutive Numbers                                    | GROUP BY / window functions        |
| Exchange Seats                                         | CASE + row numbering               |
| Department Highest Salary                              | Subquery or window rank            |
| Tree Node                                              | CASE + EXISTS                      |
| Find Median Employee Salary                            | Window ranking                     |
| Investments in 2016                                    | Grouping + conditional aggregation |
| Restaurant Growth                                      | Running totals and moving averages |

These examples cover a broad portion of the skills required in SQL assessments. The official collection also includes additional problems covering the same fundamentals in different schemas.

## **B3. How to solve every LeetCode SQL problem**

Follow the same procedure every time.

## **Five-step problem-solving method**

1. **Understand the output**

   What columns must the final result contain? One row per employee, customer, product, date or group?
2. **Inspect the schema**

   Find primary keys, foreign keys, possible duplicates and NULL values.
3. **Identify the SQL pattern**

   Filtering, JOIN, aggregation, subquery, CTE or window function?
4. **Write the simplest correct query**

   Start with the correct logic. Optimise only after correctness.
5. **Test edge cases**

   Duplicates, NULLs, ties, empty groups, multiple rows and missing dates.

## **B4. Important SQL dialect differences**

LeetCode database problems commonly use MySQL syntax. You have been learning PostgreSQL, so pay attention to these differences.

| Operation                | PostgreSQL             | MySQL                      |
| ------------------------ | ---------------------- | -------------------------- |
| Top N                    | LIMIT                  | LIMIT                      |
| Current date             | CURRENT_DATE           | CURRENT_DATE()             |
| Date truncation to month | DATE_TRUNC('month', d) | DATE_FORMAT(d, '%Y-%m-01') |
| Split string             | SPLIT_PART()           | SUBSTRING_INDEX()          |
| String aggregation       | STRING_AGG()           | GROUP_CONCAT()             |
| Conditional count        | FILTER or CASE         | Usually CASE               |
| Nth-highest salary       | DENSE_RANK / OFFSET    | DENSE_RANK / LIMIT OFFSET  |
| Recursive CTE            | WITH RECURSIVE         | WITH RECURSIVE             |
| Full outer join          | Supported              | Usually emulate with UNION |
| Date difference          | Date subtraction       | DATEDIFF()                 |

**My recommendation:** Keep learning the concepts in PostgreSQL, but practise platform questions in the dialect specified by the platform. Query logic transfers; syntax may not.

## **B5. LeetCode practice tracking**

**SQL 50 tracker**

# **0/50**

**Basic SELECT and filtering**

0/10

12345678910

**Basic JOINs**

0/10

11121314151617181920

**Aggregation and grouping**

0/10

21222324252627282930

**Subqueries and advanced SELECT**

0/10

31323334353637383940

**Advanced SQL and window functions**

0/10

41424344454647484950

The tracker is for marking your practice progress. Use the official study plan to access the actual exercises.

# **SECTION C — TECHNICAL SQL INTERVIEW PREPARATION**

SDE and Data Engineer interview preparation

In an actual technical interview, SQL questions usually come in three forms:

1. Write a query for a given problem.
2. Explain a database concept.
3. Diagnose a query or database performance issue.

We'll prepare for all three.

## **C1. SQL query-writing interview questions**

These are some of the highest-value problems to practise.

| #  | Question                                                           | Main technique            |
| -- | ------------------------------------------------------------------ | ------------------------- |
| 1  | Find the second-highest salary.                                    | MAX + subquery            |
| 2  | Find the Nth-highest salary.                                       | DENSE_RANK                |
| 3  | Find the highest-paid employee per department.                     | GROUP BY / window         |
| 4  | Find the top three employees per department.                       | DENSE_RANK                |
| 5  | Find employees earning more than their manager.                    | SELF JOIN                 |
| 6  | Find employees earning above department average.                   | Correlated query          |
| 7  | Find duplicate emails.                                             | GROUP BY + HAVING         |
| 8  | Delete duplicate records while retaining one.                      | ROW_NUMBER                |
| 9  | Find customers without orders.                                     | NOT EXISTS                |
| 10 | Find products never sold.                                          | LEFT JOIN / NOT EXISTS    |
| 11 | Find the department with the highest average salary.               | GROUP BY + ORDER BY       |
| 12 | Find departments with more employees than average department size. | CTE + HAVING              |
| 13 | Find consecutive login dates.                                      | LAG                       |
| 14 | Find customers who ordered in consecutive months.                  | LAG / date logic          |
| 15 | Calculate running revenue.                                         | SUM OVER                  |
| 16 | Calculate month-over-month growth.                                 | LAG + aggregation         |
| 17 | Find the median salary.                                            | PERCENTILE_CONT / ranking |
| 18 | Find the percentage of successful transactions.                    | Conditional aggregation   |
| 19 | Find the first order of each customer.                             | ROW_NUMBER / MIN          |
| 20 | Find customers who purchased every product in a category.          | GROUP BY + HAVING         |

### **Let's work through some more difficult examples.**

### **Problem 1 — Find the department with the highest average salary**

```
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
ORDER BY avg_salary DESC
LIMIT 1;
```

What if two departments have the same average?

Use ranking:

```
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),
ranked AS (
    SELECT
        department_id,
        avg_salary,
        RANK() OVER (ORDER BY avg_salary DESC) AS rnk
    FROM dept_avg
)
SELECT department_id, avg_salary
FROM ranked
WHERE rnk = 1;
```

This returns all departments tied for the highest average.

### **Problem 2 — Find employees who have the same salary**

```
SELECT salary, COUNT(*) AS employee_count
FROM employees
GROUP BY salary
HAVING COUNT(*) > 1;
```

To retrieve the employee records:

```
SELECT *
FROM employees
WHERE salary IN (
    SELECT salary
    FROM employees
    GROUP BY salary
    HAVING COUNT(*) > 1
)
ORDER BY salary;
```

### **Problem 3 — Find employees with no manager**

```
SELECT emp_id, name
FROM employees
WHERE manager_id IS NULL;
```

To find employees whose manager ID references no existing employee:

```
SELECT e.emp_id, e.name, e.manager_id
FROM employees e
WHERE e.manager_id IS NOT NULL
  AND NOT EXISTS (
      SELECT 1
      FROM employees m
      WHERE m.emp_id = e.manager_id
  );
```

These represent different situations:

- No manager assigned.
- Manager ID assigned, but the referenced manager does not exist.

The second should not happen when an appropriate self-referencing foreign key is enforced, but it is useful for diagnosing legacy or inconsistent data.

### **Problem 4 — Find customers who purchased every available product**

This is called a *relational division* pattern.

Tables:

- `customers(customer_id)`
- `products(product_id)`
- `orders(order_id, customer_id)`
- `order_items(order_id, product_id)`

Solution using double NOT EXISTS:

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

Read it in English:

Find customers for whom there does not exist a product that they have not purchased.

This is a powerful way of translating universal requirements into SQL.

An alternative uses aggregation:

```
SELECT o.customer_id
FROM orders o
JOIN order_items oi
    ON oi.order_id = o.order_id
GROUP BY o.customer_id
HAVING COUNT(DISTINCT oi.product_id) = (
    SELECT COUNT(*) FROM products
);
```

This second solution assumes the intended requirement is purchasing at least one unit of every currently available product and that `products` is nonempty.

### **Problem 5 — Find median salary**

PostgreSQL provides a direct solution using ordered-set aggregate functions.

```
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;
```

`PERCENTILE_CONT` calculates a continuous percentile and can interpolate between values.

There is also:

```
SELECT
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;
```

`PERCENTILE_DISC` returns an actual value from the dataset.

**Interview distinction:**

- Continuous median: May be a value between two observations.
- Discrete median: Returns an actual ordered observation.

## **C2. DBMS theoretical interview questions**

These questions go beyond SQL syntax.

| #  | Question                        | What the interviewer expects                   |
| -- | ------------------------------- | ---------------------------------------------- |
| 1  | What is a relational database?  | Relations, tuples, attributes                  |
| 2  | What is a primary key?          | Unique, non-NULL identifier                    |
| 3  | What is a foreign key?          | Referential integrity                          |
| 4  | What is normalisation?          | Redundancy and anomaly reduction               |
| 5  | What is 2NF?                    | No partial dependency                          |
| 6  | What is 3NF?                    | Formal FD condition                            |
| 7  | What is BCNF?                   | Every determinant is a superkey                |
| 8  | What is a transaction?          | Logical unit of work                           |
| 9  | Explain ACID.                   | Atomicity, consistency, isolation, durability  |
| 10 | What is an index?               | Additional retrieval structure                 |
| 11 | What is a B-Tree?               | Balanced search structure                      |
| 12 | What is a composite index?      | Multiple indexed columns                       |
| 13 | What is a deadlock?             | Circular waiting                               |
| 14 | What is MVCC?                   | Version-based concurrency control              |
| 15 | What is isolation?              | Control of concurrent transaction interactions |
| 16 | What is a materialised view?    | Persisted query result                         |
| 17 | What is a stored procedure?     | Database-side procedural routine               |
| 18 | What is a trigger?              | Automatically executed database logic          |
| 19 | What is a view?                 | Named query definition                         |
| 20 | What is a query execution plan? | Database execution strategy                    |

### **Interview-ready answers**

**Q1. What is the difference between DELETE, TRUNCATE and DROP?**

DELETE removes selected rows and supports WHERE. TRUNCATE removes all rows efficiently without a row-by-row DELETE operation. DROP removes the database object itself, including its definition.

**Q2. What is an index?**

An index is an additional data structure that allows databases to locate records efficiently without scanning every table row. It improves selected reads but requires storage and maintenance during data modifications.

**Q3. What is a composite index?**

A composite index contains multiple columns. Its column order matters because the leading columns determine which lookup and range conditions the index can efficiently support.

**Q4. What is MVCC?**

MVCC maintains multiple versions of records so concurrent transactions can read consistent snapshots while modifications occur. It reduces conflicts between ordinary readers and writers.

**Q5. What is a deadlock?**

A deadlock occurs when transactions wait on resources held by each other, creating a circular dependency. The database detects the cycle and aborts a transaction to restore progress.

**Q6. What is the difference between RANK and DENSE_RANK?**

Both assign the same rank to tied values. RANK leaves gaps after ties, whereas DENSE_RANK assigns consecutive ranks without gaps.

**Q7. What is the difference between a correlated and non-correlated subquery?**

A correlated subquery references a column from the outer query and depends on its current row. A non-correlated subquery is independent of the outer query.

**Q8. Why is SELECT \* not always recommended?**

It retrieves unnecessary columns, increases data transfer and can prevent covering-index opportunities. Selecting only required columns improves clarity and can reduce execution costs.

**Q9. What is a materialised view?**

A materialised view stores the result of a query physically. It can speed up repeated analytical reads but requires a refresh strategy to keep its data sufficiently current.

**Q10. What is query optimisation?**

Query optimisation identifies an efficient execution strategy using access paths, join algorithms, statistics and estimated costs to reduce execution time and resource consumption.

## **C3. Debugging interview questions**

Interviewers may provide incorrect SQL rather than ask you to write it from scratch.

### **Example 1 — Incorrect aggregate filtering**

Incorrect:

```
SELECT department_id, AVG(salary)
FROM employees
WHERE AVG(salary) > 70000
GROUP BY department_id;
```

Problem: Aggregate filtering cannot normally be performed in WHERE.

Correct:

```
SELECT department_id, AVG(salary)
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 70000;
```

### **Example 2 — Incorrect NULL checking**

Incorrect:

```
SELECT *
FROM employees
WHERE manager_id = NULL;
```

`NULL = NULL` evaluates to UNKNOWN, so no rows are returned.

Correct:

```
SELECT *
FROM employees
WHERE manager_id IS NULL;
```

### **Example 3 — Accidental duplicate multiplication**

Incorrect:

```
SELECT SUM(o.amount) AS total_order_value
FROM orders o
JOIN order_items oi
    ON oi.order_id = o.order_id;
```

If an order has multiple order items, its order amount gets repeated.

Correct approach: Aggregate orders independently, or aggregate order items appropriately before joining.

```
SELECT SUM(o.amount) AS total_order_value
FROM orders o
WHERE EXISTS (
    SELECT 1
    FROM order_items oi
    WHERE oi.order_id = o.order_id
);
```

### **Example 4 — Filtering a window function in WHERE**

Incorrect:

```
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees
WHERE rn <= 3;
```

Correct:

```
WITH ranked AS (
    SELECT
        name,
        salary,
        ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT name, salary
FROM ranked
WHERE rn <= 3;
```

### **Example 5 — LEFT JOIN accidentally behaving like INNER JOIN**

Incorrect for retaining departments without active employees:

```
SELECT d.department_name, e.name
FROM departments d
LEFT JOIN employees e
    ON e.department_id = d.department_id
WHERE e.status = 'Active';
```

The WHERE condition rejects the NULL-extended rows, so departments with no matching employee disappear.

Correct:

```
SELECT d.department_name, e.name
FROM departments d
LEFT JOIN employees e
    ON e.department_id = d.department_id
   AND e.status = 'Active';
```

## **C4. How to handle an SQL interview**

A strong candidate does not immediately start typing.

Use this interview response framework:

**Step 1 — Clarify the output**

Ask yourself:

- One row per employee?
- One row per department?
- One row per customer?
- One row per date?

**Step 2 — Understand relationships**

Identify:

- Primary key.
- Foreign key.
- One-to-many relationship.
- Possible duplicate-producing JOINs.

**Step 3 — Identify the logic**

Determine whether you need:

- Filtering.
- Aggregation.
- JOIN.
- Subquery.
- CTE.
- Window function.

**Step 4 — Handle edge cases**

Always consider:

- NULL.
- Duplicate values.
- Ties.
- Empty results.
- Division by zero.
- Missing relationships.

**Step 5 — Optimise**

Discuss:

- Useful indexes.
- Query execution plan.
- Selectivity.
- Unnecessary sorting.
- Repeated calculations.

### **SQL interview scoring framework**

| Area               | What a strong candidate demonstrates    |
| ------------------ | --------------------------------------- |
| Correctness        | Query returns the right records         |
| Clarity            | SQL is understandable                   |
| Edge cases         | Handles duplicates, NULL and ties       |
| Efficiency         | Avoids unnecessary work                 |
| Explanation        | Can justify why the query works         |
| Database knowledge | Understands indexes and execution plans |

# **SECTION D — TIMED MOCK SQL ASSESSMENTS**

Interview simulation

Now we move to actual testing.

You'll attempt three assessments:

- Mock OA 1: Easy — 15 minutes.
- Mock OA 2: Medium — 25 minutes.
- Mock OA 3: Hard — 35 minutes.

These are original practice assessments, not claimed to be actual company previous-year papers.

## **D1. Mock OA 1 — Easy**

Easy

15 minutes · 5 questions · 20 marks

**Schema**

`employees(emp_id, name, department_id, salary, city)`

Questions:

| # | Problem                                       | Marks |
| - | --------------------------------------------- | ----- |
| 1 | Find all employees earning more than 70,000.  | 3     |
| 2 | Find the unique cities of employees.          | 3     |
| 3 | Find total employees in each department.      | 4     |
| 4 | Find departments with more than one employee. | 5     |
| 5 | Find the second-highest distinct salary.      | 5     |

**Mock OA 1 — Answer sheet**

5 questions

**Q1. Find employees earning more than 70,000.**

SELECT \* FROM employees WHERE salary > 70000;

SELECT \* FROM employees HAVING salary > 70000;

SELECT salary FROM employees GROUP BY salary;

**Q2. Find unique cities.**

SELECT city FROM employees;

SELECT DISTINCT city FROM employees;

SELECT city, COUNT(\*) FROM employees;

**Q3. Count employees in every department.**

SELECT department_id, COUNT(\*) FROM employees;

SELECT department_id, COUNT(\*) FROM employees GROUP BY department_id;

SELECT COUNT(department_id) FROM employees;

**Q4. Find departments having more than one employee.**

WHERE COUNT(\*) > 1

GROUP BY department_id HAVING COUNT(\*) > 1

ORDER BY COUNT(\*) > 1

**Q5. Find the second-highest distinct salary.**

SELECT MAX(salary) FROM employees;

SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;

SELECT MIN(salary) FROM employees;


**Mock OA 1 — Answer key and explanations**

| Question | Correct option | Why |
| -------- | -------------- | --- |
| Q1 (3 marks) | `SELECT * FROM employees WHERE salary > 70000;` | Row-level filtering belongs in WHERE. HAVING needs grouping, and GROUP BY alone does not filter. |
| Q2 (3 marks) | `SELECT DISTINCT city FROM employees;` | DISTINCT removes duplicate cities. The COUNT option has no GROUP BY, so it is invalid. |
| Q3 (4 marks) | `SELECT department_id, COUNT(*) FROM employees GROUP BY department_id;` | Every non-aggregated selected column must appear in GROUP BY. The last option returns one total, not one row per department. |
| Q4 (5 marks) | `GROUP BY department_id HAVING COUNT(*) > 1` | Aggregates cannot be filtered in WHERE, and `ORDER BY COUNT(*) > 1` only sorts. |
| Q5 (5 marks) | `SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;` | DISTINCT keeps ties from consuming ranks. MAX and MIN return the highest and lowest values, not the second. |

Alternative for Q5, more robust and extendable to the Nth-highest:

```
SELECT salary
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 2
LIMIT 1;
```

**Scoring:** 18-20 excellent, 14-17 good, below 14 revisit Lessons 1-2 before moving on.

## **D2. Mock OA 2 — Medium**

Medium

25 minutes · 5 questions · 30 marks

**Schema:** use `employees`, `departments`, `customers` and `orders` exactly as shown in section A1.

| # | Problem | Marks |
| - | ------- | ----- |
| 1 | Find customers who have never placed an order. | 4 |
| 2 | Display every department with its employee count, including departments with zero employees. | 6 |
| 3 | Find employees earning more than their department's average salary. | 6 |
| 4 | Find the top two highest-paid employees in each department. | 7 |
| 5 | Calculate monthly completed revenue and its month-over-month percentage change. | 7 |

Write all five queries before checking the key below.

**Mock OA 2 — Answer key**

**Q1 — Anti-join (4 marks).** Expected: Neha.

```
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

Marking: 2 for the correct pattern (NOT EXISTS or LEFT JOIN ... IS NULL), 2 for correct output columns. `NOT IN` is acceptable here only if you note the NULL risk.

**Q2 — LEFT JOIN with COUNT on the right-hand column (6 marks).**

```
SELECT
    d.department_name,
    COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e
    ON e.department_id = d.department_id
GROUP BY d.department_id, d.department_name
ORDER BY employee_count DESC, d.department_name;
```

| department_name | employee_count |
| --------------- | -------------- |
| Engineering     | 3              |
| HR              | 2              |
| Finance         | 1              |
| Marketing       | 1              |
| Operations      | 0              |

Marking: 2 for LEFT JOIN from departments, 2 for `COUNT(e.emp_id)` rather than `COUNT(*)`, 2 for grouping correctly. Using `COUNT(*)` returns 1 for Operations and loses 2 marks.

**Q3 — Above department average (6 marks).** Expected: Rahul (80000), Priya (75000), Simran (85000).

```
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT e.name, e.department_id, e.salary
FROM employees e
JOIN dept_avg d
    ON d.department_id = e.department_id
WHERE e.salary > d.avg_salary
ORDER BY e.department_id, e.salary DESC;
```

Department averages: 101 → 73333.33, 102 → 65000, 103 → 90000, 104 → 70000. Neha and Riya equal their own department's average, so they are not above it. A correlated subquery or `AVG() OVER (PARTITION BY ...)` inside a CTE earns full marks too.

**Q4 — Top N per group (7 marks).**

```
WITH ranked AS (
    SELECT
        emp_id,
        name,
        department_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC, emp_id
        ) AS rn
    FROM employees
)
SELECT name, department_id, salary, rn
FROM ranked
WHERE rn <= 2
ORDER BY department_id, rn;
```

| name   | department_id | salary | rn |
| ------ | ------------- | ------ | -- |
| Simran | 101           | 85000  | 1  |
| Priya  | 101           | 75000  | 2  |
| Rahul  | 102           | 80000  | 1  |
| Karan  | 102           | 50000  | 2  |
| Neha   | 103           | 90000  | 1  |
| Riya   | 104           | 70000  | 1  |

Marking: 3 for PARTITION BY, 2 for a ranking function in a CTE or subquery (not directly in WHERE), 2 for a deterministic tie-breaker or a justified choice of DENSE_RANK.

**Q5 — Month-over-month growth (7 marks).**

```
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month_start,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'Completed'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    month_start,
    revenue,
    LAG(revenue) OVER (ORDER BY month_start) AS previous_revenue,
    ROUND(
        100.0 * (revenue - LAG(revenue) OVER (ORDER BY month_start))
        / NULLIF(LAG(revenue) OVER (ORDER BY month_start), 0),
        2
    ) AS growth_percentage
FROM monthly
ORDER BY month_start;
```

| month_start | revenue | previous_revenue | growth_percentage |
| ----------- | ------- | ---------------- | ----------------- |
| 2026-01-01  | 1300    | NULL             | NULL              |
| 2026-02-01  | 1200    | 1300             | -7.69             |
| 2026-03-01  | 700     | 1200             | -41.67            |

Marking: 2 for filtering Completed (the Pending order of 300 must be excluded from February), 2 for DATE_TRUNC grouping, 2 for LAG, 1 for NULLIF and `100.0` to avoid integer division.

**Scoring:** 26-30 strong, 20-25 passing, below 20 revisit JOINs, window functions and the Section A patterns.

## **D3. Mock OA 3 — Hard**

Hard

35 minutes · 4 questions · 40 marks

**Schema**

- `employees`, `departments`, `customers`, `orders` as in section A1.
- `logins(user_id, login_date)` as in pattern A11.
- `products(product_id)` containing products 1, 2 and 3.
- `order_items(order_id, product_id, quantity, unit_price)` as in pattern A14. Orders 101 and 103 belong to customer 1, order 102 to customer 2.

| # | Problem | Marks |
| - | ------- | ----- |
| 1 | Find the department with the second-highest average salary, returning every department on a tie. | 8 |
| 2 | For every employee who manages anyone, count all direct and indirect subordinates. | 10 |
| 3 | Find users with at least three consecutive login days and report each streak. | 10 |
| 4 | Find customers who have purchased every product, and explain your approach. | 12 |

**Mock OA 3 — Answer key**

**Q1 — Second-highest group aggregate (8 marks).** Expected: department 101.

```
WITH dept_avg AS (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),
ranked AS (
    SELECT
        department_id,
        avg_salary,
        DENSE_RANK() OVER (ORDER BY avg_salary DESC) AS rnk
    FROM dept_avg
)
SELECT department_id, avg_salary
FROM ranked
WHERE rnk = 2;
```

Ranking: 103 (90000), 101 (73333.33), 104 (70000), 102 (65000). DENSE_RANK returns all departments tied at the second average, and a `LIMIT 1 OFFSET 1` answer silently drops tied departments. Marking: 3 for aggregating first, 3 for DENSE_RANK, 2 for handling ties.

**Q2 — Recursive subordinate count (10 marks).**

```
WITH RECURSIVE reports AS (
    SELECT manager_id AS ancestor_id, emp_id AS descendant_id
    FROM employees
    WHERE manager_id IS NOT NULL

    UNION ALL

    SELECT r.ancestor_id, e.emp_id
    FROM reports r
    JOIN employees e
        ON e.manager_id = r.descendant_id
)
SELECT
    m.emp_id,
    m.name,
    COUNT(r.descendant_id) AS total_subordinates
FROM employees m
JOIN reports r
    ON r.ancestor_id = m.emp_id
GROUP BY m.emp_id, m.name
ORDER BY total_subordinates DESC, m.emp_id;
```

| emp_id | name   | total_subordinates |
| ------ | ------ | ------------------ |
| 1      | Aman   | 6                  |
| 2      | Rahul  | 3                  |
| 3      | Priya  | 1                  |
| 4      | Neha   | 1                  |

Rahul manages Neha and Karan directly and Riya indirectly through Neha. Marking: 4 for a correct anchor and recursive member, 3 for carrying the ancestor through the recursion, 2 for the final aggregation, 1 for noting that cyclic data needs a depth limit or cycle check.

**Q3 — Gaps and islands (10 marks).** Expected: user 1, Jan 01 to Jan 03, 3 days.

```
WITH distinct_logins AS (
    SELECT DISTINCT user_id, login_date
    FROM logins
),
numbered AS (
    SELECT
        user_id,
        login_date,
        login_date - (
            ROW_NUMBER() OVER (
                PARTITION BY user_id
                ORDER BY login_date
            )
        )::INT AS grp
    FROM distinct_logins
)
SELECT
    user_id,
    MIN(login_date) AS streak_start,
    MAX(login_date) AS streak_end,
    COUNT(*) AS streak_days
FROM numbered
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

Why it works: within a run of consecutive dates, the date and the row number both increase by one, so their difference is constant. A gap changes the difference and starts a new group. Marking: 3 for de-duplicating, 4 for the date-minus-row-number grouping key, 3 for the aggregation and HAVING.

**Q4 — Relational division (12 marks).** Expected: customer 1 (products 1, 2 and 3 across orders 101 and 103). Customer 2 bought only product 1.

```
SELECT o.customer_id
FROM orders o
JOIN order_items oi
    ON oi.order_id = o.order_id
GROUP BY o.customer_id
HAVING COUNT(DISTINCT oi.product_id) = (
    SELECT COUNT(*) FROM products
);
```

Alternative using double NOT EXISTS: "customers for whom no product exists that they have not bought."

```
SELECT c.customer_id
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM products p
    WHERE NOT EXISTS (
        SELECT 1
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.order_id
        WHERE o.customer_id = c.customer_id
          AND oi.product_id = p.product_id
    )
);
```

Marking: 4 for a correct approach, 4 for `COUNT(DISTINCT ...)` (plain COUNT would count repeat purchases twice), 2 for comparing against the product count, 2 for explaining the assumption that `products` is nonempty and the difference between the two methods. The NOT EXISTS version also returns customers with zero orders only if `products` is empty, which is the vacuous-truth edge case.

**Scoring:** 34-40 interview ready, 26-33 solid, below 26 revisit window functions, recursive CTEs and Section A patterns.

# **SECTION E — FINAL REVISION AND NEXT STEPS**

## **E1. One-page pattern cheat sheet**

| Need | Tool |
| ---- | ---- |
| Filter rows | WHERE |
| Filter groups | GROUP BY + HAVING |
| Nth-highest distinct | DENSE_RANK |
| Top N per group | ROW_NUMBER with PARTITION BY |
| Above group average | CTE with AVG, or correlated subquery |
| Duplicates | GROUP BY + HAVING, or ROW_NUMBER |
| No match | NOT EXISTS, or LEFT JOIN ... IS NULL |
| At least one match | EXISTS |
| Every match (division) | Double NOT EXISTS, or COUNT(DISTINCT) = total |
| Previous/next row | LAG / LEAD |
| Running total | SUM OVER with a ROWS frame |
| Consecutive streaks | Date minus ROW_NUMBER |
| Hierarchies | WITH RECURSIVE |
| Safe division | NULLIF(denominator, 0) |
| Conditional counts | SUM(CASE ...) or FILTER |

## **E2. The ten mistakes that cost the most marks**

1. Filtering an aggregate in WHERE.
2. Comparing with `= NULL` instead of `IS NULL`.
3. Using `COUNT(*)` after a LEFT JOIN when zero counts matter.
4. Putting a right-table filter in WHERE after a LEFT JOIN.
5. Using `LIMIT 1 OFFSET 1` when ties or distinct values matter.
6. Ranking functions with no tie-breaker.
7. Filtering a window function in WHERE.
8. Summing a column after a one-to-many JOIN.
9. Integer division in percentages (use `100.0`).
10. Ignoring the SQL dialect the assessment actually uses.

## **E3. Four-week practice plan**

| Week | Focus | Target |
| ---- | ----- | ------ |
| 1 | Filtering, JOINs, aggregation | LeetCode SQL 50 categories 1-3 (30 problems) |
| 2 | Subqueries, CTEs, anti-joins | Category 4 (10 problems) plus redo Mock OA 1 and 2 untimed |
| 3 | Window functions, gaps and islands, retention | Category 5 (10 problems) plus timed Mock OA 3 |
| 4 | DBMS theory, optimisation, concurrency | Answer Section C2 aloud, explain EXPLAIN output, walk through Lesson 9 scenarios |

Redo any problem you could not solve in under 10 minutes, and explain every solution aloud as if to an interviewer.

## **E4. Lesson 10 completion checklist**

- Recognise and write the 14 OA patterns in Section A without notes.
- Complete the LeetCode SQL 50 tracker.
- Answer the 20 DBMS theory questions in Section C2 aloud.
- Debug the five incorrect queries in Section C3 without looking at the fixes.
- Score at least 80% on each of the three mock assessments.

## **E5. Course progress**

**SQL + DBMS Mastery**

10 / 10 lessons

01. SQL Fundamentals - Covered
02. Aggregate Functions, GROUP BY, HAVING - Covered
03. SQL JOINs - Covered
04. Subqueries - Covered
05. CTEs and Recursive CTEs - Covered
06. Window Functions - Covered
07. Advanced SQL Functions - Covered
08. Database Design and Normalisation - Covered
09. Indexes, Transactions and Optimisation - Covered
10. Company OAs and Mock Interviews - Covered

**Next:** Lesson 11 continues with advanced drill sets and a 100-question SQL and DBMS interview bank.

