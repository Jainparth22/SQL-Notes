# **SQL Interview Mastery — Part 2**

## **Query Optimization and Database Internals**

Advanced

SDE / Backend interviews

PostgreSQL

One of the most important differences between a beginner and an experienced backend engineer is this:

A beginner asks, **"Does my SQL query return the correct result?"**

An experienced engineer also asks:

- How many rows does the database scan?
- Which index does it use?
- How many times is a table accessed?
- Does the query perform an unnecessary sort?
- Is the join algorithm efficient?
- Why is the query slow even though it uses an index?

Let's learn this from first principles.

# **1. How a database executes a SQL query**

Consider:

```
SELECT
    emp_id,
    emp_name,
    salary
FROM employees
WHERE department_id = 10
  AND salary > 50000
ORDER BY salary DESC
LIMIT 5;
```

You write SQL describing *what* data you want.

The database decides *how* to retrieve it.

This is the difference between declarative SQL and query execution.

## **The query processing pipeline**

1. **Parsing**

   Checks SQL syntax, identifies tables, columns and expressions.

   Example: Is `employees` a valid relation? Does `salary` exist?
2. **Semantic analysis**

   Checks types, permissions, functions and whether referenced objects are valid.
3. **Query rewriting**

   Applies transformations to the query, such as expanding views and processing rewrite rules.
4. **Query optimization**

   Considers possible execution strategies, estimates their costs and chooses a plan.
5. **Execution**

   Executes the selected plan and produces the result rows.

### **What is a query optimizer?**

A query optimizer is the component that tries to find an efficient execution plan.

For example, suppose a table contains 10 million employees.

There are two broad ways to find employees in department 10.

**Strategy A — Scan the entire table**

```
Read all 10 million rows
        ↓
Check department_id = 10
        ↓
Keep matching rows
```

**Strategy B — Use an index**

```
Search index for department_id = 10
        ↓
Find matching row locations
        ↓
Fetch matching rows
```

The second may be much faster, but not always.

The optimizer estimates which strategy is cheaper based on available statistics, table size, selectivity, indexes and other factors.

**Interview answer:** A query optimizer evaluates alternative execution plans using estimated costs and chooses a plan expected to retrieve the requested data efficiently.

# **2. What is an execution plan?**

An execution plan describes how the database intends to execute your query.

In PostgreSQL, the most important command for examining it is:

```
EXPLAIN
SELECT *
FROM employees
WHERE department_id = 10;
```

Example output:

```
Seq Scan on employees
  (cost=0.00..35.50 rows=10 width=100)
  Filter: (department_id = 10)
```

Don't worry about understanding every number immediately.

Let's break it down.

| Field      | Meaning                                    |
| ---------- | ------------------------------------------ |
| `Seq Scan` | Sequential table scan                      |
| `cost`     | Estimated startup and total execution cost |
| `rows`     | Estimated number of output rows            |
| `width`    | Estimated average row width in bytes       |
| `Filter`   | Condition applied to scanned rows          |

### **Understanding cost**

```
cost=0.00..35.50
```

There are two numbers:

- `0.00`: Estimated startup cost.
- `35.50`: Estimated total cost.

These are planner cost units, not milliseconds.

PostgreSQL uses cost estimates to compare plans. You should not interpret `35.50` as 35.50 ms.

### **EXPLAIN vs EXPLAIN ANALYZE**

This is an extremely common interview question.

**EXPLAIN:**

```
EXPLAIN
SELECT *
FROM employees
WHERE salary > 50000;
```

Shows the estimated plan without executing the SELECT query.

**EXPLAIN ANALYZE:**

```
EXPLAIN ANALYZE
SELECT *
FROM employees
WHERE salary > 50000;
```

Actually executes the query and reports the observed execution statistics.

A more useful diagnostic command is:

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM employees
WHERE department_id = 10;
```

This gives you:

- Estimated rows.
- Actual rows.
- Actual execution time.
- Number of loops.
- Buffer hits and reads.

Important

`EXPLAIN ANALYZE` executes the statement. Be particularly careful with `UPDATE`, `DELETE` and other statements that modify data. For controlled testing, use a transaction and roll it back:

```
BEGIN;

EXPLAIN ANALYZE
DELETE FROM employees
WHERE department_id = 10;

ROLLBACK;
```

This does not protect you from every possible side effect, such as external effects caused by triggers, so use a suitable test environment.

## **Reading actual versus estimated rows**

Example:

```
Seq Scan on employees
  (cost=0.00..200.00 rows=100 width=80)
  (actual time=0.020..15.600 rows=50000 loops=1)
  Filter: (department_id = 10)
  Rows Removed by Filter: 950000
```

Interpretation:

- Estimated rows: 100.
- Actual rows: 50,000.
- Rows removed: 950,000.
- Actual time: approximately 15.6 ms for the scan.

The optimizer estimated 100 rows but actually received 50,000.

This is a large cardinality estimation error.

It might cause the optimizer to select an inefficient join strategy.

**Another important detail:** When a plan node runs multiple times, `actual rows` and `actual time` are generally reported per loop. Multiply rows by `loops` to understand the approximate total number of rows produced across executions of that node.

# **3. Sequential scan vs Index scan vs Bitmap scan**

These are three important PostgreSQL execution-plan concepts.

## **A. Sequential scan**

A sequential scan reads a table's pages in sequence and examines its rows.

Example:

```
SELECT *
FROM employees;
```

Potential plan:

```
Seq Scan on employees
```

Imagine:

**Table with 8 data pages**

**1**

**2**

**3**

**4**

**5**

**6**

**7**

**8**

The scan examines all table pages.

Sequential scans are often efficient when:

- The table is small.
- Most rows must be returned.
- The filter is not selective.
- Reading the table sequentially is cheaper than following an index and fetching many individual rows.

**A sequential scan is not automatically bad.**

This is an important interview point.

## **B. Index scan**

An index scan uses an index to find matching row locations, then fetches the required table rows where necessary.

For example:

```
CREATE INDEX idx_employees_department
ON employees(department_id);
```

Then:

```
SELECT *
FROM employees
WHERE department_id = 10;
```

Potential plan:

```
Index Scan using idx_employees_department on employees
  Index Cond: (department_id = 10)
```

Conceptually:

**Index-based lookup**

**B-Tree index**

Find department_id = 10

**Matching row locations**

**Heap table**

Fetch required row data

PostgreSQL calls the main table storage a heap.

Unlike a clustered primary-key index in MySQL InnoDB, PostgreSQL's ordinary B-Tree index does not automatically determine the physical order of the table.

## **C. Bitmap index scan and bitmap heap scan**

PostgreSQL can use a bitmap strategy when there are many matching rows or when several indexes can be combined efficiently.

Example:

```
SELECT *
FROM employees
WHERE department_id = 10
   OR city = 'Bhopal';
```

Potential plan:

```
Bitmap Heap Scan on employees
  Recheck Cond: (...)
  -> BitmapOr
       -> Bitmap Index Scan on idx_employees_department
       -> Bitmap Index Scan on idx_employees_city
```

Conceptually:

1. Scan an index to find matching tuple locations.
2. Build a bitmap of those locations.
3. Combine bitmap results if needed.
4. Fetch table pages more efficiently.

### **Comparison**

| Scan type        | How it works                                                           | Typical use                              |
| ---------------- | ---------------------------------------------------------------------- | ---------------------------------------- |
| Sequential scan  | Reads table pages                                                      | Small tables or many matching rows       |
| Index scan       | Uses index and fetches rows                                            | Selective lookups                        |
| Bitmap heap scan | Collects matching locations before fetching pages                      | Multiple or moderately selective matches |
| Index-only scan  | Retrieves required data from index when visibility information permits | Queries covered by an index              |

**Interview trap:** An index does not guarantee an index scan. PostgreSQL may correctly choose a sequential scan even when an index exists.

# **4. Indexes — how they actually help**

An index is an additional data structure that helps the database locate rows without scanning the entire table.

Think of a book.

Without an index:

\- You might inspect every page to find a topic.

With an index:

- You locate the topic in the index.
- You jump to the relevant page.

A database index follows a similar principle.

## **B-Tree index**

PostgreSQL's default index type is B-Tree.

It is useful for:

- Equality searches.
- Range searches.
- Sorting.
- Comparisons such as `<`, `<=`, `>`, `>=`.
- Prefix-oriented lookups under compatible collation and operator-class conditions.

Example:

```
CREATE INDEX idx_employees_salary
ON employees(salary);
```

Useful queries:

```
SELECT *
FROM employees
WHERE salary = 75000;
```

```
SELECT *
FROM employees
WHERE salary BETWEEN 50000 AND 80000;
```

```
SELECT *
FROM employees
ORDER BY salary;
```

An index can support these operations, although the optimizer may still prefer another plan.

## **Why not index every column?**

Indexes are not free.

Every index:

- Requires disk storage.
- Must be maintained during inserts.
- Must be maintained during relevant updates.
- Adds work during deletes.
- Can increase backup and maintenance overhead.

Example:

Suppose an application inserts 10,000 employee records daily.

If you have many indexes, each insert may require additional index maintenance.

**Interview answer:** Indexes improve certain read operations at the cost of storage and write overhead. They should be designed around actual query patterns rather than created indiscriminately.

# **5. Composite indexes — a very important interview topic**

A composite index contains multiple columns.

Example:

```
CREATE INDEX idx_emp_dept_salary
ON employees(department_id, salary);
```

This index stores entries ordered primarily by department ID and then by salary within the department.

Consider the following queries.

**Query A:**

```
SELECT *
FROM employees
WHERE department_id = 10;
```

This can use the index.

**Query B:**

```
SELECT *
FROM employees
WHERE department_id = 10
  AND salary > 60000;
```

This is a particularly good match for the index:

```
(department_id, salary)
```

**Query C:**

```
SELECT *
FROM employees
WHERE salary > 60000;
```

The index is generally less effective for this query because the leading column is not constrained.

This is commonly explained by the **leftmost-prefix principle**.

**Composite index: (department_id, salary)**

Leading column

**department_id**

Organizes index entries first

Second column

**salary**

Orders entries within each department

The leading column strongly influences which searches and ordering operations the index can support efficiently.

### **What should be the order of columns?**

Consider these two indexes:

```
CREATE INDEX idx1
ON employees(department_id, salary);
```

```
CREATE INDEX idx2
ON employees(salary, department_id);
```

Which one should you choose?

Look at the workload.

If your common query is:

```
WHERE department_id = 10
  AND salary > 60000
```

then:

```
(department_id, salary)
```

is a natural choice.

A common heuristic is:

1. Put columns used in equality conditions first.
2. Put the range condition after them.
3. Consider the ordering requirements and other frequent query patterns.

This is a heuristic, not an absolute rule. Examine real query plans and workload patterns.

### **A subtle point: equality before range**

For this query:

```
SELECT *
FROM employees
WHERE department_id = 10
  AND salary BETWEEN 50000 AND 80000
ORDER BY salary;
```

A suitable index is:

```
CREATE INDEX idx_department_salary
ON employees(department_id, salary);
```

The database can navigate to department 10 and then traverse a relevant salary range.

# **6. Covering indexes and Index Only Scan**

Suppose your frequent query is:

```
SELECT emp_id, salary
FROM employees
WHERE department_id = 10;
```

You have:

```
CREATE INDEX idx_department
ON employees(department_id);
```

The index helps identify matching employees, but fetching `emp_id` and `salary` may still require visiting the heap table.

PostgreSQL supports `INCLUDE`:

```
CREATE INDEX idx_department_covering
ON employees(department_id)
INCLUDE (emp_id, salary);
```

Now the index stores:

- Key column: `department_id`
- Included columns: `emp_id`, `salary`

This can allow an **Index Only Scan**.

Example possible plan:

```
Index Only Scan using idx_department_covering on employees
  Index Cond: (department_id = 10)
  Heap Fetches: 0
```

But there is an important PostgreSQL detail.

An Index Only Scan is only able to avoid heap visits when the visibility map indicates that the relevant heap pages are all-visible. Otherwise, heap fetches may still occur.

Thus, having all requested columns in the index does not guarantee zero heap access.

### **When should you use INCLUDE?**

When:

- The query is frequent.
- The key columns are used for filtering or navigation.
- Additional output columns can be stored as included attributes.
- The storage and write overhead is justified.

Do not blindly include every column.

# **7. SARGable queries — make conditions index-friendly**

SARGable stands for *Search ARGument ABLE*.

A SARGable condition is structured so that the database can efficiently use an index to locate matching values.

Consider this index:

```
CREATE INDEX idx_emp_salary
ON employees(salary);
```

### **Example A — Index-friendly**

```
SELECT *
FROM employees
WHERE salary > 50000;
```

The indexed column is compared directly with a constant.

### **Example B — Potentially problematic**

```
SELECT *
FROM employees
WHERE salary * 12 > 600000;
```

The database must evaluate the expression for rows. The normal index on `salary` may not be useful for navigating this condition.

Rewrite:

```
SELECT *
FROM employees
WHERE salary > 50000;
```

### **Example C — Function applied to a date column**

Suppose you want employees hired in November 2026.

An ordinary index exists:

```
CREATE INDEX idx_hire_date
ON employees(hire_date);
```

Less index-friendly:

```
SELECT *
FROM employees
WHERE EXTRACT(MONTH FROM hire_date) = 11
  AND EXTRACT(YEAR FROM hire_date) = 2026;
```

Better:

```
SELECT *
FROM employees
WHERE hire_date >= DATE '2026-11-01'
  AND hire_date < DATE '2026-12-01';
```

The second form creates a range directly on the indexed column.

### **Example D — Leading wildcard**

```
WHERE emp_name LIKE '%parth%';
```

A conventional B-Tree index normally cannot efficiently navigate this arbitrary substring search.

PostgreSQL offers other options, including trigram indexes using `pg_trgm` and GIN or GiST.

**Interview takeaway:** Avoid unnecessarily applying functions or calculations to indexed columns in filtering predicates. Prefer equivalent range or direct-column conditions, and verify with `EXPLAIN`.

# **8. Partial indexes, expression indexes and unique indexes**

These are useful PostgreSQL features that go beyond ordinary B-Tree indexes.

## **A. Partial index**

Suppose 90% of your orders are completed, but your application frequently searches for pending orders.

```
CREATE INDEX idx_pending_orders
ON orders(order_date)
WHERE status = 'Pending';
```

This index only contains entries for pending orders.

Useful query:

```
SELECT *
FROM orders
WHERE status = 'Pending'
  AND order_date >= DATE '2026-11-01';
```

Advantages:

- Smaller index.
- Less maintenance compared with indexing all rows in the same way.
- Efficient for a frequently queried subset.

The query predicate must be compatible with the index predicate for PostgreSQL to use it.

## **B. Expression index**

Suppose email addresses should be searched case-insensitively.

```
CREATE INDEX idx_email_lower
ON users(LOWER(email));
```

Then:

```
SELECT *
FROM users
WHERE LOWER(email) = 'parth@example.com';
```

The indexed expression matches the query expression.

## **C. Unique index**

```
CREATE UNIQUE INDEX idx_unique_email
ON users(email);
```

Prevents duplicate non-NULL email values under ordinary unique-index semantics.

For PostgreSQL, ordinary unique indexes allow multiple NULLs by default. PostgreSQL also supports `NULLS NOT DISTINCT` when NULL values should be treated as equal for uniqueness.

# **9. Query optimization — a real backend example**

Suppose you need the five newest orders for customer 101.

A query:

```
SELECT *
FROM orders
WHERE customer_id = 101
ORDER BY order_date DESC
LIMIT 5;
```

A suitable index:

```
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date DESC);
```

Why is this useful?

The index is organized by customer, then by descending order date.

The database can locate that customer's records in the appropriate ordering and may avoid a separate sort.

But the actual result depends on the table, data distribution, statistics and planner decisions.

### **What if you query many customers?**

```
SELECT *
FROM orders
ORDER BY order_date DESC
LIMIT 5;
```

The previous composite index may not help as effectively because the query doesn't constrain `customer_id`.

A different index could be relevant:

```
CREATE INDEX idx_orders_date
ON orders(order_date DESC);
```

**Key principle:** An index should support a specific workload. There is no universally best index.

# **10. OFFSET pagination vs keyset pagination**

This is a very practical backend interview topic.

Suppose an API returns orders in pages of 20.

### **Approach A — OFFSET pagination**

```
SELECT *
FROM orders
ORDER BY order_id
LIMIT 20 OFFSET 100000;
```

This asks the database to skip many preceding rows.

As the offset grows, this can become increasingly expensive.

### **Approach B — Keyset pagination**

Suppose the last order ID from the previous page was 100000.

```
SELECT *
FROM orders
WHERE order_id > 100000
ORDER BY order_id
LIMIT 20;
```

This uses the last seen key to continue from that point.

With a suitable index, it can avoid scanning and discarding all preceding records.

### **But what if sorting by timestamp?**

Use a stable, unique ordering:

```
SELECT *
FROM orders
WHERE (order_date, order_id)
        < (TIMESTAMP '2026-11-20 12:00:00', 100000)
ORDER BY order_date DESC, order_id DESC
LIMIT 20;
```

A corresponding index:

```
CREATE INDEX idx_orders_cursor
ON orders(order_date DESC, order_id DESC);
```

This is useful for cursor-based pagination in production APIs.

### **Comparison**

| OFFSET pagination                          | Keyset pagination                                                 |
| ------------------------------------------ | ----------------------------------------------------------------- |
| Easy to implement                          | Slightly more complex                                             |
| Useful for jumping to a page number        | Useful for next/previous navigation                               |
| Large offsets can be expensive             | Usually scales better with a suitable index                       |
| Concurrent changes can shift page contents | More stable cursor progression, but not a full snapshot guarantee |

# **11. Practical query optimization checklist**

Whenever you receive a slow-query debugging task, follow this process:

1. **Understand the requirement**

   Confirm the expected rows, filters, joins, order and output columns.
2. **Measure first**

   Run `EXPLAIN (ANALYZE, BUFFERS)` in a safe environment.
3. **Check row estimates**

   Compare estimated rows and actual rows at each important plan node.
4. **Inspect expensive operations**

   Look at large sequential scans, nested-loop iterations, sorts, spills and repeated scans.
5. **Inspect indexes**

   Identify whether an existing index matches the filtering, joining and sorting pattern.
6. **Improve the SQL or index design**

   Remove unnecessary work, fix join cardinality, use appropriate indexes and avoid non-SARGable predicates.
7. **Measure again**

   Compare execution time, buffer activity and result correctness.

### **Bonus: Common execution-plan problems**

| Plan symptom                       | Possible issue               | What to investigate                                  |
| ---------------------------------- | ---------------------------- | ---------------------------------------------------- |
| Large Seq Scan                     | Many rows examined           | Selectivity and index usefulness                     |
| Huge estimated/actual row mismatch | Poor cardinality estimate    | Statistics, correlated predicates, data distribution |
| Nested Loop with many loops        | Repeated inner-table lookups | Join cardinality and inner index                     |
| Sort taking significant time       | Large sort input             | Index-supported ordering, memory                     |
| Hash operation with disk batches   | Memory pressure              | Join size, `work_mem`, query shape                   |
| Repeated sequential scans          | Repeated table access        | Join strategy and query design                       |

Do not automatically replace every sequential scan or nested loop. The plan must be judged against the actual workload.

# **12. Interview checkpoint**

**Optimization knowledge test**

6 questions

**Q1. What does EXPLAIN ANALYZE do?**

Only parses SQL

Executes the query and reports actual execution statistics

Creates indexes automatically

Always forces an index scan

**Q2. Why might PostgreSQL choose a sequential scan even when an index exists?**

Indexes are never used for filtering

The index is corrupted

Scanning much of the table sequentially may be cheaper

PostgreSQL cannot estimate costs

**Q3. For WHERE department_id = 10 AND salary > 50000, which composite index is a natural choice?**

(salary, department_id) always

(department_id, salary)

(emp_name)

(hire_date)

**Q4. What can a PostgreSQL INCLUDE column help enable?**

Index Only Scan

Foreign key cascade

Automatic table partitioning

Transaction rollback

**Q5. Which query is typically more index-friendly with an ordinary B-Tree index on hire_date?**

EXTRACT(YEAR FROM hire_date) = 2026

hire_date >= DATE '2026-01-01' AND hire_date < DATE '2027-01-01'

TO_CHAR(hire_date, 'YYYY') = '2026'

hire_date::text LIKE '2026%'

**Q6. What is a keyset-pagination advantage?**

It guarantees a complete database snapshot

It always supports arbitrary page jumps efficiently

It can avoid skipping a large OFFSET when supported by an index

It does not need ORDER BY

Evaluate my answers

## **What remains in your SQL course?**

We've now covered the core SQL learning roadmap, advanced OA patterns, logical query processing, and practical query optimization.

The next topics are:

**Part 3 — Advanced PostgreSQL and backend SQL**

1. `INSERT ... ON CONFLICT` — UPSERT.
2. JSON and JSONB operations.
3. Views vs materialized views.
4. Stored procedures and functions.
5. Triggers and their real-world uses.
6. Cursors and batch processing.
7. Table partitioning and when it helps.
8. Connection pooling and prepared statements.

**Part 4 — Final interview preparation**

1. 100 high-frequency SQL and DBMS interview questions.
2. SQL coding problems with multiple solutions.
3. Real backend database scenarios.
4. Final timed mock interview.
5. Weak-topic revision based on your assessment results.

**My advice:** Spend particular attention on `EXPLAIN ANALYZE`, composite indexes, `LEFT JOIN` semantics and transaction behaviour. These are highly practical topics for backend engineering interviews, not just theoretical DBMS questions.

