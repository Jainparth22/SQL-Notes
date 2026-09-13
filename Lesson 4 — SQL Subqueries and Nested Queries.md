# **Lesson 4 — SQL Subqueries and Nested Queries**



High priority for interviews and OAs

Intermediate → Advanced



Subqueries are one of the most important techniques for solving SQL interview questions.

They help you solve problems where the answer depends on another query's result.

Examples:

- Find employees earning more than the company's average salary.
- Find employees earning more than their department's average salary.
- Find the second-highest salary.
- Find customers who have never placed an order.
- Find the highest-paid employee in each department.
- Find employees whose salaries are greater than their managers' salaries.

By the end of this lesson, you should understand when to use a subquery, JOIN, or correlated subquery.

## **1. What is a subquery?**

A subquery is a SQL query written inside another SQL query.

Consider:

**employees**

| emp_id | name   | department_id | salary |
| ------ | ------ | ------------- | ------ |
| 1      | Aman   | 101           | 60000  |
| 2      | Rahul  | 102           | 80000  |
| 3      | Priya  | 101           | 75000  |
| 4      | Neha   | 103           | 90000  |
| 5      | Karan  | 102           | 50000  |
| 6      | Simran | 101           | 85000  |
| 7      | Riya   | 104           | 70000  |

**Problem:** Find employees earning more than the company's average salary.

We need two operations:

1. Calculate the average salary.
2. Find employees earning more than that average.

We can write two separate queries.

First:

```
SELECT AVG(salary)
FROM employees;
```

Result:

```
72857.14
```

Second:

```
SELECT *
FROM employees
WHERE salary > 72857.14;
```

But what if we combine these into one query?

```
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

That's a subquery.

**How the query works**

**Inner query**

```
SELECT AVG(salary)
FROM employees;
```

# **72,857.14**

Returns the company's average salary.

**Outer query**

```
WHERE salary > 72857.14
```

Filters employees using the inner query's result.

Final result

**Rahul, Priya, Neha and Simran**

Conceptually, understand the inner query first, then the outer query. Actual database optimisers may rewrite or execute the query differently.

# **2. Types of subqueries**

There are several ways to classify subqueries.

| Type                     | Meaning                        |
| ------------------------ | ------------------------------ |
| Scalar subquery          | Returns one value              |
| Single-row subquery      | Returns at most one row        |
| Multiple-row subquery    | Returns multiple rows          |
| Multiple-column subquery | Returns multiple columns       |
| Correlated subquery      | Depends on the outer query     |
| Non-correlated subquery  | Independent of the outer query |

Let's study them individually.

## **A. Scalar subquery**

A scalar subquery returns exactly one column and at most one row.

Example:

```
SELECT
    name,
    salary
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

The inner query returns a single numerical value.

**Interview rule:** A scalar subquery used as a value must return no more than one row. If it returns multiple rows, the database generally raises an error.

Example of a potentially problematic query:

```
SELECT *
FROM employees
WHERE salary = (
    SELECT salary
    FROM employees
    WHERE department_id = 101
);
```

Department 101 has multiple employees, so the subquery returns multiple salaries.

This cannot be used with ordinary scalar equality.

Solutions include `IN`, `ANY`, `ALL`, or an aggregate function, depending on the requirement.

## **B. Multiple-row subquery**

Suppose you need employees working in departments 101 and 103.

```
SELECT *
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE department_id IN (101, 103)
);
```

The inner query can return multiple department IDs.

The outer query uses `IN` to match them.

# **3. IN and NOT IN**

These operators are extremely important in OA questions.

### **IN**

`IN` checks whether a value matches at least one value in a list or subquery.

Example:

```
SELECT *
FROM employees
WHERE department_id IN (101, 103);
```

Equivalent to:

```
SELECT *
FROM employees
WHERE department_id = 101
   OR department_id = 103;
```

Now using a subquery:

```
SELECT *
FROM employees
WHERE department_id IN (
    SELECT department_id
    FROM departments
    WHERE department_name IN ('Engineering', 'Finance')
);
```

### **NOT IN**

Find employees who do not belong to departments 101 and 103:

```
SELECT *
FROM employees
WHERE department_id NOT IN (101, 103);
```

However, there is an important interview trap.

**NULL and NOT IN**

Suppose the subquery returns:

```
101
103
NULL
```

Then:

```
WHERE department_id NOT IN (
    SELECT department_id
    FROM some_table
);
```

may return no rows, because comparisons against that NULL can evaluate to UNKNOWN.

This is called the `NOT IN` NULL problem.

A safer alternative for anti-match problems is usually `NOT EXISTS`, which we'll discuss shortly.

# **4. EXISTS and NOT EXISTS**

Extremely important for interviews

`EXISTS` checks whether a subquery returns at least one row.

It returns TRUE if matching rows exist and FALSE otherwise.

Consider two tables:

**customers**

| customer_id | name  |
| ----------- | ----- |
| 1           | Aman  |
| 2           | Rahul |
| 3           | Priya |
| 4           | Neha  |

**orders**

| order_id | customer_id | amount |
| -------- | ----------- | ------ |
| 101      | 1           | 500    |
| 102      | 2           | 800    |
| 103      | 1           | 300    |

Question:

**Find customers who have placed at least one order.**

```
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

Result:

| customer_id | name  |
| ----------- | ----- |
| 1           | Aman  |
| 2           | Rahul |

Why?

- Aman has orders.
- Rahul has an order.
- Priya and Neha don't have orders.

The inner query checks whether a matching order exists for each outer customer.

### **Why SELECT 1?**

In an EXISTS subquery, the actual selected values do not matter. EXISTS only checks whether a row exists.

These are conceptually equivalent:

```
WHERE EXISTS (SELECT 1 FROM orders ...)
```

```
WHERE EXISTS (SELECT * FROM orders ...)
```

`SELECT 1` is a common convention because it makes the purpose clear.

## **NOT EXISTS**

Now reverse the requirement.

**Find customers who have never placed an order.**

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

Result:

| customer_id | name  |
| ----------- | ----- |
| 3           | Priya |
| 4           | Neha  |

This is one of the most useful SQL OA patterns.

### **IN vs EXISTS**

| IN                                    | EXISTS                                     |
| ------------------------------------- | ------------------------------------------ |
| Checks membership in a set            | Checks whether matching rows exist         |
| Useful for value matching             | Useful for existence checks                |
| Often straightforward for small lists | Natural for correlated relationship checks |
| NULL behaviour matters                | Usually safer for anti-match checks        |

Do not memorise that EXISTS is always faster than IN. Modern query optimisers can transform both into similar execution plans. Performance depends on the query, indexes and database engine.

# **5. Correlated vs non-correlated subqueries**

This is an important technical interview topic.

## **A. Non-correlated subquery**

An independent subquery does not reference the outer query.

Example:

```
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

The inner query does not depend on any particular outer employee.

## **B. Correlated subquery**

A correlated subquery refers to a column from the outer query.

Consider this problem:

**Find employees who earn more than their department's average salary.**

A normal company-wide average is not sufficient. We need a different average for every department.

Solution:

```
SELECT
    e.emp_id,
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

Understand the critical line:

```
WHERE e2.department_id = e.department_id
```

- `e` belongs to the outer query.
- `e2` belongs to the inner query.

The inner query uses the current outer employee's department to calculate the relevant average.

**Conceptual execution**

**Department 101**

Average salary = 73,333.33

Priya (75,000) and Simran (85,000) qualify.

**Department 102**

Average salary = 65,000

Rahul (80,000) qualifies.

**Department 103**

Average salary = 90,000

Neha does not qualify because her salary equals the average.

Final result:

| name   | department_id | salary |
| ------ | ------------- | ------ |
| Priya  | 101           | 75000  |
| Simran | 101           | 85000  |
| Rahul  | 102           | 80000  |

A correlated subquery is logically evaluated with reference to each applicable outer row, although the optimiser can transform it into a more efficient execution strategy.

**Interview answer:**

A correlated subquery references columns from its outer query and depends on the current outer row. A non-correlated subquery is independent and can be evaluated without referencing the outer query.

# **6. ANY and ALL operators**

These are less commonly used than IN and EXISTS, but they are useful in interviews.

## **A. ANY**

`ANY` means the condition must be true for at least one value returned by the subquery.

Example:

**Find employees whose salary is greater than at least one employee in department 102.**

```
SELECT *
FROM employees
WHERE salary > ANY (
    SELECT salary
    FROM employees
    WHERE department_id = 102
);
```

Department 102 salaries:

```
80000
50000
```

Aman earns 60000.

He earns more than at least one of these salaries (50000), so he qualifies.

Mathematically:

\\[ salary > ANY(S) \\]

means:

\\[ salary > \min(S) \\]

assuming the set contains non-NULL values.

## **B. ALL**

`ALL` means the condition must be true for every value returned by the subquery.

```
SELECT *
FROM employees
WHERE salary > ALL (
    SELECT salary
    FROM employees
    WHERE department_id = 102
);
```

Department 102 salaries:

```
80000
50000
```

An employee must earn more than both salaries.

Mathematically:

\\[ salary > ALL(S) \\]

means:

\\[ salary > \max(S) \\]

assuming non-NULL values.

### **ANY vs ALL**

| Operator | Requirement                                 |
| -------- | ------------------------------------------- |
| `> ANY`  | Greater than at least one value             |
| `> ALL`  | Greater than every value                    |
| `< ANY`  | Less than at least one value                |
| `< ALL`  | Less than every value                       |
| `= ANY`  | Equivalent to IN in ordinary membership use |

**Remember:** NULL values and empty subquery results can affect ANY/ALL comparisons. Learn the basic semantics first, then consider three-valued logic when debugging advanced queries.

# **7. Second-highest salary — classic interview question**

Must solve without memorising

This is one of the most commonly asked SQL interview problems.

Suppose salaries are:

| Employee | Salary |
| -------- | ------ |
| Aman     | 60000  |
| Rahul    | 80000  |
| Priya    | 75000  |
| Neha     | 90000  |
| Karan    | 50000  |
| Simran   | 85000  |

**Question: Find the second-highest distinct salary.**

We can solve this with a nested query.

### **Method 1 — MAX with subquery**

```
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```

Let's understand:

Step 1: Inner query finds maximum salary.

```
SELECT MAX(salary)
FROM employees;
```

Result: `90000`

Step 2: Outer query keeps salaries below 90000.

Step 3: MAX finds the largest remaining salary.

Result: `85000`

**Visualise the solution**

Highest salary

# **90,000**

Remove the highest

~~**90,000**~~

Next highest salary

# **85,000**

### **Method 2 — ORDER BY + LIMIT + OFFSET**

PostgreSQL and MySQL:

```
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

Explanation:

- `ORDER BY salary DESC`: Highest first.
- `DISTINCT`: Remove duplicate salaries.
- `OFFSET 1`: Skip the highest distinct salary.
- `LIMIT 1`: Return the next one.

### **Important interview distinction**

Suppose salaries are:

```
90000
90000
85000
80000
```

Without DISTINCT:

```
SELECT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

Returns `90000`.

With DISTINCT, it returns `85000`.

Therefore, understand whether the question asks for the second row or the second-highest distinct salary.

# **8. Nth-highest salary**

Now generalise the second-highest salary problem.

**Question: Find the third-highest distinct salary.**

```
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;
```

For the Nth-highest salary, the pattern is:

```
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET N_MINUS_1;
```

For the fifth-highest salary, OFFSET is 4.

Another important approach uses a nested query:

```
SELECT MIN(salary) AS third_highest_salary
FROM (
    SELECT DISTINCT salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 3
) AS top_salaries;
```

The inner query gets the top three distinct salaries. The outer query takes the minimum of those three.

We'll later learn how to solve this elegantly using `DENSE_RANK()`.

# **9. Find the highest-paid employee in each department**

This is another classic OA problem.

**Question: Return employees who have the maximum salary in their respective departments.**

```
SELECT
    e.emp_id,
    e.name,
    e.department_id,
    e.salary
FROM employees e
WHERE e.salary = (
    SELECT MAX(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

Output:

| name   | department_id | salary |
| ------ | ------------- | ------ |
| Simran | 101           | 85000  |
| Rahul  | 102           | 80000  |
| Neha   | 103           | 90000  |
| Riya   | 104           | 70000  |

Why does this work?

For every employee, the inner query calculates the maximum salary of that employee's department. The outer query retains employees whose salary equals that maximum.

**Important:** If two employees tie for the highest salary in a department, this query returns both employees. That is often exactly what an interviewer expects.

# **10. Find employees earning more than their managers**

This question combines SELF JOIN concepts with correlated subqueries.

Consider a different employee hierarchy table:

| emp_id | name  | manager_id | salary |
| ------ | ----- | ---------- | ------ |
| 1      | Aman  | NULL       | 100000 |
| 2      | Rahul | 1          | 90000  |
| 3      | Priya | 1          | 120000 |
| 4      | Neha  | 2          | 95000  |
| 5      | Karan | 2          | 70000  |

Question:

**Find employees whose salary is greater than their manager's salary.**

Solution using correlated subquery:

```
SELECT
    e.name,
    e.salary
FROM employees e
WHERE e.salary > (
    SELECT m.salary
    FROM employees m
    WHERE m.emp_id = e.manager_id
);
```

Output:

| name  | salary |
| ----- | ------ |
| Priya | 120000 |
| Neha  | 95000  |

Alternative using SELF JOIN:

```
SELECT
    e.name AS employee_name,
    e.salary AS employee_salary,
    m.name AS manager_name,
    m.salary AS manager_salary
FROM employees e
JOIN employees m
    ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

**Interview lesson:** The same problem can often be solved using JOINs or correlated subqueries. Learn both approaches and understand which is more readable for a given problem.

# **11. Subqueries inside FROM and SELECT**

Until now, most of our subqueries were inside WHERE.

Subqueries can also appear in FROM and SELECT.

## **A. Subquery inside FROM — derived table**

Question:

Find departments whose average salary is greater than 70,000.

```
SELECT *
FROM (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) AS department_stats
WHERE avg_salary > 70000;
```

Here, the inner query produces a temporary result set.

The outer query treats that result like a table.

This is called a **derived table**.

Important: In PostgreSQL, a subquery in FROM must have an alias.

## **B. Subquery inside SELECT**

Question:

Display every employee along with the company's average salary.

```
SELECT
    e.name,
    e.salary,
    (
        SELECT AVG(salary)
        FROM employees
    ) AS company_avg_salary
FROM employees e;
```

Result conceptually:

| name  | salary | company_avg_salary |
| ----- | ------ | ------------------ |
| Aman  | 60000  | 72857.14           |
| Rahul | 80000  | 72857.14           |
| Priya | 75000  | 72857.14           |
| Neha  | 90000  | 72857.14           |

The same company-wide average appears alongside each employee.

You can also make this correlated:

```
SELECT
    e.name,
    e.department_id,
    e.salary,
    (
        SELECT AVG(e2.salary)
        FROM employees e2
        WHERE e2.department_id = e.department_id
    ) AS department_avg
FROM employees e;
```

Now each employee gets their own department's average salary.

# **12. JOIN vs Subquery**

Interviewers frequently ask this question.

| Scenario                                           | Suitable approach                  |
| -------------------------------------------------- | ---------------------------------- |
| Retrieve related columns from multiple tables      | JOIN                               |
| Check whether a matching record exists             | EXISTS                             |
| Find records without a match                       | NOT EXISTS                         |
| Compare salary against company average             | Scalar subquery                    |
| Compare employee salary against department average | Correlated subquery                |
| Find aggregate statistics and filter them          | GROUP BY + HAVING or derived table |
| Find top N records per group                       | Window functions                   |

Neither JOIN nor subquery is universally faster.

Database optimisers can transform many query patterns into equivalent execution plans. For performance decisions, use `EXPLAIN` and actual query measurements.

# **13. SQL problem-solving patterns to memorise**

These are patterns, not individual solutions.

| Problem statement                         | Think about         |
| ----------------------------------------- | ------------------- |
| Greater than company average              | Scalar subquery     |
| Greater than department average           | Correlated subquery |
| Exists at least one order                 | EXISTS              |
| Never placed an order                     | NOT EXISTS          |
| Belongs to one of multiple departments    | IN                  |
| Greater than every salary in a department | ALL                 |
| Greater than at least one salary          | ANY                 |
| Second-highest distinct salary            | MAX + subquery      |
| Highest employee salary per department    | Correlated MAX      |
| Intermediate aggregated dataset           | Derived table       |

## **14. Common mistakes in subqueries**

**Mistake 1: Using&#x20;**`=`**&#x20;when the subquery returns multiple rows.**

Wrong:

```
WHERE department_id = (
    SELECT department_id
    FROM departments
);
```

Correct, when multiple values are expected:

```
WHERE department_id IN (
    SELECT department_id
    FROM departments
);
```

**Mistake 2: Forgetting correlation.**

To compare an employee against their department's average, this condition is essential:

```
WHERE e2.department_id = e.department_id
```

Without it, you're calculating the average for the entire company.

**Mistake 3: Using NOT IN with NULL values.**

For finding records without matches, prefer NOT EXISTS when the subquery's result could contain NULL.

**Mistake 4: Confusing second row with second distinct value.**

Remember:

```
LIMIT 1 OFFSET 1
```

does not automatically mean second-highest distinct salary. Use DISTINCT or an appropriate ranking technique.

**Mistake 5: Assuming a correlated subquery must execute separately for every row.**

That is a conceptual way to understand it, not a guaranteed physical execution strategy. The optimiser may rewrite it.

# **15. SQL interview assessment**

Test your understanding before moving to the practical problems.

**Subquery assessment**

7 questions

**Q1. What does a scalar subquery return?**

Exactly one table

One column and at most one row

Multiple columns necessarily

Only a Boolean

**Q2. Which operator is suitable when a subquery can return multiple values?**

\=

IN

IS NULL

AS

**Q3. What distinguishes a correlated subquery?**

It always uses GROUP BY

It references the outer query

It returns exactly one row

It cannot use aggregate functions

**Q4. Which is the natural approach to find customers with no orders?**

EXISTS

NOT EXISTS

ANY

COUNT(\*) without FROM

**Q5. In a department, salaries are 50,000, 50,000 and 80,000. What does second-highest DISTINCT salary mean?**

50,000

80,000

No result

130,000

**Q6. What can happen when a scalar subquery returns multiple rows?**

It always chooses the first row

It returns their average

The query generally raises an error

It automatically converts to IN

**Q7. Which is equivalent to salary > ALL(non-NULL values) in a nonempty set?**

Salary > MIN(value)

Salary > MAX(value)

Salary = MAX(value)

Salary < MAX(value)

Submit assessment

# **16. Practical SQL assignment**

Write these queries independently. These are representative of common interview and OA patterns.

**Use the employees table from this lesson unless the question explicitly refers to another schema.**

| #  | Problem                                                                 | Difficulty |
| -- | ----------------------------------------------------------------------- | ---------- |
| 1  | Find employees earning more than the company average salary.            | Easy       |
| 2  | Find employees earning less than the company average.                   | Easy       |
| 3  | Find the second-highest distinct salary.                                | Easy       |
| 4  | Find the third-highest distinct salary.                                 | Medium     |
| 5  | Find employees earning more than their department average.              | Medium     |
| 6  | Find the highest-paid employee in each department. Return all ties.     | Medium     |
| 7  | Find departments where at least one employee earns more than 80,000.    | Medium     |
| 8  | Find departments where every employee earns more than 50,000.           | Medium     |
| 9  | Find customers who have never placed an order.                          | Medium     |
| 10 | Find employees whose salary exceeds their manager's salary.             | Medium     |
| 11 | Find employees who earn the same salary as at least one other employee. | Medium     |
| 12 | Find the department with the second-highest average salary.             | Hard       |

For question 12, think about an aggregated derived table followed by a second query.

# **17. Your SQL mastery roadmap: where we are**

## **SQL learning progress**

1. **SQL fundamentals**

   Covered

   SELECT, WHERE, ORDER BY, NULL, CASE
2. **Aggregation**

   Covered

   COUNT, SUM, AVG, GROUP BY, HAVING
3. **JOINs**

   Covered

   INNER, LEFT, RIGHT, FULL, CROSS, SELF
4. **Subqueries**

   Covered

   IN, EXISTS, ANY, ALL, correlated queries
5. **CTEs**

   Next

   WITH clause, multiple CTEs, recursive CTEs
6. **Window functions**\
   ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD
7. **Advanced SQL**\
   Date functions, string functions, conditional aggregation
8. **Database internals**\
   Indexes, transactions, ACID, isolation and query optimisation
9. **Interview/OA mastery**\
   LeetCode SQL 50, company questions and mock interviews

## **Next lesson — Lesson 5: CTEs (Common Table Expressions)**

We'll learn:

1. What a CTE is and why it exists.
2. `WITH` clause syntax.
3. Multiple CTEs in one query.
4. CTE vs subquery vs temporary table.
5. Recursive CTEs.
6. Employee hierarchy and organisational tree problems.
7. Multi-step aggregation problems.
8. Real interview questions using CTEs.

**One important recommendation:** Don't try to memorise 50 SQL queries. Learn to recognise the underlying patterns: filtering, grouping, matching, existence, comparison and ranking. That is what will make you capable of solving unfamiliar OA questions.