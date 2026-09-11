# **Lesson 2 — Aggregate Functions, GROUP BY and HAVING**

High-priority for SQL interviews and OAs

This is one of the most important SQL topics. Most company OA questions involving employee salaries, departments, sales, customers and orders use these concepts.

By the end of this lesson, you should be able to solve:

- Find the highest, lowest and average salary.
- Calculate department-wise employee counts.
- Find departments with more than 5 employees.
- Find the second-highest salary.
- Identify duplicate records.
- Perform conditional counting and aggregation.

# **1. Aggregate functions**

Aggregate functions perform calculations on multiple rows and return a single value per group (or a single value for the entire result).

Consider our existing `employees` table:

| emp_id | name   | department_id | salary | city   |
| ------ | ------ | ------------- | ------ | ------ |
| 1      | Aman   | 101           | 60000  | Delhi  |
| 2      | Rahul  | 102           | 80000  | Mumbai |
| 3      | Priya  | 101           | 75000  | Delhi  |
| 4      | Neha   | 103           | 90000  | Pune   |
| 5      | Karan  | 102           | 50000  | Mumbai |
| 6      | Simran | 101           | 85000  | Delhi  |

There are five primary aggregate functions you should know.

| Function | Purpose              | Example             |
| -------- | -------------------- | ------------------- |
| COUNT()  | Count rows or values | Number of employees |
| SUM()    | Calculate total      | Total salaries      |
| AVG()    | Calculate average    | Average salary      |
| MAX()    | Find maximum         | Highest salary      |
| MIN()    | Find minimum         | Lowest salary       |

## **A. COUNT()**

Find the total number of employees:

```
SELECT COUNT(*)
FROM employees;
```

Output:

```
6
```

Find the number of employees with a recorded salary:

```
SELECT COUNT(salary)
FROM employees;
```

**Important interview distinction:**

| Expression             | Behaviour                     |
| ---------------------- | ----------------------------- |
| COUNT(\*)              | Counts all rows               |
| COUNT(column)          | Counts non-NULL values        |
| COUNT(DISTINCT column) | Counts unique non-NULL values |

For example:

```
SELECT COUNT(DISTINCT city)
FROM employees;
```

Output: `3`

Why? Because there are three unique cities: Delhi, Mumbai and Pune.

## **B. SUM()**

Calculate the total salary paid:

```
SELECT SUM(salary)
FROM employees;
```

Output:

```
440000
```

## **C. AVG()**

Calculate the average salary:

```
SELECT AVG(salary)
FROM employees;
```

Output:

```
73333.33
```

Note: AVG ignores NULL values.

## **D. MAX() and MIN()**

```
SELECT
    MAX(salary) AS highest_salary,
    MIN(salary) AS lowest_salary
FROM employees;
```

Output:

| highest_salary | lowest_salary |
| -------------- | ------------- |
| 90000          | 50000         |

## **E. Multiple aggregate functions together**

You do not need separate queries for every calculation.

```
SELECT
    COUNT(*) AS total_employees,
    SUM(salary) AS total_salary,
    AVG(salary) AS average_salary,
    MAX(salary) AS highest_salary,
    MIN(salary) AS lowest_salary
FROM employees;
```

This returns one row containing all five calculations.

# **2. GROUP BY — the most important concept**

Imagine an interviewer asks:

**Find the average salary of employees in each department.**

You cannot just use:

```
SELECT AVG(salary)
FROM employees;
```

That gives the average salary of the entire company.

You need to divide employees into groups according to their department.

That's exactly what `GROUP BY` does.

```
SELECT
    department_id,
    AVG(salary) AS average_salary
FROM employees
GROUP BY department_id;
```

Output:

| department_id | average_salary |
| ------------- | -------------- |
| 101           | 73333.33       |
| 102           | 65000          |
| 103           | 90000          |

**Understand what GROUP BY does**

Original employee records → grouped by department

**Department 101**

Aman — 60,000

Priya — 75,000

Simran — 85,000

AVG = 73,333

**Department 102**

Rahul — 80,000

Karan — 50,000

AVG = 65,000

**Department 103**

Neha — 90,000

AVG = 90,000

### **The golden rule of GROUP BY**

Every selected column that is not inside an aggregate function generally needs to be included in the GROUP BY clause.

Correct:

```
SELECT department_id, AVG(salary)
FROM employees
GROUP BY department_id;
```

Incorrect in standard SQL:

```
SELECT department_id, name, AVG(salary)
FROM employees
GROUP BY department_id;
```

Why?

Because department 101 has three different employee names. SQL does not know which name to display for the grouped row.

This is a common interview question.

## **3. GROUP BY with multiple columns**

Suppose you need average salary for each combination of department and city.

```
SELECT
    department_id,
    city,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id, city;
```

This produces separate groups for combinations like:

- Department 101 + Delhi
- Department 102 + Mumbai
- Department 103 + Pune

**Remember:** GROUP BY multiple columns creates groups based on unique combinations, not independent groups for each column.

# **4. HAVING — filter aggregated groups**

Now imagine this interview question:

**Find departments whose average salary is greater than ₹70,000.**

We already know how to calculate average salary using GROUP BY.

But how do we filter the groups?

Use `HAVING`.

```
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 70000;
```

Output:

| department_id | avg_salary |
| ------------- | ---------- |
| 101           | 73333.33   |
| 103           | 90000      |

Department 102 is excluded because its average is only 65,000.

## **WHERE vs HAVING — extremely important**

This is a frequently asked interview question.

| WHERE                                               | HAVING                                  |
| --------------------------------------------------- | --------------------------------------- |
| Filters individual rows                             | Filters groups                          |
| Applied before GROUP BY                             | Applied after GROUP BY                  |
| Cannot directly filter using aggregate calculations | Can filter using aggregate calculations |
| Works with individual records                       | Works with aggregated results           |

Consider this problem:

**Find departments where the average salary of employees earning more than ₹60,000 exceeds ₹70,000.**

You need both WHERE and HAVING.

```
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
WHERE salary > 60000
GROUP BY department_id
HAVING AVG(salary) > 70000;
```

Let's understand the execution:

1. WHERE removes employees earning ₹60,000 or less.
2. GROUP BY groups the remaining employees by department.
3. AVG calculates the average salary of each group.
4. HAVING removes groups whose average is not greater than ₹70,000.

This distinction is extremely important in OA questions.

# **5. Common OA problem patterns**

Now we move from learning syntax to solving problems.

## **Problem 1 — Count employees in each department**

```
SELECT
    department_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;
```

Expected output:

| department_id | employee_count |
| ------------- | -------------- |
| 101           | 3              |
| 102           | 2              |
| 103           | 1              |

**Pattern:** Whenever the question says "for each department", "per category", "per customer" or "per month", think about GROUP BY.

## **Problem 2 — Find departments having more than two employees**

```
SELECT
    department_id,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 2;
```

Output: Department 101.

## **Problem 3 — Find the highest salary in each department**

```
SELECT
    department_id,
    MAX(salary) AS highest_salary
FROM employees
GROUP BY department_id;
```

Output:

| department_id | highest_salary |
| ------------- | -------------- |
| 101           | 85000          |
| 102           | 80000          |
| 103           | 90000          |

Notice that this only gives the salary, not the employee's name. We'll solve that using JOINs, subqueries and window functions in upcoming lessons.

## **Problem 4 — Find departments with total salary expenditure greater than 100,000**

```
SELECT
    department_id,
    SUM(salary) AS total_salary
FROM employees
GROUP BY department_id
HAVING SUM(salary) > 100000;
```

Output:

| department_id | total_salary |
| ------------- | ------------ |
| 101           | 220000       |
| 102           | 130000       |

## **Problem 5 — Find cities having at least two employees**

```
SELECT
    city,
    COUNT(*) AS employee_count
FROM employees
GROUP BY city
HAVING COUNT(*) >= 2;
```

Output:

| city   | employee_count |
| ------ | -------------- |
| Delhi  | 3              |
| Mumbai | 2              |

# **6. Conditional aggregation — an important OA technique**

This is a particularly useful technique.

Suppose an interviewer provides a table of student results and asks:

- How many students passed?
- How many students failed?
- What is the total number of students?

You can solve all three in a single query using `CASE WHEN` inside aggregate functions.

Consider:

**students**

| student_id | name | marks |
| ---------- | ---- | ----- |
| 1          | A    | 85    |
| 2          | B    | 35    |
| 3          | C    | 65    |
| 4          | D    | 25    |
| 5          | E    | 90    |

Assume passing marks are 40.

```
SELECT
    COUNT(*) AS total_students,

    SUM(CASE
        WHEN marks >= 40 THEN 1
        ELSE 0
    END) AS passed,

    SUM(CASE
        WHEN marks < 40 THEN 1
        ELSE 0
    END) AS failed

FROM students;
```

Output:

| total_students | passed | failed |
| -------------- | ------ | ------ |
| 5              | 3      | 2      |

How does it work?

For each row, CASE generates either 1 or 0. SUM adds those values.

An alternative:

```
COUNT(CASE WHEN marks >= 40 THEN 1 END)
```

This counts only the rows for which the CASE expression returns a non-NULL value.

**OA pattern:** Look for phrases such as "count the number of customers who...", "number of orders that..." and "total sales for category X". Conditional aggregation may help.

# **7. Duplicate detection — frequently tested**

Suppose you have a table:

**users**

| user_id | email        |
| ------- | ------------ |
| 1       | a\@gmail.com |
| 2       | b\@gmail.com |
| 3       | a\@gmail.com |
| 4       | c\@gmail.com |
| 5       | b\@gmail.com |

Question: Find duplicate email addresses.

Solution:

```
SELECT
    email,
    COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

Output:

| email        | occurrences |
| ------------ | ----------- |
| a\@gmail.com | 2           |
| b\@gmail.com | 2           |

Why does it work?

- GROUP BY combines identical email values.
- COUNT counts how many times each appears.
- HAVING keeps only groups appearing more than once.

To find duplicate records in general, remember this pattern:

```
SELECT column_name, COUNT(*)
FROM table_name
GROUP BY column_name
HAVING COUNT(*) > 1;
```

# **8. Top interview traps**

These are worth remembering.

**Trap 1: COUNT(\*) vs COUNT(column)**

If a table has 10 rows and 3 NULL values in `salary`:

```
SELECT COUNT(*) FROM employees;
-- 10

SELECT COUNT(salary) FROM employees;
-- 7
```

**Trap 2: WHERE cannot filter aggregate functions directly**

Incorrect:

```
SELECT department_id, AVG(salary)
FROM employees
WHERE AVG(salary) > 70000
GROUP BY department_id;
```

Correct:

```
SELECT department_id, AVG(salary)
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 70000;
```

**Trap 3: COUNT(DISTINCT column)**

```
SELECT COUNT(DISTINCT city)
FROM employees;
```

Counts unique non-NULL cities.

**Trap 4: GROUP BY does not automatically sort the result.**

```
SELECT department_id, COUNT(*)
FROM employees
GROUP BY department_id
ORDER BY COUNT(*) DESC;
```

Always use ORDER BY when a specific ordering is required.

**Trap 5: Filtering before grouping changes the calculation.**

These two queries can return different results:

```
-- Average among employees earning more than 60,000
SELECT department_id, AVG(salary)
FROM employees
WHERE salary > 60000
GROUP BY department_id;
```

```
-- Average among all employees, retaining departments
-- whose overall average exceeds 60,000
SELECT department_id, AVG(salary)
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 60000;
```

The WHERE condition removes individual rows; HAVING removes completed groups.

# **9. Practice: SQL interview and OA assessment**

Attempt the following without looking at the solutions.

**Assessment 2**

5 questions

**Q1. Which clause filters groups after aggregation?**

WHERE

HAVING

ORDER BY

DISTINCT

**Q2. A table has 10 rows, and 3 values in the salary column are NULL. What does COUNT(salary) return?**

10

3

7

NULL

**Q3. Which query finds departments with at least 5 employees?**

WHERE COUNT(\*) >= 5

GROUP BY department_id HAVING COUNT(\*) >= 5

GROUP BY department_id WHERE COUNT(\*) >= 5

HAVING department_id >= 5

**Q4. What does COUNT(DISTINCT city) calculate?**

All city entries

Only NULL cities

Number of unique non-NULL cities

Number of duplicate cities

**Q5. Which clause executes first logically?**

SELECT

HAVING

WHERE

FROM

Submit answers

## **10. Query-writing assignment**

These are more important than MCQs. Write the actual SQL queries.

Use our `employees` table.

| # | Problem                                                                                                      | Difficulty |
| - | ------------------------------------------------------------------------------------------------------------ | ---------- |
| 1 | Find the total number of employees in each department.                                                       | Easy       |
| 2 | Find the average salary in each city.                                                                        | Easy       |
| 3 | Find departments having more than one employee.                                                              | Easy       |
| 4 | Find departments whose maximum salary exceeds 80,000.                                                        | Medium     |
| 5 | Find the department with the highest average salary.                                                         | Medium     |
| 6 | Find cities having an average salary greater than 70,000.                                                    | Medium     |
| 7 | Find the total salary of employees in each department, but consider only employees earning more than 60,000. | Medium     |
| 8 | Find the department-wise salary range (maximum salary minus minimum salary).                                 | Medium     |

**Important:** Do not just memorise these queries. Understand how to translate an English problem into SQL clauses.

# **11. Interview revision notes**

| Concept                | Remember                   |
| ---------------------- | -------------------------- |
| COUNT(\*)              | Counts all rows            |
| COUNT(column)          | Ignores NULL               |
| COUNT(DISTINCT column) | Unique non-NULL values     |
| SUM()                  | Total                      |
| AVG()                  | Average, ignores NULL      |
| MAX() / MIN()          | Highest / lowest           |
| GROUP BY               | Creates groups             |
| HAVING                 | Filters groups             |
| WHERE                  | Filters individual records |
| CASE WHEN              | Conditional logic          |
| ORDER BY               | Sorts results              |

## **What comes next?**

**Lesson 3 — SQL JOINs**

Critical

[SQL JOINS PRACTICE. Here are some notes and question… | by Tugce | Medium](https://medium.com/%40tugcecirak44/sql-joins-practice-7bd8718650e0)

We will cover:

1. INNER JOIN
2. LEFT JOIN
3. RIGHT JOIN
4. FULL OUTER JOIN
5. CROSS JOIN
6. SELF JOIN
7. Joining three or more tables
8. Finding employees without departments
9. Detecting unmatched records
10. Real OA problems involving multiple tables.

**My recommendation:** Give special attention to JOINs, subqueries and window functions. Together, these form a large part of practical SQL interview and OA problem-solving.