# **Lesson 3 — SQL JOINs: Complete Interview & OA Mastery**



Extremely important

SDE interviews + coding assessments



JOINs are among the most frequently tested SQL concepts. They allow you to retrieve related information from multiple tables.

Our objective is to master JOINs so that you can solve problems involving employees, departments, customers, orders, products, payments and other relational data.

# **1. Why do we need JOINs?**

Consider two tables.

**Table 1: employees**

| emp_id | name   | department_id | salary |
| ------ | ------ | ------------- | ------ |
| 1      | Aman   | 101           | 60000  |
| 2      | Rahul  | 102           | 80000  |
| 3      | Priya  | 101           | 75000  |
| 4      | Neha   | 103           | 90000  |
| 5      | Karan  | 102           | 50000  |
| 6      | Simran | 101           | 85000  |
| 7      | Riya   | 104           | 70000  |
| 8      | Dev    | NULL          | 55000  |

**Table 2: departments**

| department_id | department_name |
| ------------- | --------------- |
| 101           | Engineering     |
| 102           | HR              |
| 103           | Finance         |
| 104           | Marketing       |
| 105           | Operations      |

Notice something?

The employee table contains `department_id`, but it doesn't contain the department's full name.

The department table contains the department names.

Suppose the interviewer asks:

**Find each employee's name, salary and department name.**

You need information from both tables.

This is where JOIN comes into the picture.

# **2. Basic JOIN syntax**

```
SELECT columns
FROM table1
JOIN table2
ON table1.common_column = table2.common_column;
```

Example:

```
SELECT
    employees.name,
    employees.salary,
    departments.department_name
FROM employees
JOIN departments
ON employees.department_id = departments.department_id;
```

Output:

| name   | salary | department_name |
| ------ | ------ | --------------- |
| Aman   | 60000  | Engineering     |
| Rahul  | 80000  | HR              |
| Priya  | 75000  | Engineering     |
| Neha   | 90000  | Finance         |
| Karan  | 50000  | HR              |
| Simran | 85000  | Engineering     |
| Riya   | 70000  | Marketing       |

Dev is missing because his department ID is NULL.

Operations is also missing because no employee belongs to that department.

Why?

Because we used an INNER JOIN.

# **3. Types of SQL JOINs**

[Day 32 (week 7) — Inner and Outer Joins | by Eric Abell | Medium](https://medium.com/%40eric_abell/day-32-week-7-inner-and-outer-joins-a0b9aa8b2270)

There are six major JOIN patterns you should understand.

| JOIN            | What it returns                     |
| --------------- | ----------------------------------- |
| INNER JOIN      | Matching rows from both tables      |
| LEFT JOIN       | All rows from left + matching right |
| RIGHT JOIN      | All rows from right + matching left |
| FULL OUTER JOIN | All rows from both tables           |
| CROSS JOIN      | Every possible combination          |
| SELF JOIN       | A table joined with itself          |

Let's understand each.

# **4. INNER JOIN**

Most frequently used

INNER JOIN returns only the rows where the join condition matches in both tables.

```
SELECT
    e.name,
    d.department_name
FROM employees e
INNER JOIN departments d
ON e.department_id = d.department_id;
```

Here, `e` and `d` are table aliases.

Output:

| name   | department_name |
| ------ | --------------- |
| Aman   | Engineering     |
| Rahul  | HR              |
| Priya  | Engineering     |
| Neha   | Finance         |
| Karan  | HR              |
| Simran | Engineering     |
| Riya   | Marketing       |

Dev is excluded because NULL does not match any department ID.

Operations is excluded because no employee matches it.

**Important:** `JOIN` without specifying a type means `INNER JOIN` in standard SQL.

### **When do you use INNER JOIN?**

When you only want records that have corresponding matches in both tables.

Examples:

- Customers who have placed orders.
- Employees belonging to existing departments.
- Students enrolled in courses.
- Products that have sales records.

**Interview question: What happens when there is no matching record in an INNER JOIN?**

The unmatched record is excluded from the result. INNER JOIN only returns rows satisfying the join condition, so unmatched rows from either table are not included.

# **5. LEFT JOIN**

Very important for OAs

LEFT JOIN returns:

1. Every record from the left table.
2. Matching records from the right table.
3. NULL values for right-side columns when no match exists.

Syntax:

```
SELECT columns
FROM table1
LEFT JOIN table2
ON table1.common_column = table2.common_column;
```

Example:

```
SELECT
    e.name,
    e.salary,
    d.department_name
FROM employees e
LEFT JOIN departments d
ON e.department_id = d.department_id;
```

Output:

| name   | salary | department_name |
| ------ | ------ | --------------- |
| Aman   | 60000  | Engineering     |
| Rahul  | 80000  | HR              |
| Priya  | 75000  | Engineering     |
| Neha   | 90000  | Finance         |
| Karan  | 50000  | HR              |
| Simran | 85000  | Engineering     |
| Riya   | 70000  | Marketing       |
| Dev    | 55000  | NULL            |

Dev is included because all employees from the left table must be returned.

### **Practical example**

**Question: Find all employees, including those who are not assigned to any department.**

```
SELECT
    e.emp_id,
    e.name,
    d.department_name
FROM employees e
LEFT JOIN departments d
ON e.department_id = d.department_id;
```

This is a classic LEFT JOIN application.

### **Another important OA pattern**

**Find employees who do not belong to any department.**

```
SELECT e.name
FROM employees e
LEFT JOIN departments d
ON e.department_id = d.department_id
WHERE d.department_id IS NULL;
```

Output:

| name |
| ---- |
| Dev  |

This is called an **ANTI JOIN pattern**.

We'll study this pattern more deeply in subqueries.

# **6. RIGHT JOIN**

RIGHT JOIN is the opposite orientation of LEFT JOIN.

It returns:

- All rows from the right table.
- Matching rows from the left table.
- NULL values for unmatched left-side columns.

Example:

```
SELECT
    e.name,
    d.department_name
FROM employees e
RIGHT JOIN departments d
ON e.department_id = d.department_id;
```

Output:

| name   | department_name |
| ------ | --------------- |
| Aman   | Engineering     |
| Rahul  | HR              |
| Priya  | Engineering     |
| Neha   | Finance         |
| Karan  | HR              |
| Simran | Engineering     |
| Riya   | Marketing       |
| NULL   | Operations      |

Operations appears even though no employee belongs to it.

**Practical use case:** Find all departments, including departments that currently have no employees.

In practice, you can usually rewrite RIGHT JOIN as LEFT JOIN by switching the table order. Many developers prefer LEFT JOIN because it is easier to read consistently.

# **7. FULL OUTER JOIN**

Important interview concept

FULL OUTER JOIN returns everything from both tables:

- Matching records.
- Unmatched records from the left table.
- Unmatched records from the right table.

```
SELECT
    e.name,
    d.department_name
FROM employees e
FULL OUTER JOIN departments d
ON e.department_id = d.department_id;
```

Conceptual output:

| name   | department_name |
| ------ | --------------- |
| Aman   | Engineering     |
| Rahul  | HR              |
| Priya  | Engineering     |
| Neha   | Finance         |
| Karan  | HR              |
| Simran | Engineering     |
| Riya   | Marketing       |
| Dev    | NULL            |
| NULL   | Operations      |

Note: The exact display order is not guaranteed without ORDER BY.

### **PostgreSQL vs MySQL**

PostgreSQL supports FULL OUTER JOIN directly.

MySQL does not support native FULL OUTER JOIN syntax.

A typical MySQL alternative is:

```
SELECT
    e.name,
    d.department_name
FROM employees e
LEFT JOIN departments d
ON e.department_id = d.department_id

UNION ALL

SELECT
    e.name,
    d.department_name
FROM employees e
RIGHT JOIN departments d
ON e.department_id = d.department_id
WHERE e.emp_id IS NULL;
```

Why use `UNION ALL`?

The LEFT JOIN already returns matching records and unmatched employee records. The second query adds only unmatched department records, avoiding duplicated matching records.

# **8. CROSS JOIN**

CROSS JOIN returns the Cartesian product of two tables.

Suppose:

Table A:

| color |
| ----- |
| Red   |
| Blue  |

Table B:

| size  |
| ----- |
| Small |
| Large |

Query:

```
SELECT *
FROM colors
CROSS JOIN sizes;
```

Output:

| color | size  |
| ----- | ----- |
| Red   | Small |
| Red   | Large |
| Blue  | Small |
| Blue  | Large |

Number of resulting rows:

\\[ 2 \times 2 = 4 \\]

For our employee and department tables:

\\[ 8 \times 5 = 40 \\]

**Interview question: What is a Cartesian product?**

A Cartesian product combines each row of one table with every row of another table. A CROSS JOIN produces this result, and the number of rows equals the product of the two table sizes.

Common applications:

- Generating every date and product combination.
- Creating combinations of sizes and colors.
- Generating testing datasets.

Be careful: accidentally missing a JOIN condition can produce an enormous Cartesian product.

# **9. SELF JOIN**

This is a very important interview concept.

A SELF JOIN means joining a table with itself.

Consider an employee hierarchy.

**employees**

| emp_id | name  | manager_id |
| ------ | ----- | ---------- |
| 1      | Aman  | NULL       |
| 2      | Rahul | 1          |
| 3      | Priya | 1          |
| 4      | Neha  | 2          |
| 5      | Karan | 2          |

The `manager_id` column references another employee's `emp_id`.

Question:

**Find each employee and their manager's name.**

```
SELECT
    e.name AS employee_name,
    m.name AS manager_name
FROM employees e
LEFT JOIN employees m
ON e.manager_id = m.emp_id;
```

Output:

| employee_name | manager_name |
| ------------- | ------------ |
| Aman          | NULL         |
| Rahul         | Aman         |
| Priya         | Aman         |
| Neha          | Rahul        |
| Karan         | Rahul        |

Here:

- `e` represents the employee.
- `m` represents the manager.

Both aliases refer to the same physical table.

Why LEFT JOIN?

Because Aman does not have a manager, but we still want to display Aman.

### **Real-world applications**

- Employee-manager hierarchies.
- Categories and parent categories.
- Linked nodes.
- Referral relationships.
- Finding people from the same team.

**Remember:** SELF JOIN is not a separate SQL keyword. It is a normal JOIN where both sides reference the same table.

# **10. Joining more than two tables**

Real-world OAs frequently provide three or four tables.

Let's introduce an order management database.

**customers**

| customer_id | customer_name |
| ----------- | ------------- |
| 1           | Aman          |
| 2           | Rahul         |
| 3           | Priya         |

**orders**

| order_id | customer_id | amount |
| -------- | ----------- | ------ |
| 101      | 1           | 500    |
| 102      | 2           | 800    |
| 103      | 1           | 300    |

**payments**

| payment_id | order_id | payment_status |
| ---------- | -------- | -------------- |
| 1          | 101      | Completed      |
| 2          | 102      | Pending        |
| 3          | 103      | Completed      |

Question:

**Find customer names, their order IDs, order amounts and payment statuses.**

Solution:

```
SELECT
    c.customer_name,
    o.order_id,
    o.amount,
    p.payment_status
FROM customers c
INNER JOIN orders o
    ON c.customer_id = o.customer_id
INNER JOIN payments p
    ON o.order_id = p.order_id;
```

Output:

| customer_name | order_id | amount | payment_status |
| ------------- | -------- | ------ | -------------- |
| Aman          | 101      | 500    | Completed      |
| Rahul         | 102      | 800    | Pending        |
| Aman          | 103      | 300    | Completed      |

Notice how:

1. Customers are joined with orders using `customer_id`.
2. Orders are joined with payments using `order_id`.

You can chain multiple JOINs like this.

### **Important OA strategy**

Before writing a query involving several tables, identify the relationships.

| From        | To          | Joining key |
| ----------- | ----------- | ----------- |
| customers   | orders      | customer_id |
| orders      | payments    | order_id    |
| orders      | order_items | order_id    |
| order_items | products    | product_id  |

Understanding the schema is often half the problem.

# **11. ON vs WHERE — a very important interview trap**

This is where many candidates make mistakes.

Consider:

**Question: Find all departments and their employees who are from Delhi. Departments without Delhi employees must also be displayed.**

Incorrect query:

```
SELECT
    d.department_name,
    e.name
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
WHERE e.city = 'Delhi';
```

Why is this wrong?

The WHERE clause removes rows where `e.city` is NULL. So departments without matching Delhi employees disappear.

Correct query:

```
SELECT
    d.department_name,
    e.name
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
   AND e.city = 'Delhi';
```

The city restriction is now part of the JOIN condition.

This keeps all departments and matches only their Delhi employees.

**Remember this distinction**

**Condition in ON**

Controls which rows match during the JOIN. Unmatched left-side rows are preserved by LEFT JOIN.

**Condition in WHERE**

Filters the result after the JOIN. It can remove unmatched rows and change the effect of an outer JOIN.

This is an excellent interview discussion point.

# **12. JOIN + GROUP BY — the most useful combination**

Now let's combine everything from Lessons 1, 2 and 3.

### **Problem 1: Find the number of employees in each department.**

We need department names, not just department IDs.

```
SELECT
    d.department_name,
    COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
ORDER BY employee_count DESC;
```

Output:

| department_name | employee_count |
| --------------- | -------------- |
| Engineering     | 3              |
| HR              | 2              |
| Finance         | 1              |
| Marketing       | 1              |
| Operations      | 0              |

Notice something important:

We used `LEFT JOIN`, not `INNER JOIN`, because we also want departments having zero employees.

We used `COUNT(e.emp_id)`, not `COUNT(*)`.

**Why?**

For Operations, LEFT JOIN produces a row containing NULL on the employee side.

- COUNT(\*) counts that row, resulting in 1.
- COUNT(e.emp_id) ignores the NULL, resulting in 0.

This is a classic OA trap.

### **Problem 2: Find the average salary of each department.**

```
SELECT
    d.department_name,
    AVG(e.salary) AS avg_salary
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name;
```

### **Problem 3: Find departments having at least two employees.**

```
SELECT
    d.department_name,
    COUNT(e.emp_id) AS employee_count
FROM departments d
LEFT JOIN employees e
    ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name
HAVING COUNT(e.emp_id) >= 2;
```

Notice the combination:

- LEFT JOIN to include departments.
- GROUP BY to create department groups.
- COUNT to calculate employees.
- HAVING to filter groups.

This is how actual SQL OA problems are constructed.

# **13. JOIN pitfalls you must know**

| Mistake                                          | Why it matters                    |
| ------------------------------------------------ | --------------------------------- |
| Missing ON condition                             | Can produce a Cartesian product   |
| Joining wrong columns                            | Produces incorrect matches        |
| Using INNER instead of LEFT                      | May lose unmatched records        |
| WHERE on right-side column after LEFT JOIN       | Can remove unmatched rows         |
| COUNT(\*) after LEFT JOIN                        | May count NULL-extended rows      |
| Forgetting duplicate matches                     | Can multiply result rows          |
| Joining tables without understanding cardinality | Can inflate SUM and COUNT results |

### **Advanced concept: JOIN multiplication**

Suppose a customer has two orders, and each order has three order items.

Joining customers → orders → order_items produces six rows for that customer.

This is not necessarily an error. It is the correct relational result.

However, summing order amounts after that JOIN can accidentally count each order multiple times.

For example, if an order amount is 500 and it has three items, a joined result may contain:

| order_id | order_amount | item_id |
| -------- | ------------ | ------- |
| 101      | 500          | 1       |
| 101      | 500          | 2       |
| 101      | 500          | 3       |

Then:

```
SUM(order_amount)
```

returns 1500 instead of 500.

**Solution:** Aggregate at the correct level before joining, or use a separate subquery/CTE to calculate totals.

We'll work on this when we study subqueries and CTEs.

# **14. Interview knowledge checkpoint**

These are common questions you should be able to answer.

| Question                                   | Expected answer                                         |
| ------------------------------------------ | ------------------------------------------------------- |
| INNER JOIN vs LEFT JOIN?                   | Matching records vs all left records plus matches       |
| LEFT JOIN vs RIGHT JOIN?                   | Preserves left vs preserves right                       |
| FULL OUTER JOIN?                           | Preserves unmatched records from both tables            |
| What is SELF JOIN?                         | Joining a table with itself                             |
| What is CROSS JOIN?                        | Cartesian product                                       |
| Does JOIN automatically remove duplicates? | No                                                      |
| Can we join more than two tables?          | Yes                                                     |
| Does LEFT JOIN preserve NULL values?       | Yes, where no match exists                              |
| What is an anti-join?                      | Find rows without a match                               |
| What is join cardinality?                  | Number of rows produced by relationships between tables |

# **15. Interactive SQL assessment**

These questions test your understanding, not just your memory.

**SQL JOIN assessment**

7 questions

**Q1. You need all customers, including customers who never placed an order. Which JOIN should you use?**

INNER JOIN

LEFT JOIN from customers to orders

RIGHT JOIN from customers to orders

CROSS JOIN

**Q2. What happens to unmatched right-side columns in a LEFT JOIN?**

They become 0

They become empty strings

They become NULL

The entire row is removed

**Q3. Which JOIN is used to find employees and their manager names from the same table?**

CROSS JOIN

SELF JOIN

FULL JOIN only

UNION

**Q4. A LEFT JOIN has 10 left-table records, and one record has no match. How many rows are returned?**

9

10

11

Depends on the right table size only

**Q5. You need every possible combination of 4 products and 3 sizes. What is the resulting row count?**

7

12

4

3

**Q6. Which query counts departments with zero employees correctly after a LEFT JOIN?**

COUNT(\*)

COUNT(e.emp_id)

COUNT(d.department_id)

SUM(d.department_id)

**Q7. What can happen if you put e.city = 'Delhi' in WHERE after a LEFT JOIN?**

All unmatched departments are preserved

It can remove unmatched departments

It converts the LEFT JOIN into CROSS JOIN

It automatically creates a SELF JOIN

Submit assessment

# **16. Practical OA assignment**

Now write the SQL queries yourself.

Use the `employees` and `departments` tables introduced in this lesson.

| #  | Problem                                                              | Difficulty |
| -- | -------------------------------------------------------------------- | ---------- |
| 1  | Display every employee with their department name.                   | Easy       |
| 2  | Find employees who do not belong to any department.                  | Easy       |
| 3  | Display every department, including departments with zero employees. | Easy       |
| 4  | Count the employees in each department, including zero.              | Medium     |
| 5  | Find departments having an average salary greater than 70,000.       | Medium     |
| 6  | Display employee names and their manager names using SELF JOIN.      | Medium     |
| 7  | Find employees whose department is Engineering.                      | Easy       |
| 8  | Find departments that do not have any employees.                     | Medium     |
| 9  | Find the total salary expenditure for each department.               | Medium     |
| 10 | Find the department with the maximum number of employees.            | Medium     |

For questions 6–8, use the additional manager and unmatched-department examples explained above where applicable.

# **17. What you should know before the next lesson**

**Lesson 3 completion checklist**

Your learning progress

**0/11**

Understand INNER JOIN

Understand LEFT JOIN

Understand RIGHT JOIN

Understand FULL OUTER JOIN

Understand CROSS JOIN

Understand SELF JOIN

Join three or more tables

Understand ON vs WHERE

Understand anti-join patterns

Solve JOIN + GROUP BY problems

Understand duplicate multiplication

## **Next: Lesson 4 — Subqueries and Nested SQL**

[Subquery in SQL: A Beginner’s Guide | by Fatmanurkutlu | Medium](https://medium.com/%40fatmanurkutlu1/subquery-in-sql-a-beginners-guide-4f9256eadadc)

We will cover:

1. Single-row and multiple-row subqueries.
2. Subqueries with IN, EXISTS, ANY and ALL.
3. Correlated vs non-correlated subqueries.
4. NOT EXISTS and anti-join problems.
5. Second-highest and Nth-highest salary.
6. Employees earning more than their department average.
7. Find employees earning more than their managers.
8. Find the highest-paid employee in each department.
9. Subquery vs JOIN: when to use which.
10. Real interview and OA problems.

**My advice:** Do not rush through JOINs. Make sure you can independently write INNER JOIN, LEFT JOIN, anti-join and JOIN + GROUP BY queries. These are essential foundations for advanced SQL problem-solving.