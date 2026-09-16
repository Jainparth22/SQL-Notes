# **Lesson 7 — Advanced SQL Functions: Strings, Dates, NULL Handling and Conditional Logic**



High priority for OAs

Intermediate → Advanced



So far, we've learned how to retrieve, filter, join, aggregate, rank and analyse data.

Now we'll learn how to manipulate the actual values stored inside database columns.

These functions are particularly useful in company OAs involving:

- Customer and employee records.
- Email addresses and phone numbers.
- Transaction dates and timestamps.
- Monthly and yearly reports.
- Missing or inconsistent data.
- Conditional calculations.
- Data cleaning and transformation.

We'll continue using **PostgreSQL**.

# **Part 1 — SQL String Functions**

## **1. LENGTH()**

Returns the number of characters in a string.

```
SELECT LENGTH('PARTH');
```

Output:

```
5
```

Example:

**Find employees whose names contain more than five characters.**

```
SELECT *
FROM employees
WHERE LENGTH(name) > 5;
```

Important distinction:

- `LENGTH()` returns the character count in PostgreSQL.
- `CHAR_LENGTH()` also returns the number of characters.
- `OCTET_LENGTH()` returns the number of bytes.

For ordinary English text, LENGTH and CHAR_LENGTH will usually give the same result.

## **2. LOWER() and UPPER()**

Convert text to lowercase or uppercase.

```
SELECT LOWER('PARTH JAIN');
```

Output:

```
parth jain
```

```
SELECT UPPER('parth jain');
```

Output:

```
PARTH JAIN
```

**OA problem: Find employees named Parth regardless of capitalization.**

```
SELECT *
FROM employees
WHERE LOWER(name) = 'parth';
```

This is useful when stored data has inconsistent capitalization.

## **3. INITCAP()**

Converts the first letter of each word to uppercase.

```
SELECT INITCAP('parth jain');
```

Output:

```
Parth Jain
```

Useful for normalising names.

## **4. TRIM(), LTRIM() and RTRIM()**

These functions remove unwanted characters, particularly spaces.

Suppose the database contains:

```
'   Parth Jain   '
```

Remove spaces from both ends:

```
SELECT TRIM('   Parth Jain   ');
```

Output:

```
Parth Jain
```

Remove spaces from the left:

```
SELECT LTRIM('   Parth Jain');
```

Remove spaces from the right:

```
SELECT RTRIM('Parth Jain   ');
```

**OA problem: Find users with duplicate email addresses, ignoring extra spaces and capitalization.**

Consider:

| user_id | email             |
| ------- | ----------------- |
| 1       | Parth\@gmail.com  |
| 2       | parth\@gmail.com  |
| 3       | `Parth@gmail.com` |
| 4       | rahul\@gmail.com  |

Solution:

```
SELECT
    LOWER(TRIM(email)) AS normalized_email,
    COUNT(*) AS occurrences
FROM users
GROUP BY LOWER(TRIM(email))
HAVING COUNT(*) > 1;
```

This is a common data-cleaning pattern.

## **5. SUBSTRING()**

Extracts part of a string.

Syntax:

```
SUBSTRING(string FROM start FOR length)
```

Example:

```
SELECT SUBSTRING('PARTHJAIN' FROM 1 FOR 5);
```

Output:

```
PARTH
```

Another example:

```
SELECT SUBSTRING('DATABASE' FROM 5 FOR 4);
```

Output:

```
BASE
```

Remember that PostgreSQL string positions start at 1, not 0.

### **OA problem: Extract the first three characters of employee names.**

```
SELECT
    name,
    SUBSTRING(name FROM 1 FOR 3) AS prefix
FROM employees;
```

## **6. LEFT() and RIGHT()**

Extract characters from the beginning or end.

```
SELECT LEFT('PARTHJAIN', 5);
```

Output: `PARTH`

```
SELECT RIGHT('PARTHJAIN', 4);
```

Output: `JAIN`

Example:

**Extract the last four digits of a phone number.**

```
SELECT RIGHT(phone_number, 4)
FROM users;
```

## **7. CONCAT() and CONCAT_WS()**

`CONCAT()` combines strings.

```
SELECT CONCAT('PARTH', ' ', 'JAIN');
```

Output:

```
PARTH JAIN
```

`CONCAT_WS()` combines strings using a separator.

```
SELECT CONCAT_WS('-', '2026', '11', '20');
```

Output:

```
2026-11-20
```

Useful for creating:

- Full names.
- Addresses.
- Formatted identifiers.
- Date strings.

PostgreSQL's CONCAT also treats NULL arguments as empty strings.

## **8. REPLACE()**

Replaces one substring with another.

```
SELECT REPLACE(
    'I am learning MySQL',
    'MySQL',
    'PostgreSQL'
);
```

Output:

```
I am learning PostgreSQL
```

**OA problem: Remove hyphens from phone numbers.**

```
SELECT REPLACE(phone_number, '-', '')
FROM users;
```

## **9. POSITION()**

Finds the position of a substring.

```
SELECT POSITION('@' IN 'parth@gmail.com');
```

Output:

```
6
```

This means `@` occurs at position 6.

## **10. SPLIT_PART()**

Useful OA function

Splits a string based on a delimiter and returns the required part.

Syntax:

```
SPLIT_PART(string, delimiter, field_number)
```

Example:

```
SELECT SPLIT_PART(
    'parth@gmail.com',
    '@',
    1
);
```

Output:

```
parth
```

To extract the domain:

```
SELECT SPLIT_PART(
    'parth@gmail.com',
    '@',
    2
);
```

Output:

```
gmail.com
```

**OA problem: Extract domains from email addresses.**

```
SELECT
    email,
    SPLIT_PART(email, '@', 2) AS domain
FROM users;
```

Result:

| email             | domain      |
| ----------------- | ----------- |
| parth\@gmail.com  | gmail.com   |
| rahul\@yahoo.com  | yahoo.com   |
| aman\@outlook.com | outlook.com |

## **11. STRING_AGG()**

This is an important aggregate function for combining multiple rows into one string.

Suppose we have:

| department_id | name   |
| ------------- | ------ |
| 101           | Aman   |
| 101           | Priya  |
| 101           | Simran |
| 102           | Rahul  |
| 102           | Karan  |

Question:

**Display all employee names in each department as a comma-separated string.**

```
SELECT
    department_id,
    STRING_AGG(name, ', ') AS employees
FROM employees
GROUP BY department_id;
```

Output:

| department_id | employees           |
| ------------- | ------------------- |
| 101           | Aman, Priya, Simran |
| 102           | Rahul, Karan        |

You can also control the order:

```
SELECT
    department_id,
    STRING_AGG(name, ', ' ORDER BY name) AS employees
FROM employees
GROUP BY department_id;
```

This is helpful when an interviewer asks you to combine values from multiple rows.

## **String functions — revision table**

| Function      | Purpose                           |
| ------------- | --------------------------------- |
| LENGTH()      | Count characters                  |
| CHAR_LENGTH() | Count characters                  |
| LOWER()       | Convert to lowercase              |
| UPPER()       | Convert to uppercase              |
| INITCAP()     | Capitalise words                  |
| TRIM()        | Remove outer spaces/characters    |
| LTRIM()       | Remove leading characters         |
| RTRIM()       | Remove trailing characters        |
| SUBSTRING()   | Extract a section                 |
| LEFT()        | Extract from beginning            |
| RIGHT()       | Extract from end                  |
| CONCAT()      | Combine strings                   |
| CONCAT_WS()   | Combine with separator            |
| REPLACE()     | Replace a substring               |
| POSITION()    | Find substring position           |
| SPLIT_PART()  | Extract delimiter-separated parts |
| STRING_AGG()  | Combine rows into a string        |

# **Part 2 — SQL Date and Time Functions**

Very important for company OAs

Date-related SQL problems are extremely common.

Examples:

- Find employees hired in 2025.
- Find customers registered in the last 30 days.
- Calculate employee experience.
- Find monthly sales.
- Calculate the number of days between two transactions.
- Find users inactive for 90 days.
- Calculate month-over-month revenue.

Let's master these.

## **12. Understanding SQL date data types**

| Data type   | Stores                             |
| ----------- | ---------------------------------- |
| DATE        | Calendar date                      |
| TIME        | Time of day                        |
| TIMESTAMP   | Date and time                      |
| TIMESTAMPTZ | Timestamp with time-zone awareness |
| INTERVAL    | Duration                           |

Example:

```
DATE '2026-11-20'
```

```
TIMESTAMP '2026-11-20 14:30:00'
```

```
INTERVAL '7 days'
```

## **13. CURRENT_DATE and CURRENT_TIMESTAMP**

Get the current date:

```
SELECT CURRENT_DATE;
```

Get the current timestamp:

```
SELECT CURRENT_TIMESTAMP;
```

You can also use:

```
SELECT NOW();
```

In PostgreSQL, `NOW()` returns the transaction-start timestamp. `CURRENT_TIMESTAMP` also represents the transaction-start timestamp.

## **14. EXTRACT()**

Extract a specific component from a date or timestamp.

Example:

```
SELECT
    EXTRACT(YEAR FROM DATE '2026-11-20') AS year,
    EXTRACT(MONTH FROM DATE '2026-11-20') AS month,
    EXTRACT(DAY FROM DATE '2026-11-20') AS day;
```

Output:

| year | month | day |
| ---- | ----- | --- |
| 2026 | 11    | 20  |

Other useful components:

```
EXTRACT(HOUR FROM timestamp_column)
EXTRACT(MINUTE FROM timestamp_column)
EXTRACT(SECOND FROM timestamp_column)
EXTRACT(DOW FROM date_column)
EXTRACT(QUARTER FROM date_column)
```

`DOW` gives the day of the week in PostgreSQL, where Sunday is 0 and Saturday is 6.

### **OA problem: Find employees hired during 2025.**

```
SELECT *
FROM employees
WHERE EXTRACT(YEAR FROM hire_date) = 2025;
```

This works, but applying a function to an indexed date column can make ordinary index usage less straightforward.

A generally preferable form is:

```
SELECT *
FROM employees
WHERE hire_date >= DATE '2025-01-01'
  AND hire_date < DATE '2026-01-01';
```

This is a useful query optimisation habit.

## **15. DATE_TRUNC()**

Important for analytics OAs

`DATE_TRUNC()` truncates a timestamp to a specified level of precision.

For example:

```
SELECT DATE_TRUNC(
    'month',
    TIMESTAMP '2026-11-20 14:35:40'
);
```

Output:

```
2026-11-01 00:00:00
```

Other examples:

```
DATE_TRUNC('year', timestamp_column)
DATE_TRUNC('month', timestamp_column)
DATE_TRUNC('week', timestamp_column)
DATE_TRUNC('day', timestamp_column)
```

### **OA problem: Calculate monthly sales.**

Consider:

**orders**

| order_id | order_date | amount |
| -------- | ---------- | ------ |
| 1        | 2026-01-05 | 500    |
| 2        | 2026-01-15 | 800    |
| 3        | 2026-02-10 | 600    |
| 4        | 2026-02-20 | 1000   |

Solution:

```
SELECT
    DATE_TRUNC('month', order_date) AS sales_month,
    SUM(amount) AS total_sales
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY sales_month;
```

Output:

| sales_month | total_sales |
| ----------- | ----------- |
| Jan 2026    | 1300        |
| Feb 2026    | 1600        |

This is a standard monthly reporting query.

## **16. INTERVAL**

INTERVAL represents a duration.

Examples:

```
INTERVAL '7 days'
INTERVAL '3 months'
INTERVAL '2 years'
INTERVAL '5 hours'
```

You can perform date arithmetic:

```
SELECT CURRENT_DATE + INTERVAL '7 days';
```

Subtract 30 days:

```
SELECT CURRENT_DATE - INTERVAL '30 days';
```

### **OA problem: Find customers registered during the last 30 days.**

Assuming `created_at` is a timestamp:

```
SELECT *
FROM customers
WHERE created_at >= NOW() - INTERVAL '30 days'
  AND created_at <= NOW();
```

This checks a rolling 30-day period relative to the current transaction timestamp.

For a date-only column:

```
SELECT *
FROM customers
WHERE registration_date >= CURRENT_DATE - 30
  AND registration_date <= CURRENT_DATE;
```

## **17. AGE()**

Calculates the difference between two dates or timestamps as an interval.

```
SELECT AGE(
    DATE '2026-11-20',
    DATE '2020-11-20'
);
```

Output:

```
6 years
```

### **OA problem: Calculate employee experience in complete years.**

```
SELECT
    name,
    hire_date,
    EXTRACT(
        YEAR FROM AGE(CURRENT_DATE, hire_date)
    ) AS experience_years
FROM employees;
```

This calculates complete calendar years of service.

Remember: experience is different from simply dividing the difference in days by 365.

## **18. Date subtraction**

PostgreSQL supports direct subtraction.

```
SELECT
    DATE '2026-11-20' -
    DATE '2026-11-01' AS difference;
```

Output:

```
19
```

The result is the number of days between two dates.

For timestamps:

```
SELECT
    TIMESTAMP '2026-11-20 15:00:00' -
    TIMESTAMP '2026-11-20 10:00:00';
```

Output:

```
05:00:00
```

## **19. TO_CHAR()**

Formats dates and numbers into strings.

Example:

```
SELECT TO_CHAR(
    DATE '2026-11-20',
    'DD-MM-YYYY'
);
```

Output:

```
20-11-2026
```

Useful format patterns:

| Pattern | Meaning           |
| ------- | ----------------- |
| YYYY    | Four-digit year   |
| MM      | Month             |
| DD      | Day               |
| HH24    | 24-hour clock     |
| MI      | Minute            |
| SS      | Second            |
| Month   | Full month name   |
| Mon     | Abbreviated month |

Example:

```
SELECT TO_CHAR(
    CURRENT_TIMESTAMP,
    'DD Mon YYYY HH24:MI'
);
```

## **20. Date function differences across SQL dialects**

Not every database supports identical syntax.

| Requirement       | PostgreSQL             | MySQL                       |
| ----------------- | ---------------------- | --------------------------- |
| Current date      | CURRENT_DATE           | CURRENT_DATE()              |
| Current timestamp | CURRENT_TIMESTAMP      | CURRENT_TIMESTAMP()         |
| Extract year      | EXTRACT(YEAR FROM d)   | YEAR(d)                     |
| Truncate to month | DATE_TRUNC('month', d) | DATE_FORMAT(d, '%Y-%m-01')  |
| Date difference   | d1 - d2                | DATEDIFF(d1, d2)            |
| Format date       | TO_CHAR()              | DATE_FORMAT()               |
| Add days          | d + INTERVAL '7 days'  | DATE_ADD(d, INTERVAL 7 DAY) |
| Split strings     | SPLIT_PART()           | SUBSTRING_INDEX()           |

**Important for interviews and OAs:** Always check which SQL dialect the assessment uses. We are learning PostgreSQL, so don't blindly copy PostgreSQL syntax into MySQL assessments.

# **Part 3 — NULL Handling**

Frequently tested interview concepts

NULL means missing or unknown information.

Let's use this table:

**employees**

| emp_id | name  | salary | bonus |
| ------ | ----- | ------ | ----- |
| 1      | Aman  | 60000  | 5000  |
| 2      | Rahul | 80000  | NULL  |
| 3      | Priya | 75000  | 7000  |
| 4      | Neha  | 90000  | NULL  |
| 5      | Karan | 50000  | 3000  |

## **21. COALESCE()**

Returns the first non-NULL value.

Syntax:

```
COALESCE(value1, value2, value3, ...)
```

Example:

```
SELECT COALESCE(NULL, NULL, 100, 200);
```

Output:

```
100
```

### **OA problem: Treat missing employee bonuses as zero.**

```
SELECT
    name,
    salary,
    COALESCE(bonus, 0) AS bonus
FROM employees;
```

Result:

| name  | salary | bonus |
| ----- | ------ | ----- |
| Aman  | 60000  | 5000  |
| Rahul | 80000  | 0     |
| Priya | 75000  | 7000  |
| Neha  | 90000  | 0     |
| Karan | 50000  | 3000  |

### **Calculate total compensation**

```
SELECT
    name,
    salary + COALESCE(bonus, 0) AS total_compensation
FROM employees;
```

Why do we need COALESCE?

Because:

```
80000 + NULL
```

returns NULL, not 80000.

COALESCE prevents missing bonus values from propagating into the calculation.

## **22. NULLIF()**

Returns NULL if the two arguments are equal. Otherwise, it returns the first argument.

Syntax:

```
NULLIF(value1, value2)
```

Example:

```
SELECT NULLIF(10, 10);
```

Output:

```
NULL
```

Another example:

```
SELECT NULLIF(10, 5);
```

Output:

```
10
```

### **OA problem: Prevent division by zero.**

Suppose we want to calculate a conversion rate:

\\[ \text{Conversion Rate} = \frac{\text{Conversions}}{\text{Visitors}} \times 100 \\]

Incorrect:

```
SELECT conversions / visitors
FROM website_stats;
```

If visitors is zero, this can raise a division-by-zero error.

Correct:

```
SELECT
    100.0 * conversions /
    NULLIF(visitors, 0) AS conversion_rate
FROM website_stats;
```

When visitors is zero, NULLIF returns NULL, preventing division by zero.

## **23. COALESCE vs NULLIF**

| COALESCE                     | NULLIF                                 |
| ---------------------------- | -------------------------------------- |
| Returns first non-NULL value | Returns NULL when values are equal     |
| Handles missing values       | Handles special equality cases         |
| Useful for defaults          | Useful for division-by-zero protection |

Remember:

- COALESCE: Find the first available value.
- NULLIF: Convert a specific value into NULL.

# **Part 4 — CASE WHEN and Conditional Logic**

You already learned CASE WHEN in Lesson 1. Now let's understand its advanced applications.

## **24. CASE WHEN**

Syntax:

```
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

Example:

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

The first matching condition is used.

### **Multiple conditions**

```
SELECT
    name,
    salary,

    CASE
        WHEN department_id = 101
             AND salary >= 80000
            THEN 'Engineering High Earner'

        WHEN salary >= 80000
            THEN 'Other High Earner'

        ELSE 'Other Employee'
    END AS employee_category

FROM employees;
```

## **25. Conditional aggregation**

This is one of the most valuable techniques for OA solving.

Consider a transactions table:

| transaction_id | status    | amount |
| -------------- | --------- | ------ |
| 1              | Completed | 1000   |
| 2              | Failed    | 500    |
| 3              | Completed | 1500   |
| 4              | Pending   | 700    |
| 5              | Failed    | 300    |

Question:

Find:

- Total transactions.
- Successful transactions.
- Failed transactions.
- Successful transaction value.

Solution:

```
SELECT
    COUNT(*) AS total_transactions,

    SUM(
        CASE
            WHEN status = 'Completed'
            THEN 1
            ELSE 0
        END
    ) AS successful_transactions,

    SUM(
        CASE
            WHEN status = 'Failed'
            THEN 1
            ELSE 0
        END
    ) AS failed_transactions,

    SUM(
        CASE
            WHEN status = 'Completed'
            THEN amount
            ELSE 0
        END
    ) AS successful_amount

FROM transactions;
```

Output:

| total_transactions | successful_transactions | failed_transactions | successful_amount |
| ------------------ | ----------------------- | ------------------- | ----------------- |
| 5                  | 2                       | 2                   | 2500              |

This is called conditional aggregation.

### **Alternative: PostgreSQL FILTER**

PostgreSQL supports the FILTER clause for aggregate functions.

```
SELECT
    COUNT(*) AS total_transactions,

    COUNT(*) FILTER (
        WHERE status = 'Completed'
    ) AS successful_transactions,

    COUNT(*) FILTER (
        WHERE status = 'Failed'
    ) AS failed_transactions,

    SUM(amount) FILTER (
        WHERE status = 'Completed'
    ) AS successful_amount

FROM transactions;
```

Both approaches are useful.

For cross-database compatibility, prefer `CASE WHEN` because FILTER support varies by database.

## **26. GREATEST() and LEAST()**

Find the highest or lowest value among multiple expressions.

```
SELECT GREATEST(10, 20, 30, 40);
```

Output: `40`

```
SELECT LEAST(10, 20, 30, 40);
```

Output: `10`

Practical example:

```
SELECT
    name,
    GREATEST(salary, 60000) AS adjusted_salary
FROM employees;
```

This returns at least 60000 for every employee.

**PostgreSQL-specific detail:** GREATEST and LEAST ignore NULL arguments when at least one non-NULL argument exists. This behaviour differs from some other SQL databases.

# **Part 5 — Numeric Functions and Type Conversion**

These are useful in financial and analytical OA questions.

## **27. ROUND()**

Rounds a number to the specified number of decimal places.

```
SELECT ROUND(123.4567, 2);
```

Output:

```
123.46
```

Example:

```
SELECT
    department_id,
    ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department_id;
```

## **28. CEIL() and FLOOR()**

`CEIL()` rounds upward to the nearest integer.

```
SELECT CEIL(12.2);
```

Output: `13`

`FLOOR()` rounds downward.

```
SELECT FLOOR(12.9);
```

Output: `12`

## **29. ABS()**

Returns the absolute value.

```
SELECT ABS(-250);
```

Output: `250`

Example:

```
SELECT
    ABS(actual_amount - expected_amount) AS difference
FROM transactions;
```

## **30. CAST()**

Converts one data type into another.

```
SELECT CAST('2026-11-20' AS DATE);
```

Another syntax supported by PostgreSQL:

```
SELECT '2026-11-20'::DATE;
```

Convert an integer into a decimal for accurate division:

```
SELECT
    5::NUMERIC / 2;
```

Output:

```
2.5000000000000000
```

Without a decimal type, integer division can discard the fractional part.

**OA tip:** When calculating percentages, use `100.0` rather than `100` to avoid unintended integer arithmetic.

# **Part 6 — Advanced OA Problem-Solving**

Now combine the functions you have learned.

## **Problem 1: Extract the email username and domain**

```
SELECT
    email,

    SPLIT_PART(email, '@', 1) AS username,

    SPLIT_PART(email, '@', 2) AS domain

FROM users;
```

## **Problem 2: Find customers registered in the last 30 days**

```
SELECT *
FROM customers
WHERE created_at >= NOW() - INTERVAL '30 days';
```

For a strict rolling window that excludes future timestamps:

```
WHERE created_at BETWEEN NOW() - INTERVAL '30 days'
                     AND NOW()
```

## **Problem 3: Calculate employee experience**

```
SELECT
    name,
    hire_date,
    EXTRACT(
        YEAR FROM AGE(CURRENT_DATE, hire_date)
    ) AS experience_years
FROM employees;
```

## **Problem 4: Categorise transactions**

```
SELECT
    transaction_id,
    amount,

    CASE
        WHEN amount >= 10000 THEN 'Large'
        WHEN amount >= 5000 THEN 'Medium'
        ELSE 'Small'
    END AS transaction_category

FROM transactions;
```

## **Problem 5: Monthly successful transaction amount**

```
SELECT
    DATE_TRUNC('month', transaction_date) AS month,

    SUM(amount) FILTER (
        WHERE status = 'Completed'
    ) AS successful_amount

FROM transactions

GROUP BY DATE_TRUNC('month', transaction_date)

ORDER BY month;
```

## **Problem 6: Find employees whose email domains are Gmail**

```
SELECT *
FROM employees
WHERE LOWER(
    SPLIT_PART(email, '@', 2)
) = 'gmail.com';
```

## **Problem 7: Calculate average order value safely**

```
SELECT
    SUM(amount) /
    NULLIF(COUNT(DISTINCT order_id), 0) AS average_order_value
FROM orders;
```

## **Problem 8: Find the difference between actual and expected amounts, rounded to two decimal places**

```
SELECT
    transaction_id,

    ROUND(
        ABS(actual_amount - expected_amount),
        2
    ) AS absolute_difference

FROM transactions;
```

# **Part 7 — Interview Questions**

| Question                                               | Interview-ready answer                                                    |
| ------------------------------------------------------ | ------------------------------------------------------------------------- |
| What is COALESCE?                                      | Returns the first non-NULL expression.                                    |
| What is NULLIF?                                        | Returns NULL when its two arguments are equal.                            |
| What is the difference between LENGTH and CHAR_LENGTH? | PostgreSQL treats both as character counts; OCTET_LENGTH measures bytes.  |
| What does DATE_TRUNC do?                               | Truncates a date or timestamp to a specified precision.                   |
| What does EXTRACT do?                                  | Retrieves a specified date/time component.                                |
| What is an INTERVAL?                                   | Represents a duration of time.                                            |
| What is the use of LAG?                                | Retrieves a value from a preceding row.                                   |
| What is STRING_AGG?                                    | Combines values from multiple rows into one string.                       |
| What is conditional aggregation?                       | Uses conditions within aggregate calculations.                            |
| Why use NULLIF in division?                            | Prevents division-by-zero errors by returning NULL for zero denominators. |

# **Part 8 — Interactive Assessment**

Test your knowledge of advanced SQL functions.

**Advanced SQL assessment**

8 questions

**Q1. Which function extracts the domain from 'parth\@gmail.com' in PostgreSQL?**

SUBSTRING(email, 1, 5)

SPLIT_PART(email, '@', 2)

RIGHT(email, 5)

POSITION('@' IN email)

**Q2. Which function returns the first non-NULL value?**

NULLIF

COALESCE

REPLACE

CONCAT

**Q3. Which PostgreSQL function truncates a timestamp to month level?**

EXTRACT

AGE

DATE_TRUNC

TO_CHAR

**Q4. Which is the safest expression for dividing by a potentially zero denominator?**

a / b

a / COALESCE(b, 0)

a / NULLIF(b, 0)

a \* NULLIF(b, 0)

**Q5. Which function combines values from multiple rows into a delimited string in PostgreSQL?**

CONCAT

STRING_AGG

SPLIT_PART

POSITION

**Q6. Which expression calculates complete years between hire_date and today in PostgreSQL?**

CURRENT_DATE - hire_date

EXTRACT(YEAR FROM AGE(CURRENT_DATE, hire_date))

YEAR(hire_date)

DATE_TRUNC('year', hire_date)

**Q7. Which function retrieves the second component from '2026-11-20' when split using '-'?**

SPLIT_PART('2026-11-20', '-', 2)

RIGHT('2026-11-20', 2)

SUBSTRING('2026-11-20' FROM 1 FOR 2)

POSITION('-' IN '2026-11-20')

**Q8. Which SQL technique calculates successful transaction count and failed transaction count in one grouped query?**

CROSS JOIN

Conditional aggregation

SELF JOIN only

ORDER BY

Submit assessment

# **Part 9 — Practical Coding Assignment**

Write the following queries independently.

| #  | Problem                                                                                        | Difficulty |
| -- | ---------------------------------------------------------------------------------------------- | ---------- |
| 1  | Extract usernames from email addresses.                                                        | Easy       |
| 2  | Find emails with the domain gmail.com, ignoring case.                                          | Easy       |
| 3  | Find names containing more than six characters.                                                | Easy       |
| 4  | Remove unnecessary spaces and normalise names.                                                 | Easy       |
| 5  | Find customers who registered during the last 30 days.                                         | Easy       |
| 6  | Calculate employee experience in complete years.                                               | Medium     |
| 7  | Calculate total sales for each calendar month.                                                 | Medium     |
| 8  | Find transactions where actual and expected values differ.                                     | Medium     |
| 9  | Calculate month-over-month revenue growth.                                                     | Medium     |
| 10 | Count completed, pending and failed transactions in one query.                                 | Medium     |
| 11 | Calculate conversion rate while safely handling zero visitors.                                 | Medium     |
| 12 | Group employee names into comma-separated department lists.                                    | Medium     |
| 13 | Find duplicate emails after normalising case and whitespace.                                   | Medium     |
| 14 | Calculate the percentage of successful transactions.                                           | Hard       |
| 15 | Find customers who registered in one month but never made a purchase in the following 30 days. | Hard       |

# **Part 10 — SQL Mastery Course Progress**

**SQL learning roadmap**

7 / 10 lessons

01\. SQL fundamentals

Covered

02\. Aggregate functions, GROUP BY, HAVING

Covered

03\. SQL JOINs

Covered

04\. Subqueries

Covered

05\. CTEs and recursive CTEs

Covered

06\. Window functions

Covered

07\. Advanced SQL functions

Covered

**08. Database design and normalisation**

Next

09\. Indexes, transactions and query optimisation

10\. Company OAs and mock interviews

# **Next Lesson — Lesson 8: Database Design and Normalisation**

We'll cover:

1. Relational database design from first principles.
2. Primary, foreign, candidate and composite keys.
3. Functional dependencies.
4. First Normal Form (1NF).
5. Second Normal Form (2NF).
6. Third Normal Form (3NF).
7. BCNF.
8. Partial and transitive dependencies.
9. Database denormalisation.
10. ER diagrams and cardinality.
11. One-to-one, one-to-many and many-to-many relationships.
12. Real interview questions involving schema design.

**My recommendation:** You have now covered most of the SQL query-writing toolkit. Start solving actual problems alongside these lessons rather than waiting until the course ends. Your next major goal should be to become fluent at identifying when a problem needs a JOIN, GROUP BY, subquery, CTE or window function.