# **SQL Interview Mastery — Part 3**

## **Advanced PostgreSQL and Backend Database Engineering**

Advanced

Backend Engineering

PostgreSQL

So far, you've learned how to retrieve, manipulate, aggregate and optimize relational data.

Now we're moving into something particularly important for backend engineering: **how applications interact with databases in production.**

For example:

- How do you insert a record if it doesn't exist, otherwise update it?
- How do you store flexible JSON data in PostgreSQL?
- When should you use a view or materialized view?
- What are triggers and stored procedures?
- How do you process millions of records efficiently?
- How do you manage database connections in a backend application?

Let's learn each from first principles.

# **1. UPSERT — INSERT or UPDATE**

This is one of the most frequently used PostgreSQL features in backend applications.

## **1.1 Understand the problem**

Suppose you have an inventory table:

```
CREATE TABLE inventory (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    quantity INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Initially:

| product_id | product_name | quantity |
| ---------- | ------------ | -------- |
| 101        | Laptop       | 10       |
| 102        | Mouse        | 20       |

Now suppose your application receives this data:

```
product_id = 101
product_name = Laptop
quantity = 5
```

What should happen?

- If product 101 doesn't exist, insert it.
- If product 101 already exists, update its quantity.

This operation is called **UPSERT**.

UPSERT = INSERT + UPDATE.

## **1.2 The traditional approach and its problem**

A developer might write:

```
SELECT *
FROM inventory
WHERE product_id = 101;
```

If the product exists:

```
UPDATE inventory
SET quantity = quantity + 5
WHERE product_id = 101;
```

Otherwise:

```
INSERT INTO inventory (
    product_id,
    product_name,
    quantity
)
VALUES (101, 'Laptop', 5);
```

There is a potential concurrency problem.

Imagine two application requests arriving simultaneously.

**Two concurrent requests**

**Request A**

Checks product 103. Doesn't exist.

Plans to INSERT.

**Request B**

Checks product 103. Doesn't exist.

Also plans to INSERT.

Both attempt insertion. One succeeds; the other may encounter a unique-constraint violation.

The read-then-write approach is not inherently atomic.

## **1.3 PostgreSQL solution — ON CONFLICT**

```
INSERT INTO inventory (
    product_id,
    product_name,
    quantity
)
VALUES (
    101,
    'Laptop',
    5
)
ON CONFLICT (product_id)
DO UPDATE
SET
    quantity = inventory.quantity + EXCLUDED.quantity,
    updated_at = CURRENT_TIMESTAMP;
```

This is the PostgreSQL UPSERT syntax.

Understand its components:

| Component      | Meaning                                                |
| -------------- | ------------------------------------------------------ |
| `INSERT INTO`  | Attempts to insert a row                               |
| `ON CONFLICT`  | Defines what happens when a uniqueness conflict occurs |
| `(product_id)` | Identifies the conflict target                         |
| `DO UPDATE`    | Update the existing row                                |
| `EXCLUDED`     | The proposed row that could not be inserted            |

### **What is EXCLUDED?**

Suppose the existing record is:

| product_id | quantity |
| ---------- | -------- |
| 101        | 10       |

New incoming record:

| product_id | quantity |
| ---------- | -------- |
| 101        | 5        |

In:

```
SET quantity = inventory.quantity + EXCLUDED.quantity
```

- `inventory.quantity` = existing quantity = 10.
- `EXCLUDED.quantity` = incoming quantity = 5.

Result:

\\[ 10+5=15 \\]

This is useful for inventory updates, counters and accumulated statistics.

## **1.4 ON CONFLICT DO NOTHING**

Sometimes you don't want an existing record to be updated.

```
INSERT INTO inventory (
    product_id,
    product_name,
    quantity
)
VALUES (
    101,
    'Laptop',
    5
)
ON CONFLICT (product_id)
DO NOTHING;
```

If the product already exists, PostgreSQL skips the conflicting insertion.

## **1.5 UPSERT with RETURNING**

You can return the inserted or updated record:

```
INSERT INTO inventory (
    product_id,
    product_name,
    quantity
)
VALUES (
    101,
    'Laptop',
    5
)
ON CONFLICT (product_id)
DO UPDATE
SET quantity = inventory.quantity + EXCLUDED.quantity
RETURNING *;
```

`RETURNING` is very useful in backend applications because you can obtain the resulting database record without another SELECT query.

## **1.6 Conditional UPSERT**

Suppose inventory must never exceed 100 units.

```
INSERT INTO inventory (
    product_id,
    product_name,
    quantity
)
VALUES (
    101,
    'Laptop',
    5
)
ON CONFLICT (product_id)
DO UPDATE
SET quantity = inventory.quantity + EXCLUDED.quantity
WHERE inventory.quantity + EXCLUDED.quantity <= 100
RETURNING *;
```

If the condition is false, PostgreSQL does not update the existing row.

The existing row remains unchanged.

**Important:** The row may not appear in `RETURNING` when the conflict occurs but the update condition is false.

### **Interview questions**

**Q. Is UPSERT atomic?**

Yes. PostgreSQL's `INSERT ... ON CONFLICT DO UPDATE` performs conflict detection and the resulting insert or update as a database operation with concurrency control. A read-then-write implementation in application code is not equivalent.

**Q. Does UPSERT require a unique constraint?**

The conflict target must correspond to a usable unique constraint or unique index, or a suitable exclusion constraint for supported conflict handling. A typical `ON CONFLICT (column)` example uses a primary key or unique index.

**Q. What's the difference between DO NOTHING and DO UPDATE?**

`DO NOTHING` skips the conflicting insertion; `DO UPDATE` modifies the existing conflicting row.

# **2. JSON and JSONB in PostgreSQL**

This is especially important for backend developers working with APIs.

## **2.1 Why do we need JSON?**

Imagine you are building a user profile system.

Traditional relational design:

```
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(100),
    country VARCHAR(100),
    theme VARCHAR(20),
    language VARCHAR(20)
);
```

But consider different users.

User A:

```
{
  "theme": "dark",
  "language": "en",
  "notifications": true
}
```

User B:

```
{
  "theme": "light",
  "language": "hi",
  "notifications": false,
  "timezone": "Asia/Kolkata"
}
```

Some users have different attributes.

You could continuously add new columns, but that isn't always the best design for flexible or optional attributes.

PostgreSQL provides `JSON` and `JSONB`.

## **2.2 JSON vs JSONB**

| Feature                  | JSON                                        | JSONB                                     |
| ------------------------ | ------------------------------------------- | ----------------------------------------- |
| Storage                  | Stores JSON text representation             | Stores a decomposed binary representation |
| Parsing                  | Reparsed when processed                     | Avoids reparsing the original text        |
| Indexing                 | Limited compared with JSONB                 | Rich indexing support                     |
| Whitespace and key order | Preserved                                   | Not preserved                             |
| Duplicate object keys    | Preserved in input text                     | Only the last duplicate key is retained   |
| Typical use              | Retaining original JSON text representation | Querying and indexing structured JSON     |

For most applications that frequently query JSON fields, **JSONB is generally the preferred choice**.

## **2.3 Create a table with JSONB**

```
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    profile JSONB
);
```

Insert data:

```
INSERT INTO user_profiles
VALUES (
    1,
    'Parth',
    '{
        "theme": "dark",
        "language": "en",
        "notifications": true,
        "skills": ["Python", "SQL", "Java"]
    }'::jsonb
);
```

The `::jsonb` syntax casts the supplied string into JSONB.

## **2.4 JSONB operators**

These operators are essential.

Consider:

```
{
  "theme": "dark",
  "language": "en",
  "skills": ["Python", "SQL", "Java"]
}
```

\| Operator | Meaning |\
\|---|---|\
\| `->` | Extract JSON object field or array element |\
\| `->>` | Extract a field or element as text |\
\| `#>` | Extract a nested JSON path |\
\| `#>>` | Extract a nested path as text |\
\| `@>` | Check JSON containment |\
\| `?` | Check whether an object has a key or an array contains a string element |\
\| `?|` | Check whether any specified keys/elements exist |\
\| `?&` | Check whether all specified keys/elements exist |

### **Example A — Extract a JSON value**

```
SELECT profile -> 'theme'
FROM user_profiles;
```

Result:

```
"dark"
```

### **Example B — Extract as text**

```
SELECT profile ->> 'theme'
FROM user_profiles;
```

Result:

```
dark
```

The distinction matters.

`->` returns JSONB.

`->>` returns text.

### **Example C — Extract nested attributes**

Suppose:

```
{
  "personal": {
    "city": "Bhopal",
    "country": "India"
  }
}
```

Query:

```
SELECT profile #> '{personal,city}'
FROM user_profiles;
```

As text:

```
SELECT profile #>> '{personal,city}'
FROM user_profiles;
```

### **Example D — Filter by JSON attributes**

```
SELECT *
FROM user_profiles
WHERE profile ->> 'theme' = 'dark';
```

### **Example E — Check containment**

Find profiles containing Python in their skills array:

```
SELECT *
FROM user_profiles
WHERE profile @> '{"skills":["Python"]}'::jsonb;
```

This uses JSONB containment semantics.

## **2.5 Updating JSONB**

Suppose you want to change the theme to light.

```
UPDATE user_profiles
SET profile = jsonb_set(
    profile,
    '{theme}',
    '"light"'::jsonb
)
WHERE user_id = 1;
```

Notice the double quotes inside the JSONB value:

```
'"light"'::jsonb
```

The JSON string value is `"light"`.

A JSON string and a SQL text value are not the same representation.

### **Add another key**

```
UPDATE user_profiles
SET profile = profile || '{"timezone":"Asia/Kolkata"}'::jsonb
WHERE user_id = 1;
```

The `||` operator merges JSONB values.

For objects, it combines top-level keys; a right-hand value replaces a matching left-hand key.

For nested updates, use `jsonb_set()` with the appropriate path. Intermediate path elements generally need to exist for the path update to take effect.

## **2.6 Convert JSON arrays into SQL rows**

Suppose:

```
{
  "skills": ["Python", "SQL", "Java"]
}
```

You want one SQL row per skill.

```
SELECT
    u.user_id,
    skill.value
FROM user_profiles u
CROSS JOIN LATERAL
    jsonb_array_elements_text(
        u.profile -> 'skills'
    ) AS skill(value);
```

Expected result:

| user_id | value  |
| ------- | ------ |
| 1       | Python |
| 1       | SQL    |
| 1       | Java   |

Here, `jsonb_array_elements_text()` expands the JSON array into rows.

`LATERAL` allows the function to use the JSON value from the current user row.

## **2.7 Indexing JSONB**

Without an appropriate index, filtering on JSON attributes can require examining many rows.

Create a GIN index:

```
CREATE INDEX idx_profile_gin
ON user_profiles
USING GIN (profile);
```

Useful for containment and supported existence queries.

For a particular frequently queried scalar:

```
CREATE INDEX idx_profile_theme
ON user_profiles ((profile ->> 'theme'));
```

This expression index is useful for:

```
SELECT *
FROM user_profiles
WHERE profile ->> 'theme' = 'dark';
```

PostgreSQL also offers the `jsonb_path_ops` GIN operator class:

```
CREATE INDEX idx_profile_path
ON user_profiles
USING GIN (profile jsonb_path_ops);
```

It can be more efficient for supported containment searches, but supports fewer operators than the default JSONB GIN operator class.

### **When should you use JSONB?**

Good use cases:

- Flexible user preferences.
- Optional metadata.
- External API payloads.
- Configurations.
- Semi-structured event attributes.

Avoid blindly putting everything in JSONB.

For frequently filtered, joined, validated and constrained business entities, normal relational columns are often a better choice.

**Interview takeaway:** JSONB provides flexible semi-structured storage while retaining PostgreSQL querying and indexing capabilities. It should complement relational modeling, not replace it indiscriminately.

# **3. Views vs Materialized Views**

Both features let you save a query definition, but they behave differently.

## **3.1 What is a view?**

A view is a named SQL query.

Example:

```
CREATE VIEW completed_orders AS
SELECT
    order_id,
    customer_id,
    order_date,
    total_amount
FROM orders
WHERE status = 'Completed';
```

Now you can query:

```
SELECT *
FROM completed_orders;
```

It behaves like a virtual table.

A normal PostgreSQL view does not store the query's result rows as a separate materialized data set.

When queried, its defining query is incorporated into the query that reads the view.

### **Advantages of views**

- Simplify complex SQL.
- Reuse query definitions.
- Provide controlled access to selected columns or rows.
- Help separate application-facing interfaces from underlying table structures.

### **Can we update a view?**

Some views are automatically updatable.

For example:

```
CREATE VIEW employee_public AS
SELECT
    emp_id,
    emp_name,
    department_id
FROM employees;
```

A simple view like this may support:

```
UPDATE employee_public
SET emp_name = 'Parth'
WHERE emp_id = 101;
```

But views involving grouping, aggregates, `DISTINCT`, set operations and various other constructs generally cannot be directly updated without additional mechanisms such as `INSTEAD OF` triggers.

## **3.2 What is a materialized view?**

A materialized view stores the result of a query.

Example:

```
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT
    DATE_TRUNC('month', order_date) AS sales_month,
    SUM(total_amount) AS total_revenue
FROM orders
WHERE status = 'Completed'
GROUP BY DATE_TRUNC('month', order_date);
```

Now:

```
SELECT *
FROM monthly_sales;
```

Reads the stored results.

Unlike a normal view, the materialized view does not automatically reflect every change to its source tables.

You must refresh it.

```
REFRESH MATERIALIZED VIEW monthly_sales;
```

## **3.3 Refresh concurrently**

For a materialized view used by a production application, a regular refresh may block concurrent readers while it refreshes.

PostgreSQL supports:

```
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
```

But there is a requirement:

The materialized view must already be populated and have a suitable qualifying unique index using only column names and covering all rows, without a partial predicate or expression.

Example:

```
CREATE UNIQUE INDEX idx_monthly_sales_month
ON monthly_sales(sales_month);
```

Then:

```
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;
```

This allows concurrent reads during the refresh, with additional resource and processing overhead.

### **Comparison**

| Feature                                             | View                              | Materialized view                 |
| --------------------------------------------------- | --------------------------------- | --------------------------------- |
| Stores result rows                                  | No                                | Yes                               |
| Reflects source-table changes on subsequent queries | Yes                               | Only after refresh                |
| Faster reads for expensive precomputed queries      | Not inherently                    | Often                             |
| Requires refresh                                    | No                                | Yes                               |
| Uses separate storage                               | No stored result set              | Yes                               |
| Good for                                            | Simplification and access control | Precomputed reports and analytics |

### **Practical backend example**

Imagine an e-commerce dashboard showing monthly sales across millions of orders.

Computing the aggregate for every dashboard request could be expensive.

You can precompute the aggregate in a materialized view and refresh it periodically.

This trades data freshness for faster report queries.

**Interview question: Can a materialized view become stale?**

Yes. It contains the result captured at its last refresh, so source-table changes are not automatically reflected.

# **4. SQL Functions vs Stored Procedures**

Both can encapsulate reusable database logic.

But there are important differences.

## **4.1 SQL function**

A function is a database object that returns a value or set of rows.

Example:

```
CREATE FUNCTION calculate_annual_salary(
    monthly_salary NUMERIC
)
RETURNS NUMERIC
LANGUAGE SQL
IMMUTABLE
AS $$
    SELECT monthly_salary * 12;
$$;
```

Execute it:

```
SELECT calculate_annual_salary(50000);
```

Result:

```
600000
```

### **Function characteristics**

- Has a return type.
- Can return scalar values, composite values or sets of rows.
- Can be used in SQL expressions where its return type is compatible.
- Can be written in SQL or procedural languages such as PL/pgSQL.

### **What does IMMUTABLE mean?**

PostgreSQL function volatility categories include:

| Category    | Meaning                                                                     |
| ----------- | --------------------------------------------------------------------------- |
| `IMMUTABLE` | Same arguments always produce the same result                               |
| `STABLE`    | Same arguments produce the same result within a single statement's snapshot |
| `VOLATILE`  | May produce different results within a statement or have side effects       |

Our annual-salary function is immutable because multiplication always produces the same result for the same input.

Do not mark a function immutable just to encourage optimization. Its declaration must match its actual behaviour.

## **4.2 Function returning a table**

Example: Find all employees in a department.

```
CREATE FUNCTION get_department_employees(
    dept_id INT
)
RETURNS TABLE (
    emp_id INT,
    emp_name VARCHAR,
    salary NUMERIC
)
LANGUAGE SQL
AS $$
    SELECT
        e.emp_id,
        e.emp_name,
        e.salary
    FROM employees e
    WHERE e.department_id = dept_id;
$$;
```

Execute:

```
SELECT *
FROM get_department_employees(10);
```

This returns rows like a table-valued query.

## **4.3 PL/pgSQL function**

PL/pgSQL is PostgreSQL's procedural language.

It supports:

- Variables.
- IF/ELSE.
- Loops.
- Exceptions.
- SQL statements.
- Control flow.

Example:

```
CREATE FUNCTION salary_grade(
    salary_value NUMERIC
)
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    IF salary_value >= 100000 THEN
        RETURN 'High';

    ELSIF salary_value >= 50000 THEN
        RETURN 'Medium';

    ELSE
        RETURN 'Low';
    END IF;
END;
$$;
```

Execute:

```
SELECT
    emp_name,
    salary_grade(salary)
FROM employees;
```

## **4.4 Stored procedure**

A stored procedure is invoked using `CALL`.

```
CREATE PROCEDURE increase_department_salary(
    dept_id INT,
    increment_amount NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE employees
    SET salary = salary + increment_amount
    WHERE department_id = dept_id;
END;
$$;
```

Execute:

```
CALL increase_department_salary(10, 5000);
```

All employees in department 10 receive a ₹5,000 increase.

### **Function vs Procedure**

| Feature             | Function                                        | Procedure                                                  |
| ------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| Invocation          | `SELECT` or another permitted SQL context       | `CALL`                                                     |
| Return value        | Required return definition, possibly `void`     | No function-style return value; can have output parameters |
| Used in SELECT      | Yes, when compatible                            | No                                                         |
| Transaction control | Cannot perform transaction control commands     | Can in supported invocation contexts                       |
| Common use          | Calculations, transformations, reusable queries | Operational database tasks and workflows                   |

### **Important PostgreSQL transaction detail**

A procedure can perform transaction control under supported conditions, such as a top-level `CALL` invocation that is not inside an explicit transaction block.

It cannot freely issue `COMMIT` or `ROLLBACK` in every context. For example, nested procedure invocation and exception-block restrictions matter.

**Interview takeaway:** A function returns a value and can participate in SQL expressions. A procedure is invoked with `CALL` and is suitable for procedural database operations. PostgreSQL procedures have more flexible transaction-control capabilities under specific conditions.

# **5. Triggers — automatically execute logic on database events**

A trigger executes a specified function when an event occurs on a table or view.

Common trigger events:

- INSERT
- UPDATE
- DELETE
- TRUNCATE (for statement-level triggers)

## **5.1 Real-world example: Automatically update updated_at**

Suppose you have:

```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(100),
    salary NUMERIC,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Whenever an employee record is updated, you want `updated_at` to change automatically.

Without a trigger, every application update would need to set it manually.

### **Step 1: Create the trigger function**

```
CREATE FUNCTION set_updated_at()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;
```

### **Step 2: Create the trigger**

```
CREATE TRIGGER trg_set_updated_at
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();
```

Now:

```
UPDATE employees
SET salary = 80000
WHERE emp_id = 101;
```

The timestamp is automatically updated.

## **5.2 Understand NEW and OLD**

These are special trigger records.

| Trigger record | Meaning                           |
| -------------- | --------------------------------- |
| `NEW`          | New row for INSERT or UPDATE      |
| `OLD`          | Existing row for UPDATE or DELETE |

For example:

```
UPDATE employees
SET salary = 80000
WHERE emp_id = 101;
```

Inside a row-level UPDATE trigger:

- `OLD.salary` = salary before update.
- `NEW.salary` = salary after update.

## **5.3 BEFORE vs AFTER triggers**

| BEFORE trigger                                             | AFTER trigger                                   |
| ---------------------------------------------------------- | ----------------------------------------------- |
| Executes before the triggering operation completes         | Executes after the operation                    |
| Can modify `NEW` for row-level INSERT/UPDATE               | Used for actions based on completed row changes |
| Can return `NEW` or `OLD`, or NULL to skip a row operation | Return value is ignored                         |

Example use cases:

- BEFORE: Validate or modify incoming data.
- AFTER: Record an audit entry after a change.

## **5.4 Create an audit table**

```
CREATE TABLE employee_audit (
    audit_id BIGSERIAL PRIMARY KEY,
    emp_id INT,
    old_salary NUMERIC,
    new_salary NUMERIC,
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Create an audit trigger function:

```
CREATE FUNCTION audit_salary_changes()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF OLD.salary IS DISTINCT FROM NEW.salary THEN

        INSERT INTO employee_audit (
            emp_id,
            old_salary,
            new_salary
        )
        VALUES (
            OLD.emp_id,
            OLD.salary,
            NEW.salary
        );

    END IF;

    RETURN NEW;
END;
$$;
```

Create the trigger:

```
CREATE TRIGGER trg_salary_audit
AFTER UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION audit_salary_changes();
```

Test:

```
UPDATE employees
SET salary = 90000
WHERE emp_id = 101;
```

Now the salary update and corresponding audit insertion occur as part of the same database transaction.

If the transaction rolls back, the audit insertion rolls back too.

### **Row-level vs Statement-level triggers**

| Row-level                            | Statement-level                         |
| ------------------------------------ | --------------------------------------- |
| Uses `FOR EACH ROW`                  | Uses `FOR EACH STATEMENT`               |
| Executes once per affected row       | Executes once per SQL statement         |
| Has access to NEW/OLD as appropriate | Does not have an individual NEW/OLD row |
| Useful for row auditing              | Useful for statement-level operations   |

Example:

```
CREATE TRIGGER trg_employee_statement
AFTER UPDATE ON employees
FOR EACH STATEMENT
EXECUTE FUNCTION some_trigger_function();
```

The trigger function must already exist and return `TRIGGER`.

### **Should you use triggers for all business logic?**

No.

Potential problems:

- Hidden side effects.
- Harder debugging.
- Additional write overhead.
- Unexpected interactions between triggers.
- Business logic becomes less visible to application developers.

Use them when automatic database-level enforcement or auditing is genuinely useful.

# **6. Cursors — processing query results row by row**

A cursor allows an application or procedural code to retrieve query results incrementally.

Imagine a table contains 10 million records.

You may not want to load every record into application memory at once.

A cursor lets you fetch results in batches or one row at a time.

## **6.1 Basic cursor example in PL/pgSQL**

```
DO $$
DECLARE
    employee_cursor CURSOR FOR
        SELECT emp_id, emp_name
        FROM employees;

    employee_record RECORD;
BEGIN
    OPEN employee_cursor;

    LOOP
        FETCH employee_cursor INTO employee_record;

        EXIT WHEN NOT FOUND;

        RAISE NOTICE 'Employee: %, %',
            employee_record.emp_id,
            employee_record.emp_name;
    END LOOP;

    CLOSE employee_cursor;
END;
$$;
```

Understand:

| Command     | Meaning                      |
| ----------- | ---------------------------- |
| `DECLARE`   | Define the cursor            |
| `OPEN`      | Execute and open its query   |
| `FETCH`     | Retrieve the next row        |
| `NOT FOUND` | Indicates no row was fetched |
| `CLOSE`     | Close the cursor             |

## **6.2 Cursor vs ordinary query**

Ordinary query:

```
SELECT *
FROM employees;
```

Returns a result set.

Cursor:

```
OPEN cursor
   ↓
FETCH first row
   ↓
Process row
   ↓
FETCH next row
   ↓
Process row
   ↓
CLOSE cursor
```

### **Important performance advice**

Don't use cursors just because a task involves millions of rows.

Often, set-based SQL is faster:

```
UPDATE employees
SET salary = salary * 1.10
WHERE department_id = 10;
```

This is generally preferable to fetching each employee and updating them individually.

Cursors are useful when:

- Sequential processing is required.
- You need incremental result retrieval.
- A complex procedural task cannot be expressed simply as set-based SQL.

For application-side batch processing, you may also use pagination or driver-supported streaming.

# **7. Table partitioning**

This is an important concept for large production databases.

Imagine an application stores 500 million transactions.

```
transactions (
    transaction_id,
    user_id,
    transaction_date,
    amount
);
```

Every query scans or searches the same large logical table.

One technique to improve manageability and potentially query performance is **partitioning**.

## **7.1 What is partitioning?**

Partitioning divides one logical table into smaller physical tables called partitions.

The application queries the parent table.

PostgreSQL routes rows and selects relevant partitions based on the partition key and partitioning strategy.

**Logical transactions table**

**transactions**

One logical table

**2026 Q1**

Jan–Mar

**2026 Q2**

Apr–Jun

**2026 Q3**

Jul–Sep

## **7.2 Range partitioning**

Useful for date-based tables.

```
CREATE TABLE transactions (
    transaction_id BIGINT,
    user_id BIGINT,
    transaction_date DATE NOT NULL,
    amount NUMERIC
)
PARTITION BY RANGE (transaction_date);
```

Create partitions:

```
CREATE TABLE transactions_2026_q1
PARTITION OF transactions
FOR VALUES FROM ('2026-01-01')
TO ('2026-04-01');
```

```
CREATE TABLE transactions_2026_q2
PARTITION OF transactions
FOR VALUES FROM ('2026-04-01')
TO ('2026-07-01');
```

The upper bound is exclusive.

So:

- January 1 belongs to Q1.
- March 31 belongs to Q1.
- April 1 belongs to Q2.

Insert into the parent:

```
INSERT INTO transactions
VALUES (
    1001,
    101,
    DATE '2026-02-15',
    5000
);
```

PostgreSQL routes the row to the correct partition.

## **7.3 Query partition pruning**

```
SELECT *
FROM transactions
WHERE transaction_date >= DATE '2026-01-01'
  AND transaction_date < DATE '2026-04-01';
```

PostgreSQL can eliminate partitions that cannot contain matching dates.

This is called **partition pruning**.

Without partition pruning, the engine may examine unnecessary partitions.

## **7.4 Types of partitioning**

| Type  | How data is divided              | Example           |
| ----- | -------------------------------- | ----------------- |
| RANGE | Values fall within ranges        | Transaction dates |
| LIST  | Values match discrete categories | Region or country |
| HASH  | Hash value determines partition  | User IDs          |

### **Range**

```
PARTITION BY RANGE (transaction_date)
```

### **List**

```
PARTITION BY LIST (region)
```

### **Hash**

```
PARTITION BY HASH (user_id)
```

## **7.5 Partitioning vs indexing**

These are not the same.

| Indexing                                  | Partitioning                                                    |
| ----------------------------------------- | --------------------------------------------------------------- |
| Creates a search structure                | Divides table storage into partitions                           |
| Helps locate rows within a table          | Helps organize large tables and eliminate irrelevant partitions |
| Generally applies to a table or partition | Operates at table-partition level                               |
| Useful for selective lookups              | Useful for data lifecycle and access patterns                   |

They can be used together.

### **Important PostgreSQL limitation**

For a partitioned table, a primary key or unique constraint defined at the parent level must include all partition key columns.

For example, if `transaction_date` is the partition key, this is not generally valid as a parent-level unique constraint:

```
PRIMARY KEY (transaction_id)
```

But this can be valid:

```
PRIMARY KEY (transaction_id, transaction_date)
```

This matters because PostgreSQL cannot enforce a global unique constraint on a partitioned table using an ordinary parent-level unique index that omits the partition key.

### **When should you partition?**

Good potential use cases:

- Very large historical transaction tables.
- Time-series data.
- High-volume log data.
- Efficient removal or archiving of old data.
- Queries that regularly filter by the partition key.

Partitioning is not automatically a performance improvement for every large table. Poor partition design can add complexity without meaningful benefits.

# **8. Connection pooling**

This is one of the most important concepts for backend developers.

Imagine your Flask/FastAPI application receives 1,000 requests per second.

Each request needs a PostgreSQL connection.

Should every request open a new database connection and close it afterward?

Generally, no.

Establishing database connections consumes resources and has overhead.

## **8.1 What is a database connection?**

A connection allows your application to communicate with the database.

For example:

```
FastAPI Application
       |
       | SQL query
       |
   PostgreSQL
```

A connection handles:

- Authentication.
- Session state.
- Transaction context.
- Query execution.
- Returning results.

Opening and closing connections for every request is often inefficient.

## **8.2 What is connection pooling?**

A connection pool maintains a reusable collection of database connections.

**Connection pooling architecture**

**Backend application**

Incoming HTTP requests

**Connection pool**

Manages reusable database connections

Connection 1

Connection 2

Connection 3

**PostgreSQL server**

How it works:

1. Application receives a request.
2. It asks the pool for a connection.
3. A free connection is allocated.
4. The query executes.
5. The application returns the connection to the pool.
6. Another request can reuse it.

## **8.3 What is PgBouncer?**

PgBouncer is a lightweight PostgreSQL connection pooler.

It sits between applications and PostgreSQL:

```
Applications
     |
     v
 PgBouncer
     |
     v
 PostgreSQL
```

It helps manage large numbers of client connections without requiring an equally large number of PostgreSQL server connections.

### **PgBouncer pooling modes**

| Mode                | Behaviour                                                                                     |
| ------------------- | --------------------------------------------------------------------------------------------- |
| Session pooling     | Connection stays assigned to a client for its entire session                                  |
| Transaction pooling | Connection is assigned only for a transaction                                                 |
| Statement pooling   | Connection is assigned for a single statement; multi-statement transactions are not supported |

**Transaction pooling** can be more resource-efficient, but it has important compatibility considerations.

For example, session-level settings and temporary tables may not behave as an application expects when the server connection is reassigned between transactions.

Modern PgBouncer versions also support protocol-level prepared statements under appropriate configuration, but this should be verified for the deployed version and setup.

### **Interview question: Why not create 10,000 connections for 10,000 requests?**

Because database connections consume server resources, including memory and process-management overhead. Too many connections can reduce throughput and increase contention.

Connection pooling reuses a controlled number of connections.

# **9. Prepared statements and SQL injection**

This is both a database and application security topic.

Suppose your application receives a username.

A dangerous implementation constructs SQL by concatenating strings:

```
username = input_from_userquery = (    "SELECT * FROM users "    + "WHERE username = '" + username + "'")
```

What if the user enters:

```
' OR '1'='1
```

The resulting SQL can change the meaning of the query.

This is called **SQL injection**.

## **9.1 Parameterized queries**

Instead of combining user input with SQL strings, pass parameters separately.

Python example using a PostgreSQL driver with `%s` parameter placeholders:

```
username = "parth"cursor.execute(    "SELECT * FROM users WHERE username = %s",    (username,))
```

The driver transmits the SQL statement and parameter values separately using its parameter-binding mechanism.

The input is treated as a value rather than arbitrary SQL syntax.

This is safer and is also the normal approach for database parameter binding.

## **9.2 PostgreSQL PREPARE and EXECUTE**

PostgreSQL also supports SQL-level prepared statements.

```
PREPARE get_employee (INT) AS
SELECT
    emp_id,
    emp_name,
    salary
FROM employees
WHERE emp_id = $1;
```

Execute:

```
EXECUTE get_employee(101);
```

Execute with another parameter:

```
EXECUTE get_employee(102);
```

Deallocate:

```
DEALLOCATE get_employee;
```

Advantages:

- Separates SQL structure from parameter values.
- Helps prevent SQL injection when used properly.
- Can reduce repeated parsing/planning overhead.
- May allow plan reuse.

An important nuance: PostgreSQL may choose between custom and generic plans for prepared statements. Reusing a prepared statement does not mean every execution necessarily uses one identical fixed execution plan.

### **Interview comparison**

| String concatenation       | Parameterized query                    |
| -------------------------- | -------------------------------------- |
| Input can alter SQL syntax | Input is bound as a value              |
| Vulnerable to injection    | Protects against value-based injection |
| Unsafe                     | Recommended                            |
| Can complicate quoting     | Driver handles parameter encoding      |

For table or column names, ordinary bind parameters generally cannot replace identifiers. Use safe allowlists and correct identifier-quoting mechanisms when dynamically selecting SQL identifiers.

# **10. Complete backend example: an inventory update**

Let's combine several concepts.

Requirements:

1. Accept product ID and quantity.
2. Insert the product if it doesn't exist.
3. Otherwise add to existing stock.
4. Record the updated timestamp.
5. Return the updated row.

```
INSERT INTO inventory (
    product_id,
    product_name,
    quantity,
    updated_at
)
VALUES (
    101,
    'Laptop',
    5,
    CURRENT_TIMESTAMP
)
ON CONFLICT (product_id)
DO UPDATE
SET
    quantity = inventory.quantity + EXCLUDED.quantity,
    updated_at = CURRENT_TIMESTAMP
RETURNING
    product_id,
    product_name,
    quantity,
    updated_at;
```

In a backend application, parameterize the incoming values instead of concatenating them into this SQL statement.

This single operation demonstrates:

- UPSERT.
- A unique constraint.
- Atomic conflict handling.
- `EXCLUDED`.
- `RETURNING`.
- Timestamp management.

It is a practical example of SQL supporting real application behaviour.

# **11. Interview quick revision**

| Topic                | Must remember                                                                |
| -------------------- | ---------------------------------------------------------------------------- |
| UPSERT               | `INSERT ... ON CONFLICT DO UPDATE`                                           |
| EXCLUDED             | Proposed row that conflicted                                                 |
| JSON vs JSONB        | Text representation vs decomposed binary representation                      |
| `->` vs `->>`        | JSON value vs text                                                           |
| GIN                  | Useful for indexing JSONB containment and supported operations               |
| View                 | Saved query definition                                                       |
| Materialized view    | Stored query results                                                         |
| Refresh concurrently | Requires an eligible unique index and an already populated materialized view |
| Function             | Returns a value or set of rows                                               |
| Procedure            | Invoked using `CALL`                                                         |
| Trigger              | Automatically fires on database events                                       |
| NEW / OLD            | New and previous row values in row-level triggers                            |
| Cursor               | Incremental retrieval of result rows                                         |
| Partitioning         | Divides one logical table into smaller partitions                            |
| Connection pool      | Reuses database connections                                                  |
| Prepared statement   | Separates SQL structure and input values                                     |

# **12. Knowledge checkpoint**

**Advanced PostgreSQL assessment**

8 questions

**Q1. What is the purpose of EXCLUDED in PostgreSQL UPSERT?**

The existing row

The proposed row that encountered a conflict

The deleted row

The table being updated

**Q2. Which JSONB operator returns a value as text?**

->

->>

\#>

@>

**Q3. Does a regular PostgreSQL view store its result rows?**

Yes, always

No, it stores a query definition

Only when it has an index

Only when it uses GROUP BY

**Q4. How do you invoke a PostgreSQL stored procedure?**

SELECT

EXECUTE only

CALL

FETCH

**Q5. What is NEW in a row-level UPDATE trigger?**

The old row

The new row after the proposed update

The table name

The transaction ID

**Q6. What is the primary benefit of connection pooling?**

Eliminates SQL joins

Reuses database connections

Automatically creates indexes

Stores query results permanently

**Q7. Which partitioning type is suitable for separating data by date ranges?**

HASH

LIST

RANGE

GIN

**Q8. What is the safest normal approach to handling user-supplied SQL values?**

String concatenation

Parameterized queries

Manual quote replacement

Dynamic SQL without validation

Check answers

# **Where we are now**

**SQL learning roadmap**

Core concepts and advanced topics covered in lessons

Broad coverage

SQL fundamentals and querying

Covered

Joins, subqueries and CTEs

Covered

Window and analytical functions

Covered

Database design and normalization

Covered

Indexes and query optimization

Covered

Transactions, isolation and locking

Covered

OA and advanced interview patterns

Covered

Advanced PostgreSQL and backend SQL

Covered in this lesson

Final interview question bank

Next

## **Next: Part 4 — SQL and DBMS Final Interview Preparation**

We will move into:

1. **100 frequently asked SQL and DBMS interview questions**, organised by difficulty and topic.
2. SQL query-writing questions with at least two solutions where appropriate.
3. Differences interviewers commonly test: DELETE vs TRUNCATE vs DROP, primary key vs unique key, clustered vs nonclustered index, RANK vs DENSE_RANK vs ROW_NUMBER, and more.
4. Real backend scenarios: banking transactions, ticket booking, inventory management, race conditions and deadlocks.
5. A final mock interview with coding and theory.

One important distinction: you've now covered a broad SQL curriculum, but mastery still needs to be verified through independent coding, actual PostgreSQL execution and timed assessments. The next phase will focus on that verification.

