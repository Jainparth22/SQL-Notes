# **SQL Final Placement Preparation — Set 7**

## **Timed SQL OA Simulation · Level 1: Easy**

Level 1

PostgreSQL

10 questions

We're moving from learning SQL to testing whether you can solve problems independently.

**Test conditions**

| Parameter          | Target                 |
| ------------------ | ---------------------- |
| Questions          | 10                     |
| Time limit         | 35 minutes             |
| Total marks        | 20                     |
| Marks per question | 2                      |
| Difficulty         | Easy                   |
| Solutions          | Not provided initially |

Try solving without notes or looking at previous solutions. Use PostgreSQL syntax.

## **Part A — Database schema**

Use the following tables throughout this assessment.

**1. departments**

| Column          | Type    |
| --------------- | ------- |
| department_id   | INT, PK |
| department_name | VARCHAR |

**2. employees**

| Column        | Type    |
| ------------- | ------- |
| emp_id        | INT, PK |
| emp_name      | VARCHAR |
| department_id | INT, FK |
| salary        | NUMERIC |
| city          | VARCHAR |
| hire_date     | DATE    |

**3. customers**

| Column        | Type    |
| ------------- | ------- |
| customer_id   | INT, PK |
| customer_name | VARCHAR |
| city          | VARCHAR |

**4. orders**

| Column       | Type    |
| ------------ | ------- |
| order_id     | INT, PK |
| customer_id  | INT, FK |
| order_date   | DATE    |
| status       | VARCHAR |
| total_amount | NUMERIC |

**5. products**

| Column       | Type    |
| ------------ | ------- |
| product_id   | INT, PK |
| product_name | VARCHAR |
| category     | VARCHAR |
| price        | NUMERIC |

**6. order_items**

| Column     | Type    |
| ---------- | ------- |
| item_id    | INT, PK |
| order_id   | INT, FK |
| product_id | INT, FK |
| quantity   | INT     |
| unit_price | NUMERIC |

Assumptions: Departments may have zero employees, customers may have zero orders, and multiple customers or employees may share a city. Use `order_items` for product-level sales calculations and `orders.total_amount` for order-level totals.

## **Part B — Easy questions**

### **Q1. Employees earning above ₹50,000**

Return:

- Employee ID
- Employee name
- Salary

Sort by salary from highest to lowest.



2 marks

&#x20;Expected concepts: SELECT, WHERE, ORDER BY



### **Q2. Find unique employee cities**

Return all distinct, non-NULL cities in which employees live.

Sort alphabetically.



2 marks

&#x20;Expected concepts: DISTINCT, NULL



### **Q3. Top five highest-paid employees**

Return the five employees with the highest salaries.

Show employee ID, name and salary.

For equal salaries, use employee ID as the tie-breaker.



2 marks

&#x20;Expected concepts: ORDER BY, LIMIT



### **Q4. Employee count per department**

Return:

- Department ID
- Department name
- Number of employees

Include departments that currently have zero employees.



2 marks

&#x20;Expected concepts: LEFT JOIN, GROUP BY, COUNT



### **Q5. Departments with average salary above ₹60,000**

Return the department name and average employee salary.

Only include departments whose average salary is greater than ₹60,000.

Round the average to two decimal places.



2 marks

&#x20;Expected concepts: AVG, GROUP BY, HAVING



### **Q6. Customers who have never placed an order**

Return customer ID, customer name and city.



2 marks

&#x20;Expected concepts: NOT EXISTS or LEFT JOIN



### **Q7. Customers who placed at least three orders**

Return customer ID, customer name and total number of orders.

Sort by order count in descending order.



2 marks

&#x20;Expected concepts: JOIN, GROUP BY, HAVING



### **Q8. Completed order revenue per customer**

Calculate the total value of completed orders for every customer.

Include customers who have no completed orders, displaying zero instead of NULL.

Return customer ID, customer name and completed revenue.



2 marks

&#x20;Expected concepts: LEFT JOIN, conditional filtering, COALESCE



### **Q9. Products that have never been ordered**

Return the product ID, name and category of products that do not appear in `order_items`.



2 marks

&#x20;Expected concepts: NOT EXISTS, anti-join



### **Q10. Find the second-highest distinct salary**

Return the second-highest distinct salary across all employees.

Do not return the second employee in sorted order. Two or more employees can have the same salary.



2 marks

&#x20;Expected concepts: MAX, subquery, DISTINCT, ranking



## **Part C — Submission and evaluation**

**Your test tracker**

Questions attempted

**0/10**

Q1

Q2

Q3

Q4

Q5

Q6

Q7

Q8

Q9

Q10

**Time taken (minutes)**

20

25

35

45

60

**Difficulties or doubts**

Submit progress

## **How I'll evaluate your answers**

For each query, I'll assess:

1. **Correctness:** Does it satisfy the actual requirement?
2. **Edge cases:** Does it handle ties, NULLs, empty tables and duplicate rows?
3. **Alternative approach:** Can you solve it another way?
4. **Efficiency:** Are there unnecessary joins, scans or aggregations?
5. **Interview explanation:** Can you explain why your query works?

**Important:** Don't just read solutions. Write the SQL yourself first. Even if you cannot solve a question, submit your attempt so we can identify exactly which concept needs strengthening.

After Level 1, we'll proceed to **Level 2: 10 Medium questions**, followed by **Level 3: 5 Hard questions**.

# **SQL Final Placement Preparation — Set 7**

## **Level 2: Medium SQL OA Simulation**

Level 2

Medium

10 questions

Now we move beyond basic SQL. These questions test the patterns frequently used in SQL coding rounds for software engineering, data engineering and analyst roles.

**Test conditions**

| Parameter          | Target     |
| ------------------ | ---------- |
| Questions          | 10         |
| Time limit         | 50 minutes |
| Total marks        | 30         |
| Marks per question | 3          |
| Difficulty         | Medium     |
| Database           | PostgreSQL |

Use the same schema from Level 1: `employees`, `departments`, `customers`, `orders`, `products`, and `order_items`.

## **Part A — Questions**

### **Q1. Employees earning more than their department's average salary**

Return the employee ID, name, department ID, salary and department average salary.

Include only employees whose salary is strictly greater than their department average.

**Expected concepts:** Correlated subqueries, window functions, CTEs.

3 marks

### **Q2. Second-highest distinct salary in every department**

Return:

- Department ID
- Department name
- Second-highest distinct salary

Exclude departments that have fewer than two distinct salary values.

**Important:** If three employees earn the highest salary, the next lower distinct salary is still the second-highest salary.

**Expected concepts:** `DENSE_RANK()`, correlated subqueries, `MAX()`.

3 marks

### **Q3. Highest-paid employee in each department**

Return all employees who earn the highest salary in their respective departments.

If two or more employees share the highest salary, return all of them.

**Expected concepts:** `MAX()` + JOIN, `RANK()`, `NOT EXISTS`.

3 marks

### **Q4. Top two products by revenue in each category**

Calculate product revenue using:

\\[ \text{Revenue} = \text{quantity} \times \text{unit price} \\]

Consider only orders with status `'Completed'`.

Return the two highest-revenue products in every category, including ties at rank two.

Return product name, category and total revenue.

**Expected concepts:** Multiple JOINs, aggregation, `DENSE_RANK()`.

3 marks

### **Q5. Monthly revenue and month-over-month growth**

Calculate completed order revenue for every month.

Return:

- Month
- Current month revenue
- Previous recorded month's revenue
- Revenue difference
- Percentage growth

Use `orders.total_amount`.

Assume only months with at least one completed order need to appear. The previous month means the previous month in the resulting activity data, not necessarily the previous calendar month.

**Expected concepts:** `DATE_TRUNC()`, `LAG()`, aggregation, `NULLIF()`.

3 marks

### **Q6. Customers who placed exactly two completed orders**

Return customer ID, customer name and the number of completed orders.

Exclude customers with one, zero or more than two completed orders.

**Expected concepts:** `GROUP BY`, `HAVING`, correlated subqueries, window functions.

3 marks

### **Q7. Latest order of every customer**

Return each customer's latest order.

Show customer ID, customer name, order ID, order date and order amount.

Include customers with no orders, displaying NULL for their order details.

When multiple orders have the same order date, select the order with the highest order ID.

**Expected concepts:** `ROW_NUMBER()`, `LEFT JOIN`, CTEs, `LATERAL`.

3 marks

### **Q8. Products never included in a completed order**

Return product ID, product name and category for products that have never appeared in any completed order.

A product appearing exclusively in pending or failed orders must still qualify.

**Expected concepts:** `NOT EXISTS`, anti-joins, condition placement.

3 marks

### **Q9. Customers whose completed spending exceeds the average customer spending**

Calculate each customer's total completed order value.

Find customers whose total completed spending is greater than the average total completed spending across all customers.

Include customers with no completed orders in the average calculation as zero-spending customers.

Return customer ID, customer name and completed spending.

**Expected concepts:** CTEs, aggregation, scalar subqueries, `COALESCE()`.

3 marks

### **Q10. Find dates when completed revenue exceeded the previous recorded sales date**

For each calendar date, calculate total completed order revenue.

Return dates where the revenue was greater than the revenue on the immediately previous date with completed sales.

For the first recorded date, no comparison is possible.

Return current date, current revenue, previous recorded date and previous revenue.

**Expected concepts:** `DATE_TRUNC()`, `LAG()`, `GROUP BY`, CTEs.

3 marks

## **Part B — What each question tests**

| Question | Core SQL pattern                        | Interview difficulty |
| -------- | --------------------------------------- | -------------------- |
| Q1       | Above-average comparison                | Medium               |
| Q2       | Second-highest distinct value per group | Medium               |
| Q3       | Highest record including ties           | Medium               |
| Q4       | Top-N-per-group                         | Medium               |
| Q5       | Month-over-month analytics              | Medium               |
| Q6       | Exact-count filtering                   | Medium               |
| Q7       | Latest record per entity                | Medium               |
| Q8       | Anti-join with conditions               | Medium               |
| Q9       | Aggregate compared with aggregate       | Medium–Hard          |
| Q10      | Time-series comparison                  | Medium               |

## **Part C — Your assessment tracker**

**Level 2 progress**

# **0/10**

Q1

Q2

Q3

Q4

Q5

Q6

Q7

Q8

Q9

Q10

**Time taken**

30 min

40 min

50 min

60 min

75 min

**Question you found most difficult**

Question 4

**Your observations**

Submit progress

## **Part D — Important interview challenge**

After solving a question, try to provide a second solution.

For example:

| Problem                       | First approach        | Alternative approach     |
| ----------------------------- | --------------------- | ------------------------ |
| Above department average      | Correlated subquery   | Window `AVG()`           |
| Second-highest salary         | `DENSE_RANK()`        | Nested `MAX()`           |
| Highest salary per department | `MAX()` + JOIN        | `RANK()`                 |
| Exactly two orders            | `GROUP BY` + `HAVING` | Correlated `COUNT()`     |
| Latest order                  | `ROW_NUMBER()`        | PostgreSQL `DISTINCT ON` |
| Products never ordered        | `NOT EXISTS`          | LEFT JOIN + NULL         |

This is particularly important because an interviewer may ask you to remove a particular SQL construct or produce a different solution.

## **What's next?**

**Level 3 — Hard SQL OA Simulation:**

The final five questions will test complex combinations of:

1. Gaps and islands with consecutive activity.
2. Customer retention and cohort analysis.
3. Sessionisation using event timestamps.
4. Relational division and universal conditions.
5. Complex multi-table analytics and query debugging.

After Level 3, we'll do a final mock interview covering SQL theory, query writing, execution order, indexing, transactions and optimization.

**My recommendation:** Attempt Level 1 and Level 2 yourself before looking at solutions. We need to measure actual independent problem-solving ability, not just familiarity with the queries.

# **SQL Final Placement Preparation — Set 7**

## **Level 3: Hard SQL OA Simulation**

Hard

PostgreSQL

5 questions

This is the final and most challenging part of the SQL OA simulation.

These questions combine multiple concepts. You may need two or more CTEs, window functions, correlated subqueries, aggregations and joins to solve a single problem.

**Test conditions**

| Parameter          | Target                 |
| ------------------ | ---------------------- |
| Questions          | 5                      |
| Time limit         | 60 minutes             |
| Total marks        | 25                     |
| Marks per question | 5                      |
| Difficulty         | Hard                   |
| Solutions          | Not provided initially |

Continue using the schema from Levels 1 and 2. We'll introduce two additional tables for this assessment.

## **Additional tables**

**user_logins**

| Column     | Type |
| ---------- | ---- |
| user_id    | INT  |
| login_date | DATE |

**events**

| Column     | Type      |
| ---------- | --------- |
| event_id   | INT, PK   |
| user_id    | INT       |
| event_name | VARCHAR   |
| event_time | TIMESTAMP |

## **Q1. Find the longest consecutive login streak for every user**

**Problem:** Find the longest consecutive login streak for every user.

Return:

- User ID
- Streak start date
- Streak end date
- Number of consecutive days

Requirements:

1. Consecutive means consecutive calendar dates.
2. Duplicate login records on the same date count as one login.
3. If a user has multiple streaks of the same maximum length, return all of them.
4. Users without any login records are not included.

**Example input:**

| user_id | login_date |
| ------- | ---------- |
| 101     | Nov 1      |
| 101     | Nov 2      |
| 101     | Nov 3      |
| 101     | Nov 5      |
| 101     | Nov 6      |
| 101     | Nov 7      |
| 102     | Nov 1      |
| 102     | Nov 2      |
| 102     | Nov 4      |

Expected result:

| user_id | streak_start | streak_end | days |
| ------- | ------------ | ---------- | ---- |
| 101     | Nov 1        | Nov 3      | 3    |
| 101     | Nov 5        | Nov 7      | 3    |
| 102     | Nov 1        | Nov 2      | 2    |

**Constraints:** Try solving using `ROW_NUMBER()` and gaps-and-islands grouping. You may use another CTE to determine the maximum streak length.

5 marks

## **Q2. Calculate monthly customer cohort retention**

**Problem:** Calculate customer retention for each acquisition cohort.

A customer belongs to a cohort based on the month of their first completed order.

For every cohort, calculate how many customers made at least one completed purchase in:

- Their acquisition month (Month 0).
- The following month (Month 1).
- Two months after acquisition (Month 2).
- Three months after acquisition (Month 3).

Return:

| cohort_month | month_number | cohort_size | retained_customers | retention_rate |
| ------------ | ------------ | ----------- | ------------------ | -------------- |
| Jan 2026     | 0            | 100         | 100                | 100.00         |
| Jan 2026     | 1            | 100         | 70                 | 70.00          |
| Jan 2026     | 2            | 100         | 55                 | 55.00          |
| Jan 2026     | 3            | 100         | 40                 | 40.00          |

Requirements:

1. Only `Completed` orders count.
2. A customer can place multiple orders in one month but counts only once for that month.
3. Acquisition month is the month of the customer's first completed order.
4. Only display months for which the full cohort observation period is available, or clearly identify incomplete periods.
5. Calculate retention rate as a percentage of the original cohort size.

**Constraints:** Use `DATE_TRUNC()`, CTEs, `COUNT(DISTINCT ...)` or equivalent logic, and month-offset calculation.

**Extra challenge:** Produce the result as a pivoted table with Month 0, Month 1, Month 2 and Month 3 as separate columns.

5 marks

## **Q3. Sessionisation from user event logs**

**Problem:** Users generate events while using an application.

A session ends when the difference between two consecutive events is greater than 30 minutes.

Find each user's sessions.

Return:

- User ID
- Session number
- Session start timestamp
- Session end timestamp
- Total events
- Session duration in minutes

**Example input:**

| user_id | event_time |
| ------- | ---------- |
| 101     | 10:00      |
| 101     | 10:10      |
| 101     | 10:25      |
| 101     | 11:10      |
| 101     | 11:20      |
| 101     | 13:00      |

Expected result:

| user_id | session_number | start | end   | events | duration |
| ------- | -------------- | ----- | ----- | ------ | -------- |
| 101     | 1              | 10:00 | 10:25 | 3      | 25       |
| 101     | 2              | 11:10 | 11:20 | 2      | 10       |
| 101     | 3              | 13:00 | 13:00 | 1      | 0        |

Requirements:

1. Every user's first event starts a new session.
2. A gap strictly greater than 30 minutes starts a new session.
3. Exactly 30 minutes belongs to the same session.
4. Events must be ordered deterministically using `event_time` and `event_id`.
5. Session numbers restart for every user.

**Constraints:** Use `LAG()` and a cumulative window `SUM()`.

**Extra challenge:** Find each user's longest session and return all sessions tied for the maximum duration.

5 marks

## **Q4. Find customers who purchased every product in a category**

**Problem:** Find customers who purchased every product belonging to the category `'Electronics'`.

Assume the `customers`, `orders`, `products` and `order_items` tables.

Requirements:

1. Only completed orders count.
2. Each product needs to have been purchased at least once.
3. Multiple purchases of the same product count only once.
4. Customers who did not purchase anything must not qualify when the category contains products.
5. Define the empty-category behaviour explicitly.

**Example:**

Products:

| product_id | product_name | category    |
| ---------- | ------------ | ----------- |
| 1          | Laptop       | Electronics |
| 2          | Mouse        | Electronics |
| 3          | Keyboard     | Electronics |
| 4          | Chair        | Furniture   |

Purchases:

| customer_id | product_id |
| ----------- | ---------- |
| 101         | 1          |
| 101         | 2          |
| 101         | 3          |
| 102         | 1          |
| 102         | 2          |
| 103         | 1          |
| 103         | 3          |

Expected result:

| customer_id |
| ----------- |
| 101         |

**Constraints:** Solve this in two ways:

- Double `NOT EXISTS`.
- `GROUP BY` + `HAVING COUNT(DISTINCT ...)`.

**Extra challenge:** Find customers who have purchased at least 80% of all products in Electronics.

5 marks

## **Q5. Complex multi-table revenue analytics**

**Problem:** Find the top two customers by completed purchase revenue in each city.

Revenue must be calculated from `order_items`:

\\[ \text{Revenue} = \sum(\text{quantity} \times \text{unit price}) \\]

Requirements:

1. Only completed orders count.
2. Calculate revenue for each customer.
3. Customers with no completed purchases are excluded.
4. Rank customers separately within each city.
5. Return the top two customers per city.
6. Include all customers tied at rank two.

Return:

| city   | customer_id | customer_name | total_revenue | city_rank |
| ------ | ----------- | ------------- | ------------- | --------- |
| Bhopal | 101         | Customer A    | 50000         | 1         |
| Bhopal | 105         | Customer B    | 42000         | 2         |
| Indore | 108         | Customer C    | 39000         | 1         |
| Indore | 109         | Customer D    | 35000         | 2         |

**Constraints:** Your solution should:

1. Join the required tables correctly.
2. Calculate revenue at customer level before ranking.
3. Use `DENSE_RANK()` or `RANK()` appropriately.
4. Avoid duplicate multiplication.
5. Include a deterministic ordering in your final output.

**Extra challenge:** Explain why this query can be incorrect:

```
SELECT
    c.city,
    c.customer_id,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    RANK() OVER (
        PARTITION BY c.city
        ORDER BY SUM(oi.quantity * oi.unit_price) DESC
    ) AS city_rank
FROM customers c
JOIN orders o
    ON o.customer_id = c.customer_id
JOIN order_items oi
    ON oi.order_id = o.order_id
WHERE o.status = 'Completed'
GROUP BY c.city, c.customer_id;
```

Is it actually incorrect? Under what data conditions would it produce the wrong result, and what changes would make it more robust?

5 marks

## **Part B — Hard-level assessment tracker**

**Level 3 progress**

# **0/5**

Q1 — Longest login streak

Q2 — Cohort retention

Q3 — Sessionisation

Q4 — Relational division

Q5 — Top customers by city revenue

**Time taken**

30 min

45 min

60 min

90 min

**How did you solve these?**

Independently, without notes

With notes or previous examples

Needed hints

Could not solve most questions

**Where did you struggle?**

Submit progress

## **How we will judge your final SQL readiness**

| Score      | Assessment                               |
| ---------- | ---------------------------------------- |
| 0–10 / 25  | Advanced patterns need more practice     |
| 11–15 / 25 | Developing hard-problem ability          |
| 16–20 / 25 | Strong SQL OA preparation                |
| 21–23 / 25 | Interview-ready problem-solving          |
| 24–25 / 25 | Excellent performance on this assessment |

These scores are targets for this practice set, not a guarantee of performance in an actual company OA.

## **What remains after Set 7?**

You will have finished the three-level SQL coding simulation. The final stage is **SQL Interview Mastery and Verification**:

1. SQL interview theory — execution order, joins, keys, indexes, NULL semantics and transactions.
2. Solve common questions in multiple ways.
3. Query optimization — `EXPLAIN ANALYZE`, indexing, execution plans and slow-query debugging.
4. Practical PostgreSQL — JSONB, UPSERT, views, triggers and stored procedures.
5. Final mock interview — a mixture of SQL coding, DBMS theory and real backend scenarios.

**One rule for this final stage:** Don't treat reading a correct answer as solving a problem. Write your query, test edge cases, explain the logic, and then compare it with alternative approaches.

