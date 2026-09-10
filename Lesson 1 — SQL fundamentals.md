# **SQL Mastery: Interviews + Online Assessments**

Our target is not just memorising SQL commands. You should be able to take an unfamiliar problem, understand its requirements, construct the query, optimise it, and explain why it works.

We'll use **PostgreSQL-compatible SQL** throughout. Most concepts also work in MySQL, SQL Server, and Oracle, with some syntax differences.

## **1. Complete learning roadmap**

**Phase 1 — SQL fundamentals**

Start here

1. Database, tables, rows, columns, keys and data types
2. SELECT, DISTINCT, WHERE, ORDER BY, LIMIT
3. Operators, NULL, CASE WHEN
4. Aggregate functions and GROUP BY
5. HAVING and filtering grouped data

**Phase 2 — Relational queries**

- INNER, LEFT, RIGHT and FULL JOIN
- Self joins and multiple joins
- UNION, UNION ALL and set operations
- Subqueries and correlated subqueries

  **Phase 3 — Advanced SQL for OAs**
- CTEs and recursive CTEs
- Window functions: ROW_NUMBER, RANK, DENSE_RANK
- LEAD, LAG, running totals
- Top N per group
- Second-highest salary and duplicate detection
- Date, string and conditional problems

  **Phase 4 — Interview-level database concepts**
- Primary keys, foreign keys and constraints
- Normalisation (1NF, 2NF, 3NF, BCNF)
- Indexes and query optimisation
- ACID properties and transactions
- Isolation levels, locks and deadlocks
- Views, stored procedures and triggers

  **Phase 5 — Interview and OA practice**
- Easy, medium and hard SQL problems
- LeetCode SQL 50
- Company-style OA questions
- Query debugging and optimisation
- Mock SQL interviews

# **Lesson 1 — SQL fundamentals**

## **2. Understand the database first**

Imagine a company maintains employee information.

**Table: employees**

| emp_id | name   | department_id | salary | city   |
| ------ | ------ | ------------- | ------ | ------ |
| 1      | Aman   | 101           | 60000  | Delhi  |
| 2      | Rahul  | 102           | 80000  | Mumbai |
| 3      | Priya  | 101           | 75000  | Delhi  |
| 4      | Neha   | 103           | 90000  | Pune   |
| 5      | Karan  | 102           | 50000  | Mumbai |
| 6      | Simran | 101           | 85000  | Delhi  |

**Table: departments**

| department_id | department_name |
| ------------- | --------------- |
| 101           | Engineering     |
| 102           | HR              |
| 103           | Finance         |

Basic terminology:

| Term        | Meaning                              |
| ----------- | ------------------------------------ |
| Database    | Collection of related data           |
| Table       | Data organised into rows and columns |
| Row         | One record                           |
| Column      | One attribute                        |
| Primary Key | Uniquely identifies a row            |
| Foreign Key | References a key in another table    |
| Schema      | Structure of the database            |

Here, `emp_id` is the primary key of `employees`. `department_id` is a foreign key referencing `departments.department_id`.

## **3. SELECT — retrieve data**

The most fundamental SQL statement.

```
SELECT * 
FROM employees;
```

`*` means all columns.

To retrieve specific columns:

```
SELECT name, salary
FROM employees;
```

Result:

| name   | salary |
| ------ | ------ |
| Aman   | 60000  |
| Rahul  | 80000  |
| Priya  | 75000  |
| Neha   | 90000  |
| Karan  | 50000  |
| Simran | 85000  |

**Interview question: What is the difference between SELECT \* and SELECT column_name?**

SELECT \* retrieves every column, whereas selecting specific columns retrieves only the required attributes. Explicit column selection improves readability and can reduce unnecessary data transfer.

## **4. WHERE — filter rows**

Suppose you need employees earning more than ₹70,000.

```
SELECT *
FROM employees
WHERE salary > 70000;
```

Result:

| name   | salary |
| ------ | ------ |
| Rahul  | 80000  |
| Priya  | 75000  |
| Neha   | 90000  |
| Simran | 85000  |

Important operators:

| Operator     | Meaning                     |
| ------------ | --------------------------- |
| `=`          | Equal                       |
| `!=` or `<>` | Not equal                   |
| `>`          | Greater than                |
| `<`          | Less than                   |
| `>=`         | Greater than or equal       |
| `<=`         | Less than or equal          |
| `BETWEEN`    | Within a range, inclusive   |
| `IN`         | Matches any value in a list |
| `LIKE`       | Pattern matching            |
| `IS NULL`    | Checks for NULL             |

### **Practical examples**

**A. Salary between 60,000 and 90,000**

```
SELECT *
FROM employees
WHERE salary BETWEEN 60000 AND 90000;
```

`BETWEEN` includes both endpoints.

**B. Employees from Delhi or Mumbai**

```
SELECT *
FROM employees
WHERE city IN ('Delhi', 'Mumbai');
```

**C. Employees not from Delhi**

```
SELECT *
FROM employees
WHERE city <> 'Delhi';
```

**D. Names starting with A**

```
SELECT *
FROM employees
WHERE name LIKE 'A%';
```

Pattern matching:

| Pattern  | Meaning               |
| -------- | --------------------- |
| `'A%'`   | Starts with A         |
| `'%a'`   | Ends with a           |
| `'%ar%'` | Contains ar           |
| `'_a%'`  | Second character is a |

`%` represents zero or more characters. `_` represents exactly one character.

## **5. AND, OR and NOT**

These operators combine conditions.

**Employees from Delhi earning more than 70,000:**

```
SELECT *
FROM employees
WHERE city = 'Delhi'
AND salary > 70000;
```

**Employees from Delhi OR earning more than 80,000:**

```
SELECT *
FROM employees
WHERE city = 'Delhi'
OR salary > 80000;
```

Important: SQL evaluates `AND` before `OR`, unless parentheses change the order.

For example:

```
SELECT *
FROM employees
WHERE city = 'Delhi'
   OR city = 'Mumbai'
  AND salary > 70000;
```

This is interpreted as:

```
WHERE city = 'Delhi'
   OR (city = 'Mumbai' AND salary > 70000);
```

For clarity, explicitly use parentheses.

## **6. DISTINCT — remove duplicates**

Find all unique cities:

```
SELECT DISTINCT city
FROM employees;
```

Result:

| city   |
| ------ |
| Delhi  |
| Mumbai |
| Pune   |

Multiple-column DISTINCT:

```
SELECT DISTINCT city, department_id
FROM employees;
```

This returns unique combinations of city and department, not individually unique values in each column.

## **7. ORDER BY — sorting**

Find employees in descending order of salary:

```
SELECT name, salary
FROM employees
ORDER BY salary DESC;
```

Ascending order:

```
ORDER BY salary ASC;
```

`ASC` is the default.

Multiple-column ordering:

```
SELECT *
FROM employees
ORDER BY department_id ASC, salary DESC;
```

This first sorts by department and then sorts salaries within each department.

## **8. LIMIT and OFFSET**

Find the three highest-paid employees:

```
SELECT name, salary
FROM employees
ORDER BY salary DESC
LIMIT 3;
```

Result:

| name   | salary |
| ------ | ------ |
| Neha   | 90000  |
| Simran | 85000  |
| Rahul  | 80000  |

Skip the first two rows and retrieve the next two:

```
SELECT *
FROM employees
ORDER BY salary DESC
LIMIT 2 OFFSET 2;
```

This is useful for pagination.

**Important OA pattern:** Always sort before applying LIMIT when the problem asks for highest, lowest, latest or earliest records.

## **9. NULL — one of the most important interview concepts**

NULL means a missing or unknown value. It is not equivalent to zero, an empty string or FALSE.

Suppose an employee has no recorded salary.

Incorrect:

```
SELECT *
FROM employees
WHERE salary = NULL;
```

Correct:

```
SELECT *
FROM employees
WHERE salary IS NULL;
```

To find non-NULL salaries:

```
SELECT *
FROM employees
WHERE salary IS NOT NULL;
```

Why is this important?

SQL uses three-valued logic:

- TRUE
- FALSE
- UNKNOWN

A comparison such as `NULL = NULL` does not evaluate to TRUE. It evaluates to UNKNOWN.

Therefore, ordinary equality comparisons do not work for NULL.

## **10. CASE WHEN — conditional logic**

Suppose we want to classify employees by salary.

```
SELECT
    name,
    salary,
    CASE
        WHEN salary >= 80000 THEN 'High'
        WHEN salary >= 60000 THEN 'Medium'
        ELSE 'Low'
    END AS salary_category
FROM employees;
```

Result:

| name   | salary | salary_category |
| ------ | ------ | --------------- |
| Aman   | 60000  | Medium          |
| Rahul  | 80000  | High            |
| Priya  | 75000  | Medium          |
| Neha   | 90000  | High            |
| Karan  | 50000  | Low             |
| Simran | 85000  | High            |

The conditions are evaluated from top to bottom. The first matching condition is selected.

This is especially useful in OA questions involving categories, conditional counts and conditional aggregation.

# **11. Extremely important: SQL logical execution order**

This is one of the most useful concepts for interviews.

You write:

```
SELECT department_id, AVG(salary)
FROM employees
WHERE salary > 50000
GROUP BY department_id
HAVING AVG(salary) > 70000
ORDER BY AVG(salary) DESC
LIMIT 2;
```

But SQL conceptually processes the query in this order:

## **SQL logical execution order**

**1**

**FROM**

Identify the source table

**2**

**WHERE**

Filter individual rows

**3**

**GROUP BY**

Create groups

**4**

**HAVING**

Filter groups

**5**

**SELECT**

Calculate and select expressions

**6**

**DISTINCT**

Remove duplicate result rows

**7**

**ORDER BY**

Sort the result

**8**

**LIMIT / OFFSET**

Restrict returned rows

The logical processing order is different from the order in which you write the SQL query.

This explains why you generally cannot use a SELECT alias inside WHERE: WHERE is logically processed before SELECT.

For interviews, remember this order. It will help you debug queries and understand aggregation.

# **12. Your first OA-style problems**

Try these using the employee table above. Think like you're solving an actual online assessment.

**Fundamentals checkpoint**

4 questions

**Q1. Find employees whose salary is greater than 75,000.**

WHERE salary > 75000

HAVING salary > 75000

WHERE salary >= 75000

ORDER BY salary > 75000

**Q2. Find the second-highest salary using ORDER BY and LIMIT.**

ORDER BY salary DESC LIMIT 1

ORDER BY salary DESC LIMIT 1 OFFSET 1

ORDER BY salary ASC LIMIT 1 OFFSET 1

ORDER BY salary DESC OFFSET 2

**Q3. Which condition correctly selects rows with a missing city?**

city = NULL

city == NULL

city IS NULL

city != NULL

**Q4. Find all employees from Delhi or Mumbai.**

city = 'Delhi' AND city = 'Mumbai'

city IN ('Delhi', 'Mumbai')

city BETWEEN 'Delhi' AND 'Mumbai'

city LIKE 'Delhi, Mumbai'

Check answers

## **13. Interview questions you should be able to answer**

Before moving on, make sure you can explain these without memorising definitions:

1. What is the difference between WHERE and HAVING?
2. What is the difference between DELETE, TRUNCATE and DROP?
3. What is the difference between PRIMARY KEY and UNIQUE?
4. What is the difference between NULL and 0?
5. What is the difference between ORDER BY and GROUP BY?
6. What happens when you use LIMIT without ORDER BY?
7. Why can't we use aggregate functions directly inside WHERE?
8. What is the logical execution order of a SQL query?

## **14. Your first practical assignment**

Write SQL queries for these five tasks using the employee dataset:

| # | Problem                                              | Concept                  |
| - | ---------------------------------------------------- | ------------------------ |
| 1 | Find the highest-paid employee                       | ORDER BY + LIMIT         |
| 2 | Find employees earning between 60,000 and 85,000     | BETWEEN                  |
| 3 | Find employees whose names start with S              | LIKE                     |
| 4 | Find the unique departments represented in employees | DISTINCT                 |
| 5 | Find the top two highest-paid employees from Delhi   | WHERE + ORDER BY + LIMIT |

Try writing the queries yourself rather than looking up the syntax.

**Next lesson: Aggregate functions, GROUP BY and HAVING.** These are among the most frequently tested SQL concepts in interviews and OAs, and they form the foundation for questions like department-wise highest salary, average salary, duplicate groups, and conditional counting.