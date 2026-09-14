# **Lesson 5 — SQL CTEs (Common Table Expressions)**



High priority

Intermediate → Advanced



CTEs are an important SQL technique for writing readable, structured queries. They are particularly useful in complex interview and OA problems that require multiple steps of data processing.

By the end of this lesson, you should be able to solve:

- Find the department with the highest average salary.
- Find the second-highest department salary.
- Break complex queries into logical steps.
- Use multiple CTEs in one query.
- Build employee hierarchies using recursive CTEs.
- Perform hierarchical and multi-level data analysis.

We'll continue using PostgreSQL-compatible SQL.

# **1. What is a CTE?**

CTE stands for **Common Table Expression**.

It is a temporary named result set that you define using the `WITH` clause and use within a single SQL statement.

Think of it as creating a named intermediate result before writing the main query.

### **Without CTE**

Suppose the interviewer asks:

**Find employees earning more than their department's average salary.**

Using a correlated subquery:

```
SELECT
    e.name,
    e.department_id,
    e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

This works, but complex nested queries can become difficult to understand.

### **With CTE**

```
WITH department_average AS (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)

SELECT
    e.name,
    e.department_id,
    e.salary,
    d.avg_salary
FROM employees e
JOIN department_average d
    ON e.department_id = d.department_id
WHERE e.salary > d.avg_salary;
```

The CTE first calculates department-wise average salaries.

The main query then compares each employee's salary against the corresponding average.

**Understand the execution conceptually**

Step 1

**CTE: department_average**

| department_id | avg_salary |
| ------------- | ---------- |
| 101           | 73333      |
| 102           | 65000      |
| 103           | 90000      |
| 104           | 70000      |

Step 2

**JOIN employees with department_average**

Each employee gets their department's average salary.

Step 3

**Apply WHERE condition**

Keep employees whose salary exceeds the department average.

The important idea: divide a complex query into understandable stages.

# **2. CTE syntax**

Learn this structure thoroughly.

```
WITH cte_name AS (
    SELECT columns
    FROM table_name
    WHERE condition
)

SELECT *
FROM cte_name;
```

Breaking it down:

| Component     | Purpose                              |
| ------------- | ------------------------------------ |
| WITH          | Begins the CTE definition            |
| cte_name      | Gives the intermediate result a name |
| AS            | Introduces the query                 |
| SELECT inside | Generates the intermediate result    |
| Main SELECT   | Uses the CTE                         |

Important rules:

1. A CTE exists only for the SQL statement in which it is defined.
2. You can use multiple CTEs in the same statement.
3. A CTE can reference an earlier CTE defined in the same WITH clause.
4. In PostgreSQL, a CTE used in FROM behaves like a relation and can be joined like a table.
5. A CTE does not automatically mean that the database physically stores a temporary table.

## **3. Basic CTE examples**

### **Example 1: Find employees earning more than 70,000**

```
WITH high_salary_employees AS (
    SELECT *
    FROM employees
    WHERE salary > 70000
)

SELECT *
FROM high_salary_employees;
```

The CTE filters the employees. The main query retrieves the filtered result.

### **Example 2: Calculate average salary**

```
WITH salary_stats AS (
    SELECT
        AVG(salary) AS avg_salary,
        MAX(salary) AS max_salary,
        MIN(salary) AS min_salary
    FROM employees
)

SELECT *
FROM salary_stats;
```

Result:

| avg_salary | max_salary | min_salary |
| ---------- | ---------- | ---------- |
| 72857.14   | 90000      | 50000      |

### **Example 3: Department-wise employee count**

```
WITH department_count AS (
    SELECT
        department_id,
        COUNT(*) AS total_employees
    FROM employees
    GROUP BY department_id
)

SELECT *
FROM department_count
WHERE total_employees > 2;
```

Result:

| department_id | total_employees |
| ------------- | --------------- |
| 101           | 3               |

# **4. Multiple CTEs in a single query**

Important for complex OAs

You can define multiple CTEs separated by commas.

Consider this problem:

**Find employees who earn more than the average salary of their department, provided that their department has at least two employees.**

This involves two independent calculations:

1. Calculate average salary in each department.
2. Calculate employee count in each department.
3. Join those results with employees.
4. Apply the conditions.

Solution:

```
WITH department_average AS (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),

department_count AS (
    SELECT
        department_id,
        COUNT(*) AS total_employees
    FROM employees
    GROUP BY department_id
)

SELECT
    e.name,
    e.department_id,
    e.salary,
    da.avg_salary,
    dc.total_employees
FROM employees e

JOIN department_average da
    ON e.department_id = da.department_id

JOIN department_count dc
    ON e.department_id = dc.department_id

WHERE e.salary > da.avg_salary
  AND dc.total_employees >= 2;
```

Result:

| name   | department_id | salary | avg_salary | total_employees |
| ------ | ------------- | ------ | ---------- | --------------- |
| Rahul  | 102           | 80000  | 65000      | 2               |
| Priya  | 101           | 75000  | 73333      | 3               |
| Simran | 101           | 85000  | 73333      | 3               |

Notice that the query is organised into three logical parts.

This is much easier to understand and debug than deeply nested SQL.

# **5. CTE for solving second-highest salary**

Let's revisit an important interview problem.

**Find the second-highest distinct salary.**

Using CTE:

```
WITH distinct_salaries AS (
    SELECT DISTINCT salary
    FROM employees
),

ranked_salaries AS (
    SELECT
        salary,
        ROW_NUMBER() OVER (
            ORDER BY salary DESC
        ) AS rn
    FROM distinct_salaries
)

SELECT salary
FROM ranked_salaries
WHERE rn = 2;
```

Expected output:

```
85000
```

You may notice a new function:

```
ROW_NUMBER() OVER (ORDER BY salary DESC)
```

We'll study this thoroughly in Lesson 6.

For now, understand the strategy:

- CTE 1: Remove duplicate salaries.
- CTE 2: Assign sequential numbers to salaries.
- Main query: Retrieve row number 2.

This is a very common way to break a complicated problem into stages.

# **6. CTE vs Subquery vs Temporary Table vs View**

This is a favourite database interview topic.

| Feature                | CTE                      | Subquery                 | Temporary table                  | View                            |
| ---------------------- | ------------------------ | ------------------------ | -------------------------------- | ------------------------------- |
| Definition             | WITH clause              | Nested SELECT            | CREATE TEMP TABLE                | CREATE VIEW                     |
| Reusability            | Within one statement     | At its query location    | Across statements in session     | Across queries                  |
| Lifetime               | One statement            | Part of containing query | Session or transaction dependent | Persistent definition           |
| Physically stores data | Not necessarily          | Not necessarily          | Yes, when populated              | Usually stores query definition |
| Main purpose           | Organise complex queries | Nested logic             | Store intermediate data          | Reusable virtual relation       |

### **When should you use which?**

**Use a subquery:** When the nested operation is small and straightforward.

**Use a CTE:** When you want readable multi-step logic or reuse an intermediate result within a statement.

**Use a temporary table:** When you need to retain intermediate results across multiple statements or explicitly manage intermediate data.

**Use a view:** When you want a reusable named query definition.

### **Important interview question: Is a CTE always faster than a subquery?**

No. A CTE is primarily a readability and query-organisation feature. Performance depends on the database's query optimiser, the query structure, indexes and execution plan. Do not assume that rewriting a subquery as a CTE automatically improves performance.

# **7. Recursive CTEs**

Important advanced topic

This is where CTEs become particularly powerful.

A recursive CTE is a CTE that references itself.

It is used when data has hierarchical or repeated relationships.

Real-world examples:

- Employee-management hierarchies.
- Organisational structures.
- Category and subcategory trees.
- Folder structures.
- Graph traversal.
- Finding all descendants of a node.

## **A. Understand the problem**

Suppose an organisation has the following employee hierarchy.

**employees**

| emp_id | name   | manager_id |
| ------ | ------ | ---------- |
| 1      | Aman   | NULL       |
| 2      | Rahul  | 1          |
| 3      | Priya  | 1          |
| 4      | Neha   | 2          |
| 5      | Karan  | 2          |
| 6      | Simran | 3          |
| 7      | Riya   | 4          |

Here:

- Aman is the top-level manager.
- Rahul and Priya report to Aman.
- Neha and Karan report to Rahul.
- Simran reports to Priya.
- Riya reports to Neha.

**Question: Find the complete organisational hierarchy starting from Aman.**

A normal JOIN is not sufficient when the number of hierarchy levels is unknown.

We need recursion.

## **B. Syntax of recursive CTE**

```
WITH RECURSIVE cte_name AS (

    -- Anchor query
    SELECT ...

    UNION ALL

    -- Recursive query
    SELECT ...
    FROM table_name
    JOIN cte_name
        ON condition
)

SELECT *
FROM cte_name;
```

There are two critical parts.

**Recursive CTE structure**

1\. Anchor member

The initial query that identifies the starting row or rows.

2\. Recursive member

The query that retrieves the next level by referencing the previous result.

Repeat until no more rows are generated

Final result

Combined rows from the initial and recursive queries.

## **C. Complete recursive CTE example**

```
WITH RECURSIVE employee_hierarchy AS (

    -- Anchor member
    SELECT
        emp_id,
        name,
        manager_id,
        1 AS level
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive member
    SELECT
        e.emp_id,
        e.name,
        e.manager_id,
        eh.level + 1
    FROM employees e
    JOIN employee_hierarchy eh
        ON e.manager_id = eh.emp_id
)

SELECT *
FROM employee_hierarchy
ORDER BY level, emp_id;
```

Expected output:

| emp_id | name   | manager_id | level |
| ------ | ------ | ---------- | ----- |
| 1      | Aman   | NULL       | 1     |
| 2      | Rahul  | 1          | 2     |
| 3      | Priya  | 1          | 2     |
| 4      | Neha   | 2          | 3     |
| 5      | Karan  | 2          | 3     |
| 6      | Simran | 3          | 3     |
| 7      | Riya   | 4          | 4     |

Let's understand the recursion.

**Iteration 1 — Anchor query**

Finds Aman.

```
Aman
```

**Iteration 2 — Find Aman's employees**

```
Rahul
Priya
```

**Iteration 3 — Find their employees**

```
Neha
Karan
Simran
```

**Iteration 4 — Find their employees**

```
Riya
```

**Iteration 5 — No additional employees**

The recursive process terminates.

## **D. Recursion with hierarchy paths**

We can make the output more informative by tracking the full reporting path.

PostgreSQL example:

```
WITH RECURSIVE employee_hierarchy AS (

    SELECT
        emp_id,
        name,
        manager_id,
        1 AS level,
        name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.emp_id,
        e.name,
        e.manager_id,
        eh.level + 1,
        eh.path || ' -> ' || e.name
    FROM employees e
    JOIN employee_hierarchy eh
        ON e.manager_id = eh.emp_id
)

SELECT *
FROM employee_hierarchy
ORDER BY level, emp_id;
```

Output:

| name   | level | path                       |
| ------ | ----- | -------------------------- |
| Aman   | 1     | Aman                       |
| Rahul  | 2     | Aman → Rahul               |
| Priya  | 2     | Aman → Priya               |
| Neha   | 3     | Aman → Rahul → Neha        |
| Karan  | 3     | Aman → Rahul → Karan       |
| Simran | 3     | Aman → Priya → Simran      |
| Riya   | 4     | Aman → Rahul → Neha → Riya |

This is useful for real-world hierarchy and tree problems.

**Important:** Recursive queries should be designed to terminate. Cyclic relationships in the underlying data can cause repeated traversal. In production, use cycle detection or a safe recursion-depth limit where appropriate.

# **8. UNION ALL vs UNION in recursive CTEs**

These two operators are important.

| UNION                                        | UNION ALL                                                      |
| -------------------------------------------- | -------------------------------------------------------------- |
| Removes duplicate rows                       | Keeps duplicate rows                                           |
| Usually requires duplicate elimination       | Avoids that deduplication step                                 |
| Can help suppress repeated identical results | Commonly used when each recursive expansion should be retained |

Example:

```
SELECT 10
UNION
SELECT 10;
```

Result:

```
10
```

But:

```
SELECT 10
UNION ALL
SELECT 10;
```

Result:

```
10
10
```

In recursive CTEs, `UNION ALL` is commonly used because each iteration generates additional rows.

However, `UNION` does not universally prevent recursion loops: if the recursive output includes changing columns such as a level counter, repeated visits can still produce distinct rows.

# **9. Advanced CTE problem-solving examples**

Let's work through four more patterns.

## **Problem 1 — Find the department with the highest average salary**

```
WITH department_stats AS (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)

SELECT *
FROM department_stats
ORDER BY avg_salary DESC
LIMIT 1;
```

Expected result: Department 103, with an average salary of 90,000.

## **Problem 2 — Find the second-highest average department salary**

```
WITH department_stats AS (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
),

distinct_averages AS (
    SELECT DISTINCT avg_salary
    FROM department_stats
),

second_average AS (
    SELECT avg_salary
    FROM distinct_averages
    ORDER BY avg_salary DESC
    LIMIT 1 OFFSET 1
)

SELECT
    ds.department_id,
    ds.avg_salary
FROM department_stats ds
JOIN second_average sa
    ON ds.avg_salary = sa.avg_salary;
```

We use three CTEs:

1. Calculate department averages.
2. Extract distinct averages.
3. Identify the second-highest average.

Then retrieve the department or departments having that average.

This returns all tied departments.

## **Problem 3 — Find employees earning more than company average and their department average**

```
WITH company_stats AS (
    SELECT AVG(salary) AS company_avg
    FROM employees
),

department_stats AS (
    SELECT
        department_id,
        AVG(salary) AS department_avg
    FROM employees
    GROUP BY department_id
)

SELECT
    e.name,
    e.salary,
    cs.company_avg,
    ds.department_avg
FROM employees e
CROSS JOIN company_stats cs
JOIN department_stats ds
    ON e.department_id = ds.department_id
WHERE e.salary > cs.company_avg
  AND e.salary > ds.department_avg;
```

This is a good example of combining CTEs, CROSS JOIN and ordinary JOIN.

## **Problem 4 — Find the total number of employees below each manager**

For a simple direct-report count, a self-join and GROUP BY are enough:

```
SELECT
    m.emp_id,
    m.name,
    COUNT(e.emp_id) AS direct_reports
FROM employees m
LEFT JOIN employees e
    ON e.manager_id = m.emp_id
GROUP BY m.emp_id, m.name;
```

For **all descendants**, not just direct reports, use a recursive CTE.

This distinction is important in hierarchy-based OAs.

# **10. CTE interview questions**

| Question                                   | What you should know                                        |
| ------------------------------------------ | ----------------------------------------------------------- |
| What is a CTE?                             | Named intermediate result using WITH                        |
| Why use CTEs?                              | Readability and multi-step query organisation               |
| Can we use multiple CTEs?                  | Yes                                                         |
| Can a CTE reference another CTE?           | Yes, an earlier one                                         |
| What is a recursive CTE?                   | A CTE that refers to itself                                 |
| What is an anchor member?                  | Starting query                                              |
| What is a recursive member?                | Repeated expansion query                                    |
| CTE vs subquery?                           | Organisation and readability versus nested query expression |
| Does a CTE always create a physical table? | No                                                          |
| Does a CTE always make a query faster?     | No                                                          |
| How does recursion terminate?              | When the recursive member produces no new rows              |
| Can recursive CTEs find hierarchies?       | Yes                                                         |

# **11. Interactive assessment — CTEs**

**CTE assessment**

7 questions

**Q1. What keyword is used to define a CTE?**

CREATE

WITH

DECLARE

DEFINE

**Q2. What is the lifetime of an ordinary CTE?**

Permanently stored

Until the database shuts down

Within its SQL statement

For the entire session

**Q3. Which clause starts a recursive CTE in PostgreSQL?**

WITH LOOP

WITH RECURSIVE

REPEAT WITH

CREATE RECURSIVE

**Q4. What is an anchor member?**

The final SELECT

The starting query of the recursion

An index

A temporary table

**Q5. Which operator keeps duplicates?**

UNION

UNION ALL

DISTINCT

EXCEPT

**Q6. Can a CTE automatically be assumed to make a query faster?**

Yes, always

No, performance depends on the database optimiser and query

Only in PostgreSQL

Only when using UNION

**Q7. Which technique is suitable for an employee hierarchy with an unknown number of levels?**

Simple LIMIT

Recursive CTE

DISTINCT only

CROSS JOIN only

Submit assessment

# **12. Practical assignment — CTEs**

Write these queries independently.

| #  | Problem                                                                                                   | Difficulty |
| -- | --------------------------------------------------------------------------------------------------------- | ---------- |
| 1  | Find employees earning more than company average using a CTE.                                             | Easy       |
| 2  | Calculate department-wise average and maximum salaries.                                                   | Easy       |
| 3  | Find departments with more than two employees using a CTE.                                                | Easy       |
| 4  | Find the employee with the second-highest distinct salary using a CTE.                                    | Medium     |
| 5  | Find employees earning more than their department average using a CTE.                                    | Medium     |
| 6  | Find the department with the highest average salary.                                                      | Medium     |
| 7  | Use two CTEs to calculate company average and department average, then identify employees exceeding both. | Hard       |
| 8  | Build an employee hierarchy using a recursive CTE.                                                        | Hard       |
| 9  | Generate the full reporting path from the CEO to every employee.                                          | Hard       |
| 10 | Find every descendant of a specified employee in an organisational hierarchy.                             | Hard       |

# **13. Your SQL learning progress**

## **SQL Mastery Course**

Lessons completed

**5 / 10**

**01**

SQL Fundamentals

Covered

**02**

Aggregation, GROUP BY, HAVING

Covered

**03**

SQL JOINs

Covered

**04**

Subqueries

Covered

**05**

CTEs and Recursive CTEs

Covered

**06**

Window Functions

Next

**07**

Advanced SQL Functions

Upcoming

**08**

Database Design and Normalisation

Upcoming

**09**

Indexes, Transactions and Optimisation

Upcoming

**10**

Company OAs and Mock Interviews

Upcoming

# **Next lesson — Lesson 6: SQL Window Functions**

One of the most important OA topics

[Day 26: Window Functions - ROW_NUMBER, RANK, DENSE_RANK!](https://www.linkedin.com/pulse/day-26-window-functions-rownumber-rank-denserank-shubhendu-mishra-perof)

We'll cover:

1. Window functions and OVER().
2. PARTITION BY and ORDER BY.
3. ROW_NUMBER().
4. RANK() vs DENSE_RANK().
5. Finding second, third and Nth-highest salaries.
6. Top 3 employees in every department.
7. Running totals and cumulative averages.
8. LAG() and LEAD().
9. FIRST_VALUE() and LAST_VALUE().
10. Moving averages.
11. Ranking sales, customers and employees.
12. Real LeetCode SQL 50 and company OA problems.

**My recommendation:** Pay particular attention to `ROW_NUMBER()`, `RANK()` and `DENSE_RANK()`. These three functions unlock a large number of advanced SQL interview problems.