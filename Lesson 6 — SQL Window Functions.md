# **Lesson 6 — SQL Window Functions**



Extremely important for interviews and OAs

Intermediate → Advanced



Window functions are among the most powerful SQL features. They are particularly important for solving ranking, salary, sales, customer behaviour and time-series problems.

You should become very comfortable with these functions because they appear frequently in data engineering, analytics and software engineering SQL assessments.

By the end of this lesson, you'll be able to solve:

1. Second-highest and Nth-highest salary.
2. Top 3 employees in each department.
3. Rank employees based on salary.
4. Find duplicate records.
5. Calculate running totals and cumulative averages.
6. Compare current and previous rows.
7. Find the first and last transaction.
8. Calculate month-over-month growth.
9. Identify consecutive records and gaps in sequences.

We'll use PostgreSQL-compatible SQL.

# **1. First understand the problem: GROUP BY vs Window Functions**

This distinction is fundamental.

Suppose we have the following employee table:

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

Suppose you need to calculate the average salary of each department.

### **Approach 1: GROUP BY**

```
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;
```

Output:

| department_id | avg_salary |
| ------------- | ---------- |
| 101           | 73333.33   |
| 102           | 65000      |
| 103           | 90000      |
| 104           | 70000      |

Notice that we have only four rows.

GROUP BY combines multiple employee records into department-level records.

### **Approach 2: Window function**

```
SELECT
    emp_id,
    name,
    department_id,
    salary,
    AVG(salary) OVER (
        PARTITION BY department_id
    ) AS department_avg
FROM employees;
```

Output:

| name   | department_id | salary | department_avg |
| ------ | ------------- | ------ | -------------- |
| Aman   | 101           | 60000  | 73333.33       |
| Priya  | 101           | 75000  | 73333.33       |
| Simran | 101           | 85000  | 73333.33       |
| Rahul  | 102           | 80000  | 65000          |
| Karan  | 102           | 50000  | 65000          |
| Neha   | 103           | 90000  | 90000          |
| Riya   | 104           | 70000  | 70000          |

Notice the difference.

**GROUP BY vs Window Functions**

**GROUP BY**

# **4 rows**

Combines employees into department-level results.

**Window function**

# **7 rows**

Preserves all employee records and adds department-level calculations to each row.

**Golden rule:**

- GROUP BY reduces rows.
- Window functions generally preserve rows while calculating values across related rows.

This is the central idea behind window functions.

# **2. Understanding OVER()**

Every window function uses the `OVER()` clause.

Basic syntax:

```
function_name() OVER (
    PARTITION BY column_name
    ORDER BY column_name
);
```

There are three important components.

| Component    | Purpose                                 |
| ------------ | --------------------------------------- |
| Function     | ROW_NUMBER, RANK, AVG, SUM, etc.        |
| PARTITION BY | Divides rows into independent groups    |
| ORDER BY     | Defines the ordering within each window |

Let's understand them.

### **A. OVER()**

```
SELECT
    name,
    salary,
    AVG(salary) OVER () AS company_average
FROM employees;
```

There is no PARTITION BY.

Therefore, all employees belong to one window.

Every employee gets the company's average salary.

### **B. PARTITION BY**

```
SELECT
    name,
    department_id,
    salary,
    AVG(salary) OVER (
        PARTITION BY department_id
    ) AS department_average
FROM employees;
```

This creates a separate logical window for each department.

Think of it as dividing the data into groups without collapsing the rows.

### **C. ORDER BY inside OVER()**

```
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (
        ORDER BY salary DESC
    ) AS salary_position
FROM employees;
```

This assigns positions according to salary, highest first.

**Important distinction:**

`ORDER BY` inside `OVER()` controls the window calculation. It does not guarantee the final output order.

Use another ORDER BY at the end of the query when you need sorted output.

# **3. ROW_NUMBER()**

Must master

`ROW_NUMBER()` assigns a unique sequential number to each row in a window.

Syntax:

```
ROW_NUMBER() OVER (
    ORDER BY column_name
);
```

Example:

```
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (
        ORDER BY salary DESC
    ) AS row_num
FROM employees;
```

Output:

| name   | salary | row_num |
| ------ | ------ | ------- |
| Neha   | 90000  | 1       |
| Simran | 85000  | 2       |
| Rahul  | 80000  | 3       |
| Priya  | 75000  | 4       |
| Riya   | 70000  | 5       |
| Aman   | 60000  | 6       |
| Karan  | 50000  | 7       |

Every row gets a unique number.

### **ROW_NUMBER with PARTITION BY**

Now suppose the question is:

**Assign a salary position to every employee within their own department.**

```
SELECT
    name,
    department_id,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS department_position
FROM employees;
```

Output:

| name   | department_id | salary | department_position |
| ------ | ------------- | ------ | ------------------- |
| Simran | 101           | 85000  | 1                   |
| Priya  | 101           | 75000  | 2                   |
| Aman   | 101           | 60000  | 3                   |
| Rahul  | 102           | 80000  | 1                   |
| Karan  | 102           | 50000  | 2                   |
| Neha   | 103           | 90000  | 1                   |
| Riya   | 104           | 70000  | 1                   |

Notice that the row numbering resets for every department.

This is a fundamental OA pattern.

# **4. RANK() vs DENSE_RANK() vs ROW_NUMBER()**

Extremely important interview question

These three functions are frequently asked together.

Let's use a different example to understand ties.

**employee_scores**

| name   | score |
| ------ | ----- |
| Aman   | 100   |
| Rahul  | 100   |
| Priya  | 90    |
| Neha   | 80    |
| Karan  | 80    |
| Simran | 70    |

Query:

```
SELECT
    name,
    score,

    ROW_NUMBER() OVER (
        ORDER BY score DESC, name
    ) AS row_num,

    RANK() OVER (
        ORDER BY score DESC
    ) AS rank_num,

    DENSE_RANK() OVER (
        ORDER BY score DESC
    ) AS dense_rank_num

FROM employee_scores;
```

Output:

| name   | score | ROW_NUMBER | RANK | DENSE_RANK |
| ------ | ----- | ---------- | ---- | ---------- |
| Aman   | 100   | 1          | 1    | 1          |
| Rahul  | 100   | 2          | 1    | 1          |
| Priya  | 90    | 3          | 3    | 2          |
| Neha   | 80    | 4          | 4    | 3          |
| Karan  | 80    | 5          | 4    | 3          |
| Simran | 70    | 6          | 6    | 4          |

Understand the differences:

**ROW_NUMBER()**

# **1, 2, 3, 4, 5, 6**

Every row receives a unique number, even when values tie.

**RANK()**

# **1, 1, 3, 4, 4, 6**

Equal values receive the same rank. The next rank is skipped.

**DENSE_RANK()**

# **1, 1, 2, 3, 3, 4**

Equal values receive the same rank. There are no gaps.

### **Remember using a real-life analogy**

Imagine a competition.

- ROW_NUMBER: Every participant gets a different position.
- RANK: Tied participants share a position, and the next position is skipped.
- DENSE_RANK: Tied participants share a position, but no position is skipped.

### **Interview question**

**Which function should you use to find the second-highest distinct salary?**

Answer: `DENSE_RANK()`.

Why?

Because we want to rank distinct salary levels, not individual employee rows.

# **5. Find the Nth-highest salary using window functions**

Let's solve one of the most frequently asked interview questions.

**Find the second-highest distinct salary.**

```
WITH ranked_salaries AS (
    SELECT
        salary,
        DENSE_RANK() OVER (
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees
)

SELECT DISTINCT salary
FROM ranked_salaries
WHERE salary_rank = 2;
```

Result:

```
85000
```

### **Find the third-highest distinct salary**

Just change the condition:

```
WHERE salary_rank = 3;
```

### **Find the Nth-highest distinct salary**

```
WITH ranked_salaries AS (
    SELECT
        salary,
        DENSE_RANK() OVER (
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees
)

SELECT DISTINCT salary
FROM ranked_salaries
WHERE salary_rank = N;
```

Replace `N` with the required rank.

### **What if the interviewer asks for the second employee by salary rather than the second distinct salary?**

Use `ROW_NUMBER()` instead.

This distinction matters enormously.

# **6. Top N employees in every department**

Very common OA pattern

This is one of the most important applications of window functions.

**Problem: Find the top two highest-paid employees in every department.**

A common mistake is using:

```
SELECT *
FROM employees
ORDER BY salary DESC
LIMIT 2;
```

This gives the top two employees across the entire company, not two per department.

We need PARTITION BY.

### **Solution using ROW_NUMBER()**

```
WITH ranked_employees AS (
    SELECT
        emp_id,
        name,
        department_id,
        salary,

        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS rn

    FROM employees
)

SELECT *
FROM ranked_employees
WHERE rn <= 2;
```

Output:

| name   | department_id | salary | rn |
| ------ | ------------- | ------ | -- |
| Simran | 101           | 85000  | 1  |
| Priya  | 101           | 75000  | 2  |
| Rahul  | 102           | 80000  | 1  |
| Karan  | 102           | 50000  | 2  |
| Neha   | 103           | 90000  | 1  |
| Riya   | 104           | 70000  | 1  |

Notice that the row number restarts for each department.

### **Why are we using a CTE?**

You cannot directly write:

```
SELECT
    name,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees
WHERE rn <= 2;
```

This does not work in PostgreSQL because WHERE is logically processed before window functions are calculated.

Therefore, calculate the rank first using a CTE or subquery, then filter it in the outer query.

### **Important variation: What if salaries are tied?**

Suppose Department 101 has:

| Employee | Salary |
| -------- | ------ |
| Aman     | 90000  |
| Rahul    | 85000  |
| Priya    | 85000  |
| Neha     | 75000  |

The correct function depends on the wording.

**Case 1: Return exactly the top two employees.**

Use ROW_NUMBER.

```
ROW_NUMBER() OVER (
    PARTITION BY department_id
    ORDER BY salary DESC, emp_id
)
```

This returns exactly two employees, assuming the department has at least two employees.

**Case 2: Return employees in the top two distinct salary positions, including ties.**

Use DENSE_RANK.

```
DENSE_RANK() OVER (
    PARTITION BY department_id
    ORDER BY salary DESC
)
```

Then:

```
WHERE salary_rank <= 2;
```

This returns Aman, Rahul and Priya.

**Case 3: Return employees whose competition ranking is at most 2.**

Use RANK.

This also returns Aman, Rahul and Priya in the example, but unlike DENSE_RANK, subsequent rank positions can have gaps.

**OA decision guide**

| Requirement                           | Function   |
| ------------------------------------- | ---------- |
| Exactly N rows per group              | ROW_NUMBER |
| Top N distinct values, including ties | DENSE_RANK |
| Competition ranking with gaps         | RANK       |

# **7. Aggregate functions as window functions**

One of the most useful features is that aggregate functions can also work as window functions.

You already know:

```
SELECT AVG(salary)
FROM employees;
```

Now:

```
SELECT
    name,
    salary,

    COUNT(*) OVER() AS total_employees,

    SUM(salary) OVER() AS total_salary,

    AVG(salary) OVER() AS company_avg,

    MAX(salary) OVER() AS maximum_salary,

    MIN(salary) OVER() AS minimum_salary

FROM employees;
```

Every employee retains their original row, but receives the company-wide calculations.

You can also partition them:

```
SELECT
    name,
    department_id,
    salary,

    COUNT(*) OVER (
        PARTITION BY department_id
    ) AS department_count,

    SUM(salary) OVER (
        PARTITION BY department_id
    ) AS department_total_salary

FROM employees;
```

This is especially helpful when you need employee-level information alongside department-level statistics.

# **8. Running totals and cumulative sums**

Frequently tested

This is an important SQL pattern in financial applications, sales analysis and data engineering.

Consider a sales table:

**daily_sales**

| sale_date  | amount |
| ---------- | ------ |
| 2026-01-01 | 100    |
| 2026-01-02 | 200    |
| 2026-01-03 | 150    |
| 2026-01-04 | 300    |
| 2026-01-05 | 250    |

Question:

**Calculate the cumulative sales amount for each date.**

Expected output:

| sale_date | amount | cumulative_sales |
| --------- | ------ | ---------------- |
| Jan 1     | 100    | 100              |
| Jan 2     | 200    | 300              |
| Jan 3     | 150    | 450              |
| Jan 4     | 300    | 750              |
| Jan 5     | 250    | 1000             |

Solution:

```
SELECT
    sale_date,
    amount,

    SUM(amount) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND CURRENT ROW
    ) AS cumulative_sales

FROM daily_sales;
```

Understand the syntax:

```
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

This means:

- Start from the first row of the window.
- Continue through the current row.
- Calculate the sum for all rows in that frame.

**How a running total develops**

| **Day** | **Calculation** | **Total** |
| ------- | --------------- | --------- |
| Day 1   | 100             | **100**   |
| Day 2   | 100 + 200       | **300**   |
| Day 3   | 100 + 200 + 150 | **450**   |
| Day 4   | 450 + 300       | **750**   |
| Day 5   | 750 + 250       | **1,000** |

## **9. Understand window frames**

The `ROWS BETWEEN` clause deserves special attention.

Common frame definitions:

| Frame               | Meaning                         |
| ------------------- | ------------------------------- |
| UNBOUNDED PRECEDING | Starting from the first row     |
| CURRENT ROW         | The current row                 |
| 1 PRECEDING         | One row before the current row  |
| 2 PRECEDING         | Two rows before the current row |
| 1 FOLLOWING         | One row after the current row   |
| UNBOUNDED FOLLOWING | The last row in the window      |

### **A. Running total**

```
SUM(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING
             AND CURRENT ROW
)
```

### **B. Moving sum of the last three rows**

```
SUM(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 2 PRECEDING
             AND CURRENT ROW
)
```

Output:

| Date  | Amount | Three-row moving sum |
| ----- | ------ | -------------------- |
| Jan 1 | 100    | 100                  |
| Jan 2 | 200    | 300                  |
| Jan 3 | 150    | 450                  |
| Jan 4 | 300    | 650                  |
| Jan 5 | 250    | 700                  |

At Jan 4, we add Jan 2, Jan 3 and Jan 4.

### **C. Moving average of the last three rows**

```
AVG(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 2 PRECEDING
             AND CURRENT ROW
)
```

This calculates a moving average.

**Important:** ROWS means physical row positions, not calendar days. For a true rolling seven-calendar-day calculation, the query must account for dates and missing days, often using a suitable RANGE frame or a calendar table.

### **ROWS vs RANGE**

Another interview concept:

- ROWS uses physical row positions.
- RANGE considers ordering values and peer rows with equal ordering values.

When you need deterministic row-by-row cumulative calculations, explicitly specify a ROWS frame and a unique ordering key.

Also, with an ORDER BY inside an aggregate window, PostgreSQL's default frame is peer-aware. Specifying the frame explicitly avoids unexpected results when ordering values tie.

# **10. LAG() — access previous rows**

Very important for data analytics OAs

`LAG()` allows us to access a value from a previous row without performing a self-join.

Syntax:

```
LAG(column_name, offset, default_value)
OVER (
    ORDER BY column_name
);
```

Parameters:

| Parameter     | Meaning                                     |
| ------------- | ------------------------------------------- |
| column_name   | Column whose previous value is needed       |
| offset        | How many rows backward                      |
| default_value | Value returned when no preceding row exists |

Consider our daily sales table.

**Question: Display each day's sales alongside the previous day's sales.**

```
SELECT
    sale_date,
    amount,

    LAG(amount, 1) OVER (
        ORDER BY sale_date
    ) AS previous_day_sales

FROM daily_sales;
```

Output:

| sale_date | amount | previous_day_sales |
| --------- | ------ | ------------------ |
| Jan 1     | 100    | NULL               |
| Jan 2     | 200    | 100                |
| Jan 3     | 150    | 200                |
| Jan 4     | 300    | 150                |
| Jan 5     | 250    | 300                |

The first row has no previous value, so it returns NULL.

### **Find the difference from the previous day**

```
SELECT
    sale_date,
    amount,

    amount - LAG(amount) OVER (
        ORDER BY sale_date
    ) AS sales_difference

FROM daily_sales;
```

Output:

| Date  | Amount | Difference |
| ----- | ------ | ---------- |
| Jan 1 | 100    | NULL       |
| Jan 2 | 200    | 100        |
| Jan 3 | 150    | -50        |
| Jan 4 | 300    | 150        |
| Jan 5 | 250    | -50        |

### **Calculate percentage growth**

```
WITH sales_comparison AS (
    SELECT
        sale_date,
        amount,

        LAG(amount) OVER (
            ORDER BY sale_date
        ) AS previous_amount

    FROM daily_sales
)

SELECT
    sale_date,
    amount,
    previous_amount,

    ROUND(
        100.0 * (amount - previous_amount)
        / NULLIF(previous_amount, 0),
        2
    ) AS growth_percentage

FROM sales_comparison;
```

Formula:

\\[ \text{Growth \\%} = \frac{\text{Current} - \text{Previous}} {\text{Previous}} \times 100 \\]

`NULLIF(previous_amount, 0)` prevents division-by-zero errors.

# **11. LEAD() — access future rows**

`LEAD()` is essentially the opposite of LAG().

- LAG accesses previous rows.
- LEAD accesses subsequent rows.

Example:

```
SELECT
    sale_date,
    amount,

    LEAD(amount, 1) OVER (
        ORDER BY sale_date
    ) AS next_day_sales

FROM daily_sales;
```

Output:

| sale_date | amount | next_day_sales |
| --------- | ------ | -------------- |
| Jan 1     | 100    | 200            |
| Jan 2     | 200    | 150            |
| Jan 3     | 150    | 300            |
| Jan 4     | 300    | 250            |
| Jan 5     | 250    | NULL           |

### **LAG vs LEAD**

| Function        | Direction         |
| --------------- | ----------------- |
| LAG(column, 1)  | Previous row      |
| LEAD(column, 1) | Next row          |
| LAG(column, 2)  | Two rows backward |
| LEAD(column, 2) | Two rows forward  |

### **Common interview application**

Find employees whose salary is greater than the salary of the employee immediately before them in a specified ordering.

```
WITH comparison AS (
    SELECT
        name,
        salary,
        LAG(salary) OVER (
            ORDER BY emp_id
        ) AS previous_salary
    FROM employees
)

SELECT *
FROM comparison
WHERE salary > previous_salary;
```

Notice that the meaning of previous depends entirely on the ORDER BY column. Here, it means the preceding employee ID, not the preceding salary.

# **12. FIRST_VALUE() and LAST_VALUE()**

These functions retrieve the first and last values in a window.

### **A. FIRST_VALUE()**

Find the highest salary in each department alongside every employee.

```
SELECT
    name,
    department_id,
    salary,

    FIRST_VALUE(salary) OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
    ) AS highest_salary

FROM employees;
```

Since we sort by salary descending, the first value is the highest salary.

### **B. LAST_VALUE()**

Find the lowest salary in each department.

```
SELECT
    name,
    department_id,
    salary,

    LAST_VALUE(salary) OVER (
        PARTITION BY department_id
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND UNBOUNDED FOLLOWING
    ) AS lowest_salary

FROM employees;
```

Why do we explicitly define the frame?

This is a very important detail.

With ORDER BY, the default frame usually ends at the current row and its peers. Consequently, LAST_VALUE may return the current row's value instead of the last value in the complete department.

By specifying:

```
ROWS BETWEEN UNBOUNDED PRECEDING
         AND UNBOUNDED FOLLOWING
```

we include the entire partition.

Now the last value under descending salary order is the lowest salary.

**Memory trick**

**FIRST_VALUE**

First row in frame

**LAST_VALUE**

Last row in frame

Always check the ORDER BY direction and window frame before interpreting the answer.

# **13. NTILE() — divide rows into buckets**

`NTILE(N)` divides an ordered result into N approximately equal-sized buckets.

Example:

```
SELECT
    name,
    salary,

    NTILE(4) OVER (
        ORDER BY salary DESC
    ) AS salary_quartile

FROM employees;
```

This divides the employees into four groups based on salary ordering.

Applications:

- Divide customers into spending segments.
- Divide employees into salary bands.
- Divide transactions into quartiles.
- Create performance groups.

Important: NTILE distributes rows, not distinct salary values. Employees with identical salaries can end up in different buckets when a bucket boundary falls between them.

# **14. Real OA pattern — Find duplicate records**

Very common interview problem

Consider a table:

**users**

| user_id | email        |
| ------- | ------------ |
| 1       | a\@gmail.com |
| 2       | b\@gmail.com |
| 3       | a\@gmail.com |
| 4       | c\@gmail.com |
| 5       | b\@gmail.com |
| 6       | a\@gmail.com |

Question:

**Find duplicate email records while keeping the first occurrence.**

Solution:

```
WITH duplicates AS (
    SELECT
        user_id,
        email,

        ROW_NUMBER() OVER (
            PARTITION BY email
            ORDER BY user_id
        ) AS rn

    FROM users
)

SELECT *
FROM duplicates
WHERE rn > 1;
```

Output:

| user_id | email        | rn |
| ------- | ------------ | -- |
| 3       | a\@gmail.com | 2  |
| 6       | a\@gmail.com | 3  |
| 5       | b\@gmail.com | 2  |

Why does this work?

- PARTITION BY email groups identical emails.
- ORDER BY user_id sorts records within each group.
- ROW_NUMBER assigns sequential numbers.
- The first occurrence receives 1.
- Remaining occurrences receive 2, 3, etc.

To identify all duplicate records, use `rn > 1`.

To retain only one record per email, use `rn = 1`.

This is a very useful data-cleaning pattern. Actual deletion should be done only after verifying the selected records and ensuring that other tables or constraints do not depend on them.

# **15. Month-over-month growth**

This is another popular business and data engineering interview question.

Suppose a company has monthly revenue:

**monthly_revenue**

| month_start | revenue |
| ----------- | ------- |
| 2026-01-01  | 10000   |
| 2026-02-01  | 12000   |
| 2026-03-01  | 15000   |
| 2026-04-01  | 13000   |
| 2026-05-01  | 18000   |

Question:

Calculate the month-over-month percentage growth.

Formula:

\\[ \text{MoM Growth} = \frac{\text{Current Month Revenue} - \text{Previous Month Revenue}} {\text{Previous Month Revenue}} \times 100 \\]

Solution:

```
WITH revenue_comparison AS (
    SELECT
        month_start,
        revenue,

        LAG(revenue) OVER (
            ORDER BY month_start
        ) AS previous_revenue

    FROM monthly_revenue
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

FROM revenue_comparison
ORDER BY month_start;
```

Output:

| Month    | Revenue | Previous | Growth  |
| -------- | ------- | -------- | ------- |
| January  | 10000   | NULL     | NULL    |
| February | 12000   | 10000    | 20%     |
| March    | 15000   | 12000    | 25%     |
| April    | 13000   | 15000    | -13.33% |
| May      | 18000   | 13000    | 38.46%  |

### **Advanced variation: Revenue growth for every product**

Suppose you have:

| product_id | month_start | revenue |
| ---------- | ----------- | ------- |
| 1          | Jan         | 10000   |
| 1          | Feb         | 12000   |
| 1          | Mar         | 15000   |
| 2          | Jan         | 5000    |
| 2          | Feb         | 7000    |
| 2          | Mar         | 6000    |

You need to calculate growth separately for each product.

```
SELECT
    product_id,
    month_start,
    revenue,

    LAG(revenue) OVER (
        PARTITION BY product_id
        ORDER BY month_start
    ) AS previous_revenue

FROM product_revenue;
```

Why PARTITION BY?

It prevents product 2's January revenue from being compared against product 1's March revenue.

Each product gets its own independent window.

# **16. Advanced pattern — Find consecutive records**

Consider a login table:

**logins**

| user_id | login_date |
| ------- | ---------- |
| 1       | 2026-01-01 |
| 1       | 2026-01-02 |
| 1       | 2026-01-03 |
| 1       | 2026-01-05 |
| 2       | 2026-01-01 |
| 2       | 2026-01-03 |

Problem:

**Find consecutive login dates for each user.**

Use LAG:

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

SELECT
    user_id,
    login_date,
    previous_login

FROM login_comparison

WHERE login_date = previous_login + INTERVAL '1 day';
```

Output:

| user_id | login_date | previous_login |
| ------- | ---------- | -------------- |
| 1       | 2026-01-02 | 2026-01-01     |
| 1       | 2026-01-03 | 2026-01-02     |

This is useful for:

- Consecutive login days.
- Daily activity streaks.
- Consecutive transaction dates.
- Detecting missing dates.
- User retention analysis.

### **Interview extension**

**Find users who logged in for at least three consecutive days.**

This requires identifying consecutive sequences, not merely pairs. We'll revisit this in advanced SQL problem-solving using gaps-and-islands techniques.

# **17. Important window functions — revision table**

| Function    | Purpose                    | Typical question              |
| ----------- | -------------------------- | ----------------------------- |
| ROW_NUMBER  | Unique sequential position | First occurrence of duplicate |
| RANK        | Ranking with gaps          | Competition ranking           |
| DENSE_RANK  | Ranking without gaps       | Nth-highest distinct salary   |
| NTILE       | Divide rows into buckets   | Salary quartiles              |
| SUM OVER    | Running or partition total | Cumulative sales              |
| AVG OVER    | Window average             | Salary vs department average  |
| COUNT OVER  | Window count               | Employees in each department  |
| LAG         | Previous row value         | Previous month's revenue      |
| LEAD        | Next row value             | Next transaction              |
| FIRST_VALUE | First value in frame       | Highest salary                |
| LAST_VALUE  | Last value in frame        | Lowest salary                 |
| NTH_VALUE   | Nth value in frame         | Third value in a window       |

Additional functions worth exploring:

- `PERCENT_RANK()`
- `CUME_DIST()`
- `NTH_VALUE()`

You don't need to master every advanced function before understanding the common ones.

# **18. Logical execution order and window functions**

Let's extend what you learned in Lesson 1.

A simplified logical query processing sequence is:

**1**

**FROM / JOIN**\


**2**

**WHERE**\


**3**

**GROUP BY**\


**4**

**HAVING**\


**5**

**Window functions**\


Focus



**6**

**SELECT**\


**7**

**DISTINCT**\


**8**

**ORDER BY**\


**9**

**LIMIT / OFFSET**\


This explains a critical rule:

You cannot normally use a window function directly in WHERE.

Incorrect:

```
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (
        ORDER BY salary DESC
    ) AS rn
FROM employees
WHERE rn <= 3;
```

Correct:

```
WITH ranked_employees AS (
    SELECT
        name,
        salary,

        ROW_NUMBER() OVER (
            ORDER BY salary DESC
        ) AS rn

    FROM employees
)

SELECT *
FROM ranked_employees
WHERE rn <= 3;
```

**Interview question: Why do we use a CTE or subquery with window functions?**

Because window functions are calculated after the WHERE filtering stage. A CTE or derived table allows us to calculate the window result first and filter it in an outer query.

# **19. Window function interview traps**

| Trap                                 | Correct understanding                            |
| ------------------------------------ | ------------------------------------------------ |
| ROW_NUMBER vs RANK                   | Unique number vs tied ranks with gaps            |
| RANK vs DENSE_RANK                   | Gaps vs no gaps                                  |
| GROUP BY vs PARTITION BY             | Collapses rows vs preserves rows                 |
| ORDER BY in OVER vs final ORDER BY   | Calculation ordering vs display ordering         |
| Filtering a window function in WHERE | Use a CTE or subquery                            |
| LAST_VALUE without explicit frame    | May return the current row's value               |
| LAG without PARTITION BY             | Can compare across unrelated groups              |
| ROW_NUMBER without tie-breaker       | Ordering among tied rows may be nondeterministic |
| Moving average with ROWS             | Counts rows, not calendar days                   |

### **One additional advanced point**

You generally cannot nest one window function directly inside another window function.

For example, calculate a ranking first and then perform a second window operation using a CTE or subquery.

This separation makes complex analytical SQL easier to reason about.

# **20. SQL Window Functions — Interview Assessment**

Test whether you can identify the right function from a problem statement.

**Window functions assessment**

8 questions

**Q1. Which function returns unique sequential numbers, even when values are tied?**

RANK()

DENSE_RANK()

ROW_NUMBER()

NTILE()

**Q2. For scores 100, 100, 90, 80, what is the DENSE_RANK of 90?**

1

2

3

4

**Q3. Which function is appropriate for finding the second-highest distinct salary?**

ROW_NUMBER()

RANK()

DENSE_RANK()

NTILE()

**Q4. You need the top three employees in every department, including ties at third place. Which function is most appropriate?**

ROW_NUMBER()

RANK()

COUNT()

LAG()

**Q5. Which function retrieves the previous row's value?**

LEAD()

LAG()

LAST_VALUE()

FIRST_VALUE()

**Q6. Which function retrieves the next row's value?**

LEAD()

LAG()

RANK()

NTILE()

**Q7. What does PARTITION BY do?**

Deletes duplicates

Divides rows into separate windows

Sorts the final output

Filters rows

**Q8. Why explicitly specify the frame for LAST_VALUE?**

To create an index

To guarantee the intended frame, such as the entire partition

To remove NULLs

To sort the final output

Submit assessment

# **21. Practical SQL coding assignment**

These are the problems I want you to solve independently.

Use the schemas introduced in this and previous lessons.

| #  | Problem                                                                      | Difficulty | Expected technique |
| -- | ---------------------------------------------------------------------------- | ---------- | ------------------ |
| 1  | Find the second-highest distinct salary.                                     | Easy       | DENSE_RANK         |
| 2  | Find the third-highest distinct salary.                                      | Easy       | DENSE_RANK         |
| 3  | Assign a unique salary position to every employee.                           | Easy       | ROW_NUMBER         |
| 4  | Rank employees salary-wise within their departments.                         | Medium     | RANK + PARTITION   |
| 5  | Find the top three employees in each department.                             | Medium     | ROW_NUMBER         |
| 6  | Find the top two distinct salary levels in every department, including ties. | Medium     | DENSE_RANK         |
| 7  | Find employees earning more than their department average.                   | Medium     | AVG OVER           |
| 8  | Calculate cumulative salary expenditure per department.                      | Medium     | SUM OVER           |
| 9  | Calculate the three-row moving average of daily sales.                       | Medium     | AVG + ROWS         |
| 10 | Find daily sales increases compared with the previous date.                  | Medium     | LAG                |
| 11 | Calculate month-over-month revenue growth for each product.                  | Hard       | LAG + PARTITION    |
| 12 | Identify duplicate emails while preserving the earliest record.              | Medium     | ROW_NUMBER         |

**Additional challenge:**

Find employees who belong to the top 10% of earners in the company.

Hint: Explore `NTILE(10)` and `PERCENT_RANK()`.

# **22. Lesson 6 revision sheet**

**Essential syntax to remember**

**1. Ranking**

```
ROW_NUMBER() OVER (ORDER BY salary DESC)
RANK() OVER (ORDER BY salary DESC)
DENSE_RANK() OVER (ORDER BY salary DESC)
```

**2. Group-wise ranking**

```
ROW_NUMBER() OVER (
    PARTITION BY department_id
    ORDER BY salary DESC
)
```

**3. Running total**

```
SUM(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN UNBOUNDED PRECEDING
             AND CURRENT ROW
)
```

**4. Previous and next row**

```
LAG(amount, 1) OVER (ORDER BY sale_date)

LEAD(amount, 1) OVER (ORDER BY sale_date)
```

**5. Department average**

```
AVG(salary) OVER (
    PARTITION BY department_id
)
```

**6. Full-partition last value**

```
LAST_VALUE(salary) OVER (
    PARTITION BY department_id
    ORDER BY salary
    ROWS BETWEEN UNBOUNDED PRECEDING
             AND UNBOUNDED FOLLOWING
)
```

## **23. Course progress**

**SQL Mastery Course**

6 lessons covered

01\. SQL Fundamentals

Completed

02\. Aggregate Functions, GROUP BY, HAVING

Completed

03\. SQL JOINs

Completed

04\. Subqueries

Completed

05\. CTEs and Recursive CTEs

Completed

**06. Window Functions**

Current lesson

07\. Advanced SQL Functions

Next

08\. Database Design and Normalisation

Upcoming

09\. Indexes, Transactions and Optimisation

Upcoming

10\. Company OAs and Mock Interviews

Upcoming

# **Next Lesson — Lesson 7: Advanced SQL Functions**

Important for real-world OAs and interviews

We'll cover four major areas:

**1. String functions**

- LENGTH, LOWER, UPPER, TRIM
- SUBSTRING, LEFT, RIGHT
- CONCAT, REPLACE
- POSITION, SPLIT_PART
- String manipulation problems

**2. Date and time functions**

- CURRENT_DATE, CURRENT_TIMESTAMP
- EXTRACT, DATE_PART
- DATE_TRUNC, INTERVAL
- DATEDIFF equivalents
- Monthly and yearly data analysis

**3. NULL handling and conditional functions**

- COALESCE
- NULLIF
- CASE WHEN
- GREATEST and LEAST
- Conditional aggregation

**4. Practical SQL challenges**

- Find customers who registered within the last 30 days.
- Calculate employee experience.
- Extract domain names from emails.
- Find duplicate strings.
- Categorise transactions.
- Calculate age and date differences.
- Identify customers with no transactions.

**One important thing:** At this stage, don't just read SQL queries. Start writing them yourself in PostgreSQL or an online SQL editor. Your goal is to translate an English problem into the right combination of SQL operations, even when the problem is unfamiliar.