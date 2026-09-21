# **SQL Mastery — Advanced Practice Series**

We've completed the 10-part SQL and DBMS curriculum. Now the focus shifts from learning concepts to developing **independent problem-solving ability**.

We'll follow an actual coding-assessment approach: understand the schema, identify the pattern, write the query, test edge cases, and analyse complexity.

Our practice progression will be:

1. SQL OA Set 1 — Fundamentals, filtering and aggregation.
2. SQL OA Set 2 — JOINs and multi-table problems.
3. SQL OA Set 3 — Subqueries and CTEs.
4. SQL OA Set 4 — Window functions and ranking.
5. SQL OA Set 5 — Advanced analytics and complex problems.
6. SQL OA Set 6 — Mixed company-style assessments.

# **SQL OA Set 1 — Fundamentals and Aggregation**



Easy → Medium

&#x20;10 problems · PostgreSQL



## **1. Understand the database schema**

We'll use three tables.

**employees**

| emp_id | name   | department_id | salary | city   |
| ------ | ------ | ------------- | ------ | ------ |
| 1      | Aman   | 101           | 60000  | Delhi  |
| 2      | Rahul  | 102           | 80000  | Mumbai |
| 3      | Priya  | 101           | 75000  | Delhi  |
| 4      | Neha   | 103           | 90000  | Pune   |
| 5      | Karan  | 102           | 50000  | Mumbai |
| 6      | Simran | 101           | 85000  | Delhi  |
| 7      | Riya   | 104           | 70000  | Pune   |
| 8      | Dev    | 103           | 65000  | Mumbai |
| 9      | Ankit  | 104           | 55000  | Delhi  |
| 10     | Meera  | 101           | 95000  | Mumbai |

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
| 5           | Karan         | Mumbai |

**orders**

| order_id | customer_id | amount | status    |
| -------- | ----------- | ------ | --------- |
| 101      | 1           | 500    | Completed |
| 102      | 2           | 800    | Completed |
| 103      | 1           | 300    | Pending   |
| 104      | 3           | 1200   | Completed |
| 105      | 2           | 700    | Completed |
| 106      | 4           | 400    | Failed    |
| 107      | 3           | 600    | Completed |

# **2. Solve the OA problems**

## **Problem 1 — Employees earning more than ₹70,000**

Easy

**Problem:** Return employees earning more than 70,000, sorted by salary in descending order.

### **Step 1: Identify operations**

- Filter individual records → WHERE.
- Sort results → ORDER BY.

### **Step 2: Write the query**

```
SELECT
    emp_id,
    name,
    salary
FROM employees
WHERE salary > 70000
ORDER BY salary DESC;
```

Expected output:

| name   | salary |
| ------ | ------ |
| Meera  | 95000  |
| Neha   | 90000  |
| Simran | 85000  |
| Rahul  | 80000  |
| Priya  | 75000  |

**Complexity:** A basic table scan may require \\(O(N)\\) row inspections, followed by sorting up to \\(O(N\log N)\\). An appropriate index may change the execution plan.

**Interview explanation:** I use WHERE to filter employee records and ORDER BY DESC to arrange qualifying salaries from highest to lowest.

## **Problem 2 — Find distinct employee cities**

Easy

```
SELECT DISTINCT city
FROM employees
ORDER BY city;
```

Expected output:

| city   |
| ------ |
| Delhi  |
| Mumbai |
| Pune   |

**Key concept:** DISTINCT removes duplicate combinations from the selected columns.

Interview variation: Find the number of unique cities.

```
SELECT COUNT(DISTINCT city) AS total_cities
FROM employees;
```

Result: `3`.

## **Problem 3 — Find the top five highest-paid employees**

Easy

```
SELECT
    emp_id,
    name,
    salary
FROM employees
ORDER BY salary DESC, emp_id ASC
LIMIT 5;
```

Expected output:

| name   | salary |
| ------ | ------ |
| Meera  | 95000  |
| Neha   | 90000  |
| Simran | 85000  |
| Rahul  | 80000  |
| Priya  | 75000  |

Why include `emp_id`?

To ensure deterministic ordering when salaries are tied.

**Important:** LIMIT returns at most five rows. It does not guarantee five distinct salary values.

## **Problem 4 — Find employees earning between ₹60,000 and ₹85,000**

Easy

```
SELECT
    name,
    salary
FROM employees
WHERE salary BETWEEN 60000 AND 85000
ORDER BY salary DESC;
```

Remember:

\\[ 60000 \leq salary \leq 85000 \\]

BETWEEN is inclusive at both ends.

Expected employees:

| name   | salary |
| ------ | ------ |
| Simran | 85000  |
| Rahul  | 80000  |
| Priya  | 75000  |
| Riya   | 70000  |
| Dev    | 65000  |
| Aman   | 60000  |

## **Problem 5 — Count employees in every department**

Easy–Medium

**Problem:** Return every department and its employee count, including departments having zero employees.

This question tests whether you understand JOINs and aggregation together.

```
SELECT
    d.department_id,
    d.department_name,
    COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
GROUP BY
    d.department_id,
    d.department_name
ORDER BY d.department_id;
```

Expected output:

| department_name | employee_count |
| --------------- | -------------- |
| Engineering     | 4              |
| HR              | 2              |
| Finance         | 2              |
| Marketing       | 2              |
| Operations      | 0              |

**Why COUNT(e.emp_id), not COUNT(\*)?**\
For Operations, LEFT JOIN produces a row with NULL employee columns.

- COUNT(\*) counts the resulting row: 1.
- COUNT(e.emp_id) ignores NULL: 0.

Interview trap

## **Problem 6 — Find departments with average salary above ₹70,000**

Medium

```
SELECT
    d.department_name,
    AVG(e.salary) AS average_salary
FROM departments d
JOIN employees e
    ON d.department_id = e.department_id
GROUP BY
    d.department_id,
    d.department_name
HAVING AVG(e.salary) > 70000
ORDER BY average_salary DESC;
```

Expected output:

| department_name | average_salary |
| --------------- | -------------- |
| Engineering     | 78750          |
| Finance         | 77500          |

Why HAVING?

Because we need to filter departments after calculating their average salaries.

Check the two conditions:

```
WHERE salary > 70000
```

Filters individual employees.

```
HAVING AVG(salary) > 70000
```

Filters completed department groups.

## **Problem 7 — Find departments without employees**

Medium

This is an anti-join problem.

### **Approach 1: NOT EXISTS**

```
SELECT *
FROM departments d
WHERE NOT EXISTS (
    SELECT 1
    FROM employees e
    WHERE e.department_id = d.department_id
);
```

### **Approach 2: LEFT JOIN**

```
SELECT d.*
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
WHERE e.emp_id IS NULL;
```

Expected result:

| department_id | department_name |
| ------------- | --------------- |
| 105           | Operations      |

Both approaches are correct.

## **Problem 8 — Count transactions by status**

Medium

**Problem:** Find total transactions and total transaction amount for each status.

```
SELECT
    status,
    COUNT(*) AS transaction_count,
    SUM(amount) AS total_amount,
    AVG(amount) AS average_amount
FROM orders
GROUP BY status
ORDER BY transaction_count DESC;
```

Expected output:

| status    | transaction_count | total_amount | average_amount |
| --------- | ----------------- | ------------ | -------------- |
| Completed | 5                 | 3800         | 760            |
| Pending   | 1                 | 300          | 300            |
| Failed    | 1                 | 400          | 400            |

### **Interview variation: Calculate all statuses in one row**

```
SELECT
    COUNT(*) AS total_orders,

    COUNT(*) FILTER (
        WHERE status = 'Completed'
    ) AS completed_orders,

    COUNT(*) FILTER (
        WHERE status = 'Pending'
    ) AS pending_orders,

    COUNT(*) FILTER (
        WHERE status = 'Failed'
    ) AS failed_orders

FROM orders;
```

This uses PostgreSQL's aggregate FILTER clause.

## **Problem 9 — Find customers who have placed more than one order**

Medium

```
SELECT
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) AS total_orders
FROM customers c
JOIN orders o
    ON c.customer_id = o.customer_id
GROUP BY
    c.customer_id,
    c.customer_name
HAVING COUNT(o.order_id) > 1
ORDER BY total_orders DESC;
```

Expected output:

| customer_name | total_orders |
| ------------- | ------------ |
| Aman          | 2            |
| Rahul         | 2            |
| Priya         | 2            |

**Key concept:** When a question asks for customers with multiple records, think about:

```
GROUP BY customer_id
HAVING COUNT(*) > 1
```

## **Problem 10 — Calculate completed sales by customer city**

Medium

**Problem:** Find total completed order amounts for each customer city. Include cities with zero completed orders.

```
SELECT
    c.city,
    COALESCE(SUM(o.amount), 0) AS total_sales
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
   AND o.status = 'Completed'
GROUP BY c.city
ORDER BY total_sales DESC;
```

Expected output:

| city   | total_sales |
| ------ | ----------- |
| Delhi  | 2300        |
| Mumbai | 1500        |
| Pune   | 0           |

Two important concepts:

**1. Why put status in ON?**

We want to preserve customers even if they have no completed order.

**2. Why COALESCE?**

SUM over no matching non-NULL amounts returns NULL. COALESCE converts it to zero.

This is an excellent example of combining JOINs, NULL handling and aggregation.

# **3. Independent practice — Set 1**

Now try these without looking at the previous solutions.

| #  | Question                                                                           | Difficulty |
| -- | ---------------------------------------------------------------------------------- | ---------- |
| 1  | Find the third-highest distinct salary.                                            | Easy       |
| 2  | Find the employee with the minimum salary in each department.                      | Medium     |
| 3  | Find departments having more than two employees.                                   | Easy       |
| 4  | Find employees earning above the company average.                                  | Medium     |
| 5  | Find customers who have no completed orders.                                       | Medium     |
| 6  | Find the customer who spent the most on completed orders.                          | Medium     |
| 7  | Find departments where total salary expenditure exceeds ₹150,000.                  | Medium     |
| 8  | Find the second-highest distinct salary in each department.                        | Medium     |
| 9  | Find the percentage of completed orders out of all orders.                         | Medium     |
| 10 | Find all employees earning more than their department average and company average. | Hard       |

## **4. How to evaluate your solutions**

For every problem, check the following:

**Query evaluation checklist**

0/9

Query executes successfully

Returns the correct columns

Returns the correct rows

Handles duplicate values correctly

Handles NULL values correctly

Handles ties correctly

Uses the correct aggregation level

Can explain why each SQL clause is needed

Can identify a reasonable optimisation opportunity

**Next practice set:** SQL OA Set 2 — JOINs and multi-table problems, including customer-order analysis, missing relationships, self joins, multiple JOINs and aggregation across three or more tables.

Your first real mastery checkpoint is to solve the ten independent questions without referring to the worked solutions.


Yes! And this is actually a very important part of SQL interview preparation.

In many SDE, Data Engineer and SQL-focused interviews, the interviewer doesn't stop after you produce one correct query. They may say:

- "Can you solve this without using LIMIT?"
- "Can you solve it without a subquery?"
- "Can you solve it using a JOIN?"
- "Can you solve it using window functions?"
- "Can you solve it without using MAX()?"
- "Can you optimise your solution?"
- "What if duplicate salaries exist?"
- "What if there is no second-highest salary?"

**They are testing whether you understand SQL problem-solving or whether you have memorised a query.**

Let's prepare you for this properly.

# **Example: Find the second-highest distinct salary**

Consider:

**employees**

| emp_id | name   | salary |
| ------ | ------ | ------ |
| 1      | Aman   | 60000  |
| 2      | Rahul  | 80000  |
| 3      | Priya  | 75000  |
| 4      | Neha   | 90000  |
| 5      | Karan  | 50000  |
| 6      | Simran | 85000  |
| 7      | Riya   | 90000  |

Expected answer: ₹85,000

We will solve this in **six different ways**.

## **Method 1 — MAX() with subquery**

The simplest and most elegant solution.

```
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```

How it works:

1. Inner query finds 90,000.
2. WHERE removes salaries greater than or equal to 90,000.
3. Outer MAX finds 85,000.

**Concepts:** MAX + scalar subquery.

## **Method 2 — ORDER BY + LIMIT + OFFSET**

```
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

How it works:

1. DISTINCT removes duplicate salaries.
2. ORDER BY sorts in descending order.
3. OFFSET 1 skips the highest salary.
4. LIMIT 1 returns the second-highest.

**Concepts:** DISTINCT + sorting + pagination.

PostgreSQL and MySQL syntax.

## **Method 3 — DENSE_RANK()**

This is an advanced and particularly useful interview solution.

```
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees
)
SELECT DISTINCT salary
FROM ranked
WHERE salary_rank = 2;
```

Why DENSE_RANK?

Because identical salaries receive identical ranks without skipping subsequent ranks.

| Salary | DENSE_RANK |
| ------ | ---------- |
| 90000  | 1          |
| 90000  | 1          |
| 85000  | 2          |
| 80000  | 3          |
| 75000  | 4          |
| 60000  | 5          |
| 50000  | 6          |

**Concepts:** Window function + ranking + CTE.

## **Method 4 — Correlated subquery with COUNT()**

Now let's solve it without MAX(), LIMIT or ranking functions.

```
SELECT DISTINCT e1.salary
FROM employees e1
WHERE (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
) = 1;
```

Understand the logic.

For every distinct salary, count how many distinct salaries are greater than it.

| Salary | Number of distinct salaries greater | Qualifies? |
| ------ | ----------------------------------- | ---------- |
| 90000  | 0                                   | No         |
| 85000  | 1                                   | Yes        |
| 80000  | 2                                   | No         |
| 75000  | 3                                   | No         |
| 60000  | 5                                   | No         |

The second-highest salary has exactly one distinct salary greater than it.

This is a particularly good way to demonstrate problem-solving ability.

## **Method 5 — Self JOIN**

We can also use a JOIN to compare salaries.

```
SELECT DISTINCT e1.salary
FROM employees e1
JOIN employees e2
    ON e1.salary < e2.salary
GROUP BY e1.salary
HAVING COUNT(DISTINCT e2.salary) = 1;
```

Logic:

1. Join every lower salary with higher salaries.
2. Group by the lower salary.
3. Count distinct higher salaries.
4. Keep only salaries with exactly one distinct higher salary.

**Concepts:** SELF JOIN + GROUP BY + HAVING.

## **Method 6 — Nested derived table**

Another alternative:

```
SELECT MIN(salary) AS second_highest_salary
FROM (
    SELECT DISTINCT salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 2
) AS top_two;
```

Logic:

1. Find the top two distinct salaries.
2. Find the minimum of those two.

# **2. Comparison of all six approaches**

| Method | Techniques used     | Level        |
| ------ | ------------------- | ------------ |
| 1      | MAX + subquery      | Easy         |
| 2      | DISTINCT + LIMIT    | Easy         |
| 3      | DENSE_RANK + CTE    | Intermediate |
| 4      | Correlated COUNT    | Advanced     |
| 5      | SELF JOIN           | Advanced     |
| 6      | Derived table + MIN | Intermediate |

All six return 85,000 for our example.

However, they don't necessarily have identical execution costs. Their performance depends on indexes, table size, data distribution and database optimiser decisions.

# **3. What if the interviewer changes the requirement?**

This is where genuine SQL mastery begins.

### **Variation 1: Find the employee names having the second-highest salary**

```
SELECT *
FROM employees
WHERE salary = (
    SELECT MAX(salary)
    FROM employees
    WHERE salary < (
        SELECT MAX(salary)
        FROM employees
    )
);
```

This returns employee records with the second-highest salary.

### **Variation 2: Find the third-highest distinct salary**

Using MAX and nested subqueries:

```
SELECT MAX(salary)
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
    WHERE salary < (
        SELECT MAX(salary)
        FROM employees
    )
);
```

Using DENSE_RANK:

```
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (
            ORDER BY salary DESC
        ) AS rnk
    FROM employees
)
SELECT DISTINCT salary
FROM ranked
WHERE rnk = 3;
```

### **Variation 3: Find the second employee when sorted by salary**

Now the interviewer wants the second row, not the second distinct salary.

```
SELECT *
FROM employees
ORDER BY salary DESC, emp_id ASC
LIMIT 1 OFFSET 1;
```

This uses ROW_NUMBER-style positioning, so duplicate salaries are not eliminated.

### **Variation 4: Find the second-highest salary in each department**

```
WITH ranked AS (
    SELECT
        name,
        department_id,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS rnk
    FROM employees
)
SELECT *
FROM ranked
WHERE rnk = 2;
```

### **Variation 5: What if only one distinct salary exists?**

Then a second-highest distinct salary does not exist.

Method 1 using MAX returns NULL.

Method 3 using DENSE_RANK returns no row.

An interviewer may ask you how you would handle this difference in the application.

# **4. How to respond when an interviewer says "Solve it another way"**

Don't panic or assume that your previous solution was wrong.

A good response would be:

"Sure. My first approach used MAX with a subquery. Another approach is to use DENSE_RANK, which also handles duplicate salaries correctly. I can additionally solve it using a correlated subquery or self-join by counting the number of distinct salaries greater than each salary."

This communicates three things:

- You know multiple approaches.
- You understand the underlying logic.
- You can adapt to new constraints.

# **5. The SQL Alternative-Solution Mastery Map**

You should prepare multiple approaches for these 15 frequently tested problems.

| Problem                             | Alternative approaches to master                            |
| ----------------------------------- | ----------------------------------------------------------- |
| Second-highest salary               | MAX, LIMIT, DENSE_RANK, correlated COUNT, SELF JOIN         |
| Highest salary per department       | GROUP BY, correlated subquery, RANK                         |
| Top 3 per department                | ROW_NUMBER, RANK, DENSE_RANK                                |
| Find duplicates                     | GROUP BY HAVING, ROW_NUMBER, SELF JOIN                      |
| Customers without orders            | NOT EXISTS, LEFT JOIN IS NULL, NOT IN with NULL precautions |
| Employees above average             | Subquery, CTE, window AVG                                   |
| Find managers                       | SELF JOIN, recursive CTE for hierarchy                      |
| Running total                       | SUM OVER, correlated aggregation, self-join                 |
| Previous transaction                | LAG, SELF JOIN                                              |
| Consecutive dates                   | LAG, gaps-and-islands                                       |
| First order per customer            | MIN + JOIN, ROW_NUMBER, correlated MIN                      |
| Find median                         | PERCENTILE_CONT, window ranking                             |
| Customers purchasing every category | Double NOT EXISTS, GROUP BY HAVING                          |
| Monthly growth                      | LAG, self-join                                              |
| Remove duplicates                   | ROW_NUMBER, DELETE using CTE, self-join                     |

# **6. New interview preparation technique: Constraint-based SQL solving**

A particularly effective exercise is to solve the same problem under different restrictions.

For example:

**Constraint-based practice**

Select a restriction to reveal what you should practise.

Solve without LIMIT

Solve without MAX()

Solve without window functions

Solve without subqueries

Solve with only one SELECT statement

Handle ties

Optimise for large data

# **7. Your new SQL practice requirement**

From now on, for every important SQL interview question, practise at least three approaches:

1. **Primary solution:** The simplest readable solution.
2. **Alternative solution:** A different SQL construct solving the same problem.
3. **Optimised or constraint-based solution:** Consider the data size, indexes and special restrictions.

For example:

| Question                 | First solution      | Second solution | Third solution          |
| ------------------------ | ------------------- | --------------- | ----------------------- |
| Second-highest salary    | MAX subquery        | DENSE_RANK      | Correlated COUNT        |
| Duplicate emails         | GROUP BY            | ROW_NUMBER      | SELF JOIN               |
| No orders                | NOT EXISTS          | LEFT JOIN       | NOT IN with NULL safety |
| Above department average | Correlated subquery | Window AVG      | CTE + JOIN              |
| Top 3 per group          | ROW_NUMBER          | RANK            | DENSE_RANK              |

**One important caution:** Not every problem can be solved efficiently or naturally using every SQL technique. The goal is to understand alternatives and explain why one is preferable, not to force a JOIN or a window function into every solution.

## **Next: Advanced SQL Interview Drill**

We'll now move to questions where I'll give you one problem, you attempt it, and then we'll evaluate multiple solutions, edge cases, query optimisation and follow-up interviewer challenges.

Your first challenge:

**Find the third-highest distinct salary from an employee table.**

Try to solve it in three ways without looking at the earlier solutions. Then we can evaluate your approaches like a real interviewer.


# **Advanced SQL Interview Drill — Set 1**

We'll now practise SQL the way interviewers test it: **one problem, multiple approaches, changing constraints, edge cases and optimisation.**

The objective is to become comfortable when an interviewer says, "Good. Now solve it another way."

# **Problem 1 — Find the third-highest distinct salary**

Frequently asked SDE interview question

Consider:

**employees**

| emp_id | name   | salary |
| ------ | ------ | ------ |
| 1      | Aman   | 60000  |
| 2      | Rahul  | 80000  |
| 3      | Priya  | 75000  |
| 4      | Neha   | 90000  |
| 5      | Karan  | 50000  |
| 6      | Simran | 85000  |
| 7      | Riya   | 90000  |
| 8      | Meera  | 95000  |

Distinct salaries in descending order:

```
95000  → 1st
90000  → 2nd
85000  → 3rd
80000  → 4th
75000  → 5th
60000  → 6th
50000  → 7th
```

Expected answer: 85,000

## **Method 1 — Nested MAX()**

```
SELECT MAX(salary) AS third_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
    WHERE salary < (
        SELECT MAX(salary)
        FROM employees
    )
);
```

**Execution logic:**

1. Inner query finds 95,000.
2. Second query finds 90,000.
3. Outer query finds 85,000.

This method uses repeated maximum-value calculations.

**Interviewer follow-up:** Can you generalise this for the Nth-highest salary?

You could use nested queries, but this approach becomes cumbersome as N increases.

## **Method 2 — DENSE_RANK()**

This is one of the cleanest general-purpose solutions.

```
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (
            ORDER BY salary DESC
        ) AS rnk
    FROM employees
)
SELECT DISTINCT salary
FROM ranked
WHERE rnk = 3;
```

Output:

```
85000
```

Why DENSE_RANK?

Because we want distinct salary positions, and duplicate salaries should receive the same rank.

## **Method 3 — ORDER BY + LIMIT + OFFSET**

```
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;
```

Explanation:

| Clause        | Purpose                              |
| ------------- | ------------------------------------ |
| DISTINCT      | Remove duplicate salaries            |
| ORDER BY DESC | Highest salary first                 |
| OFFSET 2      | Skip the first two distinct salaries |
| LIMIT 1       | Return the next salary               |

Dialect-specific

This syntax works in PostgreSQL and MySQL. SQL Server commonly uses OFFSET/FETCH instead.

## **Method 4 — Correlated subquery using COUNT()**

Now solve it without MAX(), LIMIT or window functions.

```
SELECT DISTINCT e1.salary
FROM employees e1
WHERE (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
) = 2;
```

The logic:

For each distinct salary, count how many distinct salaries are greater than it.

The third-highest salary has exactly two distinct salaries greater than it.

| Salary | Distinct higher salaries | Result |
| ------ | ------------------------ | ------ |
| 95000  | 0                        | No     |
| 90000  | 1                        | No     |
| 85000  | 2                        | Yes    |
| 80000  | 3                        | No     |

This is a great alternative when an interviewer prohibits window functions.

## **Method 5 — SELF JOIN**

```
SELECT e1.salary
FROM employees e1
JOIN employees e2
    ON e1.salary < e2.salary
GROUP BY e1.salary
HAVING COUNT(DISTINCT e2.salary) = 2;
```

Here:

- e1 represents the candidate salary.
- e2 represents salaries higher than the candidate.
- GROUP BY groups candidate salaries.
- HAVING checks whether exactly two distinct salaries are higher.

This is another solution without window functions or LIMIT.

## **Method 6 — Derived table**

```
SELECT MIN(salary) AS third_highest_salary
FROM (
    SELECT DISTINCT salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 3
) AS top_three;
```

The inner query finds the top three distinct salaries. The outer query finds the minimum among those three.

## **Compare the six solutions**

| Method | Technique        | Best use                         |
| ------ | ---------------- | -------------------------------- |
| 1      | Nested MAX       | Simple, small N                  |
| 2      | DENSE_RANK       | General ranking                  |
| 3      | LIMIT/OFFSET     | Simple ordered retrieval         |
| 4      | Correlated COUNT | Ranking without window functions |
| 5      | SELF JOIN        | Alternative relational approach  |
| 6      | Derived table    | Top-N extraction                 |

### **Interviewer follow-up questions**

**Q1. What if the interviewer asks for the third employee by salary, not the third distinct salary?**

Use ROW_NUMBER:

```
WITH ranked AS (
    SELECT
        *,
        ROW_NUMBER() OVER (
            ORDER BY salary DESC, emp_id
        ) AS rn
    FROM employees
)
SELECT *
FROM ranked
WHERE rn = 3;
```

With this data, the third employee is Simran, earning 85,000.

**Q2. What if the interviewer asks for all employees receiving the third-highest salary?**

Use DENSE_RANK:

```
WITH ranked AS (
    SELECT *,
        DENSE_RANK() OVER (
            ORDER BY salary DESC
        ) AS rnk
    FROM employees
)
SELECT *
FROM ranked
WHERE rnk = 3;
```

**Q3. What if only two distinct salaries exist?**

There is no third-highest distinct salary.

The MAX-based aggregate approach returns NULL, whereas ranking and LIMIT approaches return no result row.

**Q4. Which solution would you choose?**

I'd generally choose DENSE_RANK for a generalised Nth-highest distinct salary problem, or MAX with a subquery for the simple second-highest salary problem. The choice depends on the requirement and SQL dialect.

# **Problem 2 — Find the highest-paid employee in each department**

Very common interview question

Consider:

| emp_id | name   | department_id | salary |
| ------ | ------ | ------------- | ------ |
| 1      | Aman   | 101           | 60000  |
| 2      | Rahul  | 102           | 80000  |
| 3      | Priya  | 101           | 75000  |
| 4      | Neha   | 103           | 90000  |
| 5      | Karan  | 102           | 50000  |
| 6      | Simran | 101           | 85000  |
| 7      | Meera  | 101           | 85000  |

Expected result:

| name   | department_id | salary |
| ------ | ------------- | ------ |
| Simran | 101           | 85000  |
| Meera  | 101           | 85000  |
| Rahul  | 102           | 80000  |
| Neha   | 103           | 90000  |

Notice that two employees are tied in Engineering.

An interviewer might deliberately introduce this tie to see whether your solution handles it.

## **Method 1 — Correlated subquery with MAX()**

```
SELECT *
FROM employees e
WHERE salary = (
    SELECT MAX(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

This is concise and returns all employees tied for maximum salary.

## **Method 2 — Window function with DENSE_RANK()**

```
WITH ranked AS (
    SELECT
        *,
        DENSE_RANK() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS rnk
    FROM employees
)
SELECT *
FROM ranked
WHERE rnk = 1;
```

## **Method 3 — GROUP BY + JOIN**

First calculate the highest salary of each department.

Then JOIN the result back to the employee table.

```
SELECT
    e.emp_id,
    e.name,
    e.department_id,
    e.salary
FROM employees e
JOIN (
    SELECT
        department_id,
        MAX(salary) AS highest_salary
    FROM employees
    GROUP BY department_id
) d
    ON e.department_id = d.department_id
   AND e.salary = d.highest_salary;
```

This is a very important alternative to window functions.

## **Method 4 — NOT EXISTS**

Find employees for whom no employee in the same department earns more.

```
SELECT *
FROM employees e
WHERE NOT EXISTS (
    SELECT 1
    FROM employees e2
    WHERE e2.department_id = e.department_id
      AND e2.salary > e.salary
);
```

This is a logical approach to finding maximum records.

Read it in English:

"Return employees for whom there does not exist another employee in their department with a higher salary."

### **Interviewer follow-up: What if you want exactly one employee per department?**

Use ROW_NUMBER and a deterministic tie-breaker:

```
WITH ranked AS (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC, emp_id ASC
        ) AS rn
    FROM employees
)
SELECT *
FROM ranked
WHERE rn = 1;
```

This deliberately selects one employee when salaries are tied.

**Important distinction:**

- DENSE_RANK = return all tied highest earners.
- ROW_NUMBER = return exactly one selected highest earner.

# **Problem 3 — Find customers who have never placed an order**

Frequently asked JOIN question

Tables:

`customers(customer_id, customer_name)`

`orders(order_id, customer_id, amount)`

## **Method 1 — NOT EXISTS**

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

## **Method 2 — LEFT JOIN**

```
SELECT c.*
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

## **Method 3 — NOT IN**

```
SELECT *
FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id
    FROM orders
    WHERE customer_id IS NOT NULL
);
```

Why did we explicitly add `IS NOT NULL`?

Because if the NOT IN subquery contains a NULL, the result can become UNKNOWN for otherwise unmatched values.

This is a common interview trap.

### **How do you explain the differences?**

| Method     | Main idea                                      |
| ---------- | ---------------------------------------------- |
| NOT EXISTS | No matching record exists                      |
| LEFT JOIN  | Find unmatched rows                            |
| NOT IN     | Exclude values in a set, with NULL precautions |

I'd generally prefer NOT EXISTS for this existence-check problem because it communicates the intent clearly.

# **Problem 4 — Find employees earning above their department average**

This question is a good test of your ability to move between different SQL concepts.

## **Method 1 — Correlated subquery**

```
SELECT *
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

## **Method 2 — Window function**

```
WITH stats AS (
    SELECT
        *,
        AVG(salary) OVER (
            PARTITION BY department_id
        ) AS department_avg
    FROM employees
)
SELECT *
FROM stats
WHERE salary > department_avg;
```

## **Method 3 — GROUP BY + JOIN**

```
SELECT
    e.*
FROM employees e
JOIN (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) d
    ON e.department_id = d.department_id
WHERE e.salary > d.avg_salary;
```

### **Which method should you choose?**

| Method              | Main advantage                                            |
| ------------------- | --------------------------------------------------------- |
| Correlated subquery | Direct expression of the requirement                      |
| Window function     | Retains employee-level rows and adds statistics           |
| GROUP BY + JOIN     | Separates department aggregation from employee comparison |

All three can be good solutions. Performance depends on the optimiser and data distribution.

# **3. How interviewers change the problem**

The most useful practice is to take one problem and introduce restrictions.

Let's use the highest-paid employee problem.

**Interviewer variation drill**

1\. Solve without window functions.

2\. Solve without MAX().

3\. Return exactly one employee per department.

4\. Return all employees tied for the highest salary.

5\. Return the top three distinct salary levels per department.

6\. Return departments even when they have no employees.

# **4. New interview challenge — solve without assistance**

These are the next three problems. Try to write at least two different solutions for each.

**Your SQL interview practice**

**Challenge 1**

Medium

**Find the third-highest distinct salary in each department.**

Suggested concept: PARTITION BY + DENSE_RANK / correlated COUNT

Reveal approach

**Challenge 2**

Hard

**Find customers who placed orders in every month of a given quarter.**

Suggested concept: GROUP BY + HAVING / relational division

Reveal approach

**Challenge 3**

Hard

**Find the longest consecutive login streak for each user.**

Suggested concept: Gaps and Islands + window functions

Reveal approach

# **5. The rule for multiple-solution interviews**

For every question, train yourself to answer these five things:

1. What is the simplest correct SQL solution?
2. Can I solve it using a different SQL construct?
3. What happens when duplicate values exist?
4. What happens when NULLs or ties exist?
5. Which approach would I choose for a large dataset, and how would I verify its performance?

**This is the level of preparation that turns SQL query memorisation into transferable interview problem-solving ability.**

# **Advanced SQL Interview Drill — Set 2**

Let's solve the three challenges from the previous session properly, with **multiple approaches, edge cases and interviewer follow-ups**.

All examples use PostgreSQL.

# **Problem 1 — Third-highest distinct salary in every department**



Medium

Multiple solutions



**Schema:** `employees(emp_id, name, department_id, salary)`

Sample data:

| emp_id | name   | department_id | salary |
| ------ | ------ | ------------- | ------ |
| 1      | Aman   | 101           | 60000  |
| 2      | Rahul  | 101           | 90000  |
| 3      | Priya  | 101           | 85000  |
| 4      | Neha   | 101           | 75000  |
| 5      | Karan  | 101           | 75000  |
| 6      | Simran | 102           | 95000  |
| 7      | Riya   | 102           | 80000  |
| 8      | Meera  | 102           | 70000  |

Expected output:

| name  | department_id | salary |
| ----- | ------------- | ------ |
| Neha  | 101           | 75000  |
| Karan | 101           | 75000  |
| Meera | 102           | 70000  |

Notice that department 101 has two employees tied at the third-highest distinct salary.

## **Method 1 — DENSE_RANK()**

```
WITH ranked AS (
    SELECT
        emp_id,
        name,
        department_id,
        salary,

        DENSE_RANK() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS salary_rank

    FROM employees
    WHERE salary IS NOT NULL
)
SELECT *
FROM ranked
WHERE salary_rank = 3;
```

**Execution logic:**

Department 101:

| Salary | Rank |
| ------ | ---- |
| 90000  | 1    |
| 85000  | 2    |
| 75000  | 3    |
| 75000  | 3    |
| 60000  | 4    |

Department 102:

| Salary | Rank |
| ------ | ---- |
| 95000  | 1    |
| 80000  | 2    |
| 70000  | 3    |

Why DENSE_RANK?

Because equal salaries receive equal ranks, and there are no gaps between distinct salary ranks.

## **Method 2 — Correlated subquery without window functions**

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
    WHERE e2.department_id = e1.department_id
      AND e2.salary > e1.salary
) = 2;
```

For every employee:

1. Look at other employees in the same department.
2. Count distinct salaries greater than the current salary.
3. Keep the employee when exactly two distinct salaries are greater.

This returns both Neha and Karan for department 101.

## **Interviewer variations**

| Follow-up                                   | What changes?                 |
| ------------------------------------------- | ----------------------------- |
| Third-highest employee, not distinct salary | ROW_NUMBER                    |
| Third distinct salary                       | DENSE_RANK                    |
| Exactly one employee per department         | ROW_NUMBER + tie-breaker      |
| Include everyone tied at third place        | DENSE_RANK                    |
| No window functions                         | Correlated COUNT              |
| No subqueries                               | Self-join + GROUP BY + HAVING |

**Important edge case:** If `department_id` can be NULL and you want all NULL-department employees grouped together, use `IS NOT DISTINCT FROM` rather than `=` in the correlated subquery.

# **Problem 2 — Customers who placed orders in every month of a quarter**



Hard

Relational division



**Schema:** `customers(customer_id, customer_name)` and `orders(order_id, customer_id, order_date, amount)`.

Assume the quarter is January–March 2026.

The requirement is:

A customer must have at least one order in January, February and March.

Having three orders in January does not satisfy the requirement.

## **Method 1 — GROUP BY + COUNT DISTINCT**

```
SELECT
    customer_id
FROM orders
WHERE order_date >= DATE '2026-01-01'
  AND order_date < DATE '2026-04-01'
GROUP BY customer_id
HAVING COUNT(
    DISTINCT DATE_TRUNC('month', order_date)
) = 3;
```

Why does this work?

- Filter orders to the quarter.
- Group by customer.
- Convert order dates into month-level values.
- Count distinct months.
- Keep customers appearing in all three months.

This is the simplest approach for a fixed three-month quarter.

To display customer names:

```
SELECT
    c.customer_id,
    c.customer_name
FROM customers c
JOIN (
    SELECT customer_id
    FROM orders
    WHERE order_date >= DATE '2026-01-01'
      AND order_date < DATE '2026-04-01'
    GROUP BY customer_id
    HAVING COUNT(
        DISTINCT DATE_TRUNC('month', order_date)
    ) = 3
) x
ON c.customer_id = x.customer_id;
```

## **Method 2 — Double NOT EXISTS**

This is a more general relational division solution.

```
WITH quarter_months AS (
    SELECT GENERATE_SERIES(
        DATE '2026-01-01',
        DATE '2026-03-01',
        INTERVAL '1 month'
    )::DATE AS month_start
)

SELECT
    c.customer_id,
    c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM quarter_months m
    WHERE NOT EXISTS (
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.customer_id
          AND o.order_date >= m.month_start
          AND o.order_date < m.month_start + INTERVAL '1 month'
    )
);
```

Read the query in English:

Find customers for whom there does not exist a month in the quarter for which there does not exist an order.

In simpler terms:

**There is no month in which the customer failed to place an order.**

This technique is extremely useful in advanced interviews.

## **Interviewer variations**

- Customers who purchased in every month of a year.
- Customers who purchased every product in a category.
- Customers who attended every required event.
- Employees who completed every mandatory training module.

These are all variations of relational division.

**Edge case:** If only completed orders should count, add `AND o.status = 'Completed'` to the order conditions in both methods.

# **Problem 3 — Longest consecutive login streak per user**



Hard

Gaps and Islands



Schema:

`logins(user_id, login_date)`

Sample data:

| user_id | login_date |
| ------- | ---------- |
| 1       | Jan 01     |
| 1       | Jan 02     |
| 1       | Jan 03     |
| 1       | Jan 05     |
| 1       | Jan 06     |
| 2       | Jan 01     |
| 2       | Jan 03     |
| 2       | Jan 04     |
| 2       | Jan 05     |
| 2       | Jan 06     |

Expected result:

| user_id | start_date | end_date | streak_length |
| ------- | ---------- | -------- | ------------- |
| 1       | Jan 01     | Jan 03   | 3             |
| 2       | Jan 03     | Jan 06   | 4             |

## **Method 1 — Gaps and Islands using ROW_NUMBER()**

This is a classic interview solution.

```
WITH distinct_logins AS (
    SELECT DISTINCT
        user_id,
        login_date::DATE AS login_date
    FROM logins
),

numbered AS (
    SELECT
        user_id,
        login_date,

        login_date -
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY login_date
        )::INT AS island_key

    FROM distinct_logins
),

streaks AS (
    SELECT
        user_id,
        island_key,
        MIN(login_date) AS start_date,
        MAX(login_date) AS end_date,
        COUNT(*) AS streak_length

    FROM numbered
    GROUP BY user_id, island_key
),

ranked AS (
    SELECT
        *,
        DENSE_RANK() OVER (
            PARTITION BY user_id
            ORDER BY streak_length DESC
        ) AS streak_rank
    FROM streaks
)

SELECT
    user_id,
    start_date,
    end_date,
    streak_length
FROM ranked
WHERE streak_rank = 1
ORDER BY user_id;
```

### **Understand the magic behind the query**

For user 1:

| Login date | Row number | Date − row number |
| ---------- | ---------- | ----------------- |
| Jan 01     | 1          | Dec 31            |
| Jan 02     | 2          | Dec 31            |
| Jan 03     | 3          | Dec 31            |
| Jan 05     | 4          | Jan 01            |
| Jan 06     | 5          | Jan 01            |

Notice:

- Consecutive dates produce the same calculated key.
- A missing date changes the key.

That is how SQL identifies separate consecutive groups.

**Gaps and Islands visualisation**

**01**

**02**

**03**

**05**

**06**

Island 1: 3 days

Island 2: 2 days

The January 4 gap splits the login sequence into two separate islands.

## **Method 2 — Using LAG()**

Some interviewers may specifically ask for a solution using LAG.

```
WITH distinct_logins AS (
    SELECT DISTINCT
        user_id,
        login_date::DATE AS login_date
    FROM logins
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
            WHEN previous_date = login_date - 1
                THEN 0
            ELSE 1
        END AS new_island

    FROM previous_dates
),

grouped AS (
    SELECT
        user_id,
        login_date,

        SUM(new_island) OVER (
            PARTITION BY user_id
            ORDER BY login_date
            ROWS BETWEEN UNBOUNDED PRECEDING
                     AND CURRENT ROW
        ) AS island_id

    FROM marked
),

streaks AS (
    SELECT
        user_id,
        MIN(login_date) AS start_date,
        MAX(login_date) AS end_date,
        COUNT(*) AS streak_length
    FROM grouped
    GROUP BY user_id, island_id
),

ranked AS (
    SELECT *,
        DENSE_RANK() OVER (
            PARTITION BY user_id
            ORDER BY streak_length DESC
        ) AS rnk
    FROM streaks
)

SELECT
    user_id,
    start_date,
    end_date,
    streak_length
FROM ranked
WHERE rnk = 1;
```

### **Which solution is better?**

| ROW_NUMBER approach                       | LAG approach                                     |
| ----------------------------------------- | ------------------------------------------------ |
| Compact gaps-and-islands technique        | Explicitly detects sequence boundaries           |
| Great for consecutive numeric/date values | Great when comparing adjacent rows               |
| Uses date minus row number                | Uses previous date + cumulative SUM              |
| Useful for many sequence problems         | Useful when boundary conditions are more complex |

Both are valid.

**Important edge cases:**

- Deduplicate login dates before calculating streaks.
- Cast timestamps to dates if the requirement is consecutive calendar days.
- Use a calendar-day comparison, not a simple row count, when there may be missing dates.
- Use DENSE_RANK to return all tied longest streaks; use ROW_NUMBER to return only one.
- Consecutive business days require a business calendar rather than simply subtracting one calendar day.

# **4. Interviewer Challenge — New Restrictions**

Now imagine you have already submitted the three solutions.

The interviewer asks you the following:

| Question                                  | Constraint                 |
| ----------------------------------------- | -------------------------- |
| Third-highest salary per department       | No window functions        |
| Customers with orders in all three months | No GROUP BY                |
| Longest consecutive login streak          | No ROW_NUMBER              |
| Highest-paid employee per department      | No MAX                     |
| Second-highest distinct salary            | No LIMIT and no DENSE_RANK |

These are not necessarily easier. They test whether you understand the relational logic underneath the SQL syntax.

# **5. Independent Interview Drill — Set 3**

Now attempt these five problems.

**Set 3 — Your SQL answer sheet**

5 questions

**Problem 1**

Medium

Find the second-highest distinct salary in each department.

Hint: Try correlated COUNT(DISTINCT) and GROUP BY + JOIN.

Show hint

**Problem 2**

Medium

Find customers who made at least three completed orders in the same calendar month.

Hint: Think DATE_TRUNC, GROUP BY and HAVING.

Show hint

**Problem 3**

Medium

Find the employee with the longest tenure in every department. Return all ties.

Hint: Think hire_date, MIN and JOIN, or RANK.

Show hint

**Problem 4**

Hard

Find users who logged in on three consecutive days but not on the fourth day.

Hint: Use date sequences, LAG/LEAD or a gaps-and-islands approach.

Show hint

**Problem 5**

Hard

Find the percentage of customers who made a second purchase within 30 days of their first purchase.

Hint: Use MIN, JOIN or LAG, conditional counting and safe division.

Show hint

# **6. What a strong SQL interview answer looks like**

For any of these problems, your answer should contain four parts:

**1. Logic:** Explain the approach in plain English.

**2. SQL:** Write the actual query with readable aliases and formatting.

**3. Edge cases:** Discuss duplicates, NULL values, ties and empty groups where relevant.

**4. Alternative:** Offer another solution when the interviewer asks for one, and explain the trade-offs.

For example:

> "I would solve this using DENSE_RANK partitioned by department because the question asks for the third distinct salary level in each department. It also preserves employees with tied salaries. Alternatively, I can use a correlated COUNT of distinct higher salaries if window functions are prohibited."

That is the type of reasoning you should practise.

**Your next milestone:** Solve Set 3 independently, with at least two SQL approaches for Problems 1, 3 and 5.

