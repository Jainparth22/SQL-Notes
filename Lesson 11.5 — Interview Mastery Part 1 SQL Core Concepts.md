# **SQL Interview Mastery — Part 1**

## **How SQL actually executes a query**

Advanced revision

SDE interviews

PostgreSQL

You've covered SQL syntax, joins, subqueries, CTEs, window functions, database design, indexing, transactions and OA problems.

Now we'll prepare for a different type of interview question:

- Why does this query return the wrong result?
- Why can't I use an alias in `WHERE`?
- What's the difference between `WHERE` and `HAVING`?
- Why does `COUNT(*)` return 1 for a customer without orders?
- Why does a `LEFT JOIN` sometimes behave like an `INNER JOIN`?
- Why does `NOT IN` fail when the subquery contains NULL?

These questions test whether you understand SQL semantics rather than just its syntax.

# **1. SQL logical execution order**

This is one of the most frequently tested SQL interview concepts.

Consider:

```
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
WHERE salary > 30000
GROUP BY department_id
HAVING AVG(salary) > 50000
ORDER BY avg_salary DESC
LIMIT 3;
```

You might naturally read the query from top to bottom.

But SQL's **logical processing order** is different.

**Logical SQL execution order**

1. **FROM**

   Identify the source tables.
2. **JOIN / ON**

   Combine rows and evaluate join conditions.
3. **WHERE**

   Filter individual rows before grouping.
4. **GROUP BY**

   Create groups of rows.
5. **HAVING**

   Filter groups after aggregation.
6. **SELECT**

   Calculate the output expressions.
7. **DISTINCT**

   Remove duplicate output rows.
8. **ORDER BY**

   Sort the result.
9. **LIMIT / OFFSET**

   Restrict the final output.

A useful simplified sequence:

```
FROM / JOIN
     ↓
   WHERE
     ↓
  GROUP BY
     ↓
   HAVING
     ↓
   SELECT
     ↓
  DISTINCT
     ↓
  ORDER BY
     ↓
 LIMIT / OFFSET
```

**Important distinction:** This is logical processing order, not necessarily the physical order in which the database engine executes operations. The optimizer can rearrange physical operations as long as query semantics are preserved.

### **Interview question: Why can't we use a SELECT alias in WHERE?**

Consider:

```
SELECT
    salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 600000;
```

In PostgreSQL, this produces an error.

Why?

Because `WHERE` logically executes before `SELECT`. The alias `annual_salary` hasn't been defined at the logical stage where `WHERE` is evaluated.

**Correct solution:**

```
SELECT
    salary * 12 AS annual_salary
FROM employees
WHERE salary * 12 > 600000;
```

Alternatively, use a CTE:

```
WITH employee_salary AS (
    SELECT
        emp_id,
        salary * 12 AS annual_salary
    FROM employees
)
SELECT *
FROM employee_salary
WHERE annual_salary > 600000;
```

### **Can we use SELECT aliases in ORDER BY?**

Yes.

```
SELECT
    emp_id,
    salary * 12 AS annual_salary
FROM employees
ORDER BY annual_salary DESC;
```

`ORDER BY` logically comes after `SELECT`, so the output alias is available.

### **Can we use aliases in HAVING?**

PostgreSQL does not allow a SELECT-list alias to be referenced directly in `HAVING`.

```
-- PostgreSQL: invalid
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING avg_salary > 50000;
```

Correct:

```
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 50000;
```

# **2. WHERE vs HAVING**

This is a classic interview question.

**WHERE filters individual rows. HAVING filters aggregated groups.**

### **Example dataset**

| emp_id | department_id | salary |
| ------ | ------------- | ------ |
| 1      | 10            | 40000  |
| 2      | 10            | 60000  |
| 3      | 10            | 80000  |
| 4      | 20            | 30000  |
| 5      | 20            | 90000  |

### **WHERE example**

Find employees earning more than ₹50,000.

```
SELECT *
FROM employees
WHERE salary > 50000;
```

This filters individual employees.

### **HAVING example**

Find departments whose average salary exceeds ₹50,000.

```
SELECT
    department_id,
    AVG(salary)
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 50000;
```

This filters groups.

### **Combine both**

Find departments where employees earning more than ₹30,000 have an average salary above ₹60,000.

```
SELECT
    department_id,
    AVG(salary) AS average_salary
FROM employees
WHERE salary > 30000
GROUP BY department_id
HAVING AVG(salary) > 60000;
```

Execution:

1. Remove employees earning ₹30,000 or less.
2. Group the remaining employees by department.
3. Calculate each department's average.
4. Retain only departments with an average above ₹60,000.

### **Interview comparison**

| WHERE                        | HAVING                                                      |
| ---------------------------- | ----------------------------------------------------------- |
| Filters rows                 | Filters groups                                              |
| Before GROUP BY              | After GROUP BY                                              |
| Normally filters raw columns | Commonly filters aggregate results                          |
| Can be used without GROUP BY | Can be used without explicit GROUP BY for aggregate queries |

**Interviewer-style answer:** WHERE filters individual records before grouping, whereas HAVING filters aggregated groups after GROUP BY. Use WHERE to reduce input rows and HAVING to apply conditions on aggregate results.

# **3. NULL — the concept that causes many incorrect queries**

NULL does not mean:

- Zero
- Empty string
- False
- A missing numeric value that can be compared normally

It represents an unknown or absent value.

Consider:

| emp_id | emp_name | manager_id |
| ------ | -------- | ---------- |
| 1      | A        | NULL       |
| 2      | B        | 1          |
| 3      | C        | 1          |
| 4      | D        | NULL       |

### **Mistake 1: Comparing NULL with equals**

Incorrect:

```
SELECT *
FROM employees
WHERE manager_id = NULL;
```

Correct:

```
SELECT *
FROM employees
WHERE manager_id IS NULL;
```

For non-NULL values:

```
WHERE manager_id IS NOT NULL;
```

Why?

SQL uses three-valued logic:

- TRUE
- FALSE
- UNKNOWN

A comparison with NULL generally evaluates to UNKNOWN.

For example:

```
NULL = NULL
```

Does not produce TRUE.

### **Mistake 2: Using NOT IN with NULL**

Suppose a subquery returns:

```
1
2
NULL
```

You execute:

```
SELECT *
FROM customers
WHERE customer_id NOT IN (
    SELECT customer_id
    FROM orders
);
```

If the subquery includes a NULL, the comparison may evaluate to UNKNOWN for otherwise unmatched customers. Consequently, no customers may be returned.

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

This directly checks whether a matching order exists.

### **Mistake 3: Misunderstanding COUNT**

Suppose a department has no employees.

```
SELECT
    d.department_id,
    COUNT(*) AS count1,
    COUNT(e.emp_id) AS count2,
    COUNT(e.salary) AS count3
FROM departments d
LEFT JOIN employees e
    ON e.department_id = d.department_id
GROUP BY d.department_id;
```

For that empty department:

| Expression        | Result |
| ----------------- | ------ |
| `COUNT(*)`        | 1      |
| `COUNT(e.emp_id)` | 0      |
| `COUNT(e.salary)` | 0      |

Why does `COUNT(*)` return 1?

A LEFT JOIN preserves the department and produces one NULL-extended row.

`COUNT(*)` counts rows, while `COUNT(column)` counts non-NULL values in that column.

### **Mistake 4: AVG ignores NULL**

Suppose salaries are:

```
10000, 20000, NULL
```

Then:

```
SELECT AVG(salary)
FROM employees;
```

Returns:

\\[ \frac{10000+20000}{2}=15000 \\]

It does not divide by three.

### **Important NULL functions**

| Function               | Meaning                      |
| ---------------------- | ---------------------------- |
| `COALESCE(a,b)`        | First non-NULL expression    |
| `NULLIF(a,b)`          | Returns NULL when a equals b |
| `IS NULL`              | Checks NULL                  |
| `IS NOT NULL`          | Checks non-NULL              |
| `IS DISTINCT FROM`     | NULL-safe inequality         |
| `IS NOT DISTINCT FROM` | NULL-safe equality           |

Example:

```
SELECT COALESCE(manager_id, 0)
FROM employees;
```

And:

```
SELECT 100 / NULLIF(denominator, 0);
```

The latter prevents division by zero by converting a zero denominator into NULL.

### **A particularly useful PostgreSQL operator**

```
SELECT *
FROM employees
WHERE manager_id IS NOT DISTINCT FROM department_id;
```

Unlike ordinary equality, this treats two NULL values as equal.

**Interviewer-style answer:** NULL represents an unknown or missing value. Comparisons involving NULL generally produce UNKNOWN, so we use IS NULL, IS NOT NULL or NULL-safe comparison operators instead of ordinary equality.

# **4. JOIN semantics — why a LEFT JOIN can become an INNER JOIN**

Consider:

```
customers (
    customer_id,
    customer_name
);

orders (
    order_id,
    customer_id,
    status
);
```

Requirement: Return every customer, along with their completed orders. Customers without completed orders must also appear.

### **Incorrect approach**

```
SELECT
    c.customer_id,
    c.customer_name,
    o.order_id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
WHERE o.status = 'Completed';
```

Why does this fail?

For customers without orders, the joined `o.status` is NULL.

The condition:

```
NULL = 'Completed'
```

evaluates to UNKNOWN.

The WHERE clause removes those rows.

As a result, customers without completed orders disappear.

### **Correct approach**

```
SELECT
    c.customer_id,
    c.customer_name,
    o.order_id
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
   AND o.status = 'Completed';
```

The condition is part of the JOIN, so unmatched customers are preserved.

**Visual comparison**

Wrong

**LEFT JOIN + WHERE**

Unmatched rows are removed by the status filter.

Correct

**LEFT JOIN + ON**

All left-side customers are preserved.

### **Interview question: What's the difference between ON and WHERE?**

| ON                                                 | WHERE                                                    |
| -------------------------------------------------- | -------------------------------------------------------- |
| Defines matching conditions between tables         | Filters rows in the resulting relation                   |
| Determines which right-side rows match a LEFT JOIN | Can remove unmatched rows if its condition rejects NULLs |
| Executed as part of join processing                | Applied after the join in logical processing             |

**Important nuance:** The optimizer may push predicates between operations when it can preserve the result. The distinction above is about the logical meaning of your query.

# **5. UNION vs UNION ALL**

Suppose you want to combine customers from two cities.

### **UNION**

```
SELECT customer_name
FROM customers
WHERE city = 'Bhopal'

UNION

SELECT customer_name
FROM customers
WHERE city = 'Indore';
```

`UNION` removes duplicate result rows.

### **UNION ALL**

```
SELECT customer_name
FROM customers
WHERE city = 'Bhopal'

UNION ALL

SELECT customer_name
FROM customers
WHERE city = 'Indore';
```

`UNION ALL` retains duplicates.

| Feature                                       | UNION | UNION ALL |
| --------------------------------------------- | ----- | --------- |
| Combines result sets                          | Yes   | Yes       |
| Removes duplicate rows                        | Yes   | No        |
| Usually needs deduplication work              | Yes   | No        |
| Usually faster when duplicates are irrelevant | No    | Yes       |

### **Important interview detail**

The number and compatible types of columns must match.

Correct:

```
SELECT customer_id, customer_name
FROM customers

UNION ALL

SELECT emp_id, emp_name
FROM employees;
```

Incorrect:

```
SELECT customer_id, customer_name
FROM customers

UNION ALL

SELECT emp_id, emp_name, salary
FROM employees;
```

The second query has a different number of columns.

# **6. Interview checkpoint — test your understanding**

Try these without reading the explanations first.

**SQL semantics test**

6 questions

**Q1. Why can't you normally reference a SELECT alias in WHERE?**

WHERE executes after SELECT

WHERE logically executes before SELECT

Aliases are not allowed in SQL

WHERE only supports numeric columns

**Q2. A LEFT JOIN returns a customer with no order. What does COUNT(\*) return for that customer's group?**

0

1

NULL

Depends on salary

**Q3. Which is the safest common pattern for customers with no matching orders when the subquery might contain NULL?**

NOT IN

NOT EXISTS

<> NULL

\= NULL

**Q4. Where should an order status condition go when you must retain customers without completed orders?**

WHERE

HAVING

LEFT JOIN ON

ORDER BY

**Q5. What does UNION ALL do?**

Removes duplicates

Sorts both result sets

Combines results while retaining duplicates

Joins tables by key

**Q6. What does AVG(salary) do with NULL salaries?**

Treats them as zero

Counts them as negative

Ignores them

Returns NULL for the whole group

Check answers

## **Revision notes — memorise these six rules**

1. **Logical order:** FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT.
2. **WHERE vs HAVING:** Rows versus groups.
3. **NULL:** Use `IS NULL`, not `= NULL`; prefer `NOT EXISTS` to NULL-sensitive `NOT IN` patterns.
4. **COUNT:** `COUNT(*)` counts rows; `COUNT(column)` ignores NULL.
5. **LEFT JOIN:** Put right-table restrictions in `ON` when unmatched left-side rows must survive.
6. **UNION:** Removes duplicates; `UNION ALL` preserves them.

## **Next lesson — Part 2**

We'll cover **SQL performance and query optimization from first principles**:

- How a database parses, plans and executes SQL.
- Sequential scan vs index scan vs bitmap scan.
- B-Tree indexes and composite-index ordering.
- Why indexes sometimes make queries slower.
- `EXPLAIN` and `EXPLAIN ANALYZE`.
- Identifying expensive joins, sorts and aggregations.
- Optimizing slow SQL queries using practical PostgreSQL examples.

This is particularly important for SDE and backend interviews: writing a query correctly is only half the job; understanding how efficiently the database executes it matters too.

