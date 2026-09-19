# **Lesson 9 — Complete Advanced DBMS: Indexing, Query Optimisation, Transactions and Backend Engineering**



Complete four-part module

SDE 1 / SDE 2 interviews



We'll go deeper than definitions. The objective is to understand how a production database works, why queries become slow, how concurrent operations are handled, and how to design reliable backend operations.

**Four sections of this module**

A

**Indexing**

B-Tree, B+ Tree, composite, partial, covering indexes

B

**Query optimisation**

EXPLAIN, execution plans, joins, performance

C

**Transactions and concurrency**

ACID, MVCC, isolation, locking, deadlocks

D

**Backend engineering applications**

Banking, inventory, booking, payment systems

We'll use PostgreSQL for practical examples.

# **SECTION A — INDEXING**

## **A1. How database indexing actually works**

Consider a table containing 10 million employees.

```
SELECT *
FROM employees
WHERE emp_id = 9876543;
```

Without a suitable index, PostgreSQL may perform a sequential scan.

Conceptually:

```
Read page 1
    ↓
Read page 2
    ↓
Read page 3
    ↓
...
    ↓
Find employee 9876543
```

It may have to inspect a large portion of the table.

With a B-Tree index:

```
                 Root page
                     |
              Internal page
                     |
                Leaf page
                     |
              Matching entry
                     |
                Heap tuple
                     |
              Employee record
```

The database follows the index structure to locate the matching entry and then retrieves the corresponding table record, when necessary.

**Important:** A database works with pages, not individual rows loaded one by one from the physical disk. An index reduces the number of pages that may need to be accessed.

## **A2. Database pages**

Databases organise data into storage pages.

A page contains:

- Page header.
- Stored tuples or index entries.
- Free space and associated metadata.

PostgreSQL uses 8 KB pages by default in standard builds.

Why does this matter?

Disk and memory access happen in blocks or pages. A good index structure tries to minimise the number of page accesses needed to find information.

This is one reason B-Trees are useful in database systems.

## **A3. B-Tree — internal working**

Must understand for interviews

A B-Tree is a self-balancing search tree with multiple keys per node.

Unlike a simple binary search tree, a B-Tree can have many children per node.

Conceptual example:

## **B-Tree search example**

**[30 | 60]**

Root

**[10 | 20]**

**[40 | 50]**

**[70 | 80]**

Simplified illustration: each internal node directs searches into an appropriate key range.

Suppose you search for 50:

1. Start at the root.
2. 50 is between 30 and 60.
3. Move to the middle child.
4. Find 50.

A balanced tree keeps the height relatively small.

For a balanced B-Tree:

\\[ T(N)=O(\log N) \\]

This is the theoretical search complexity in terms of tree operations.

Real database performance also depends on page reads, caching, row visibility and storage access.

### **Why does a database prefer B-Trees over an ordinary binary search tree?**

An ordinary binary search tree typically has two children per node.

A B-Tree stores many keys per node. Database nodes can occupy entire pages, reducing tree height and the number of page accesses.

### **B-Tree vs B+ Tree**

| B-Tree                                                           | B+ Tree                                            |
| ---------------------------------------------------------------- | -------------------------------------------------- |
| Keys and records may be stored at internal and leaf nodes        | Actual indexed entries are stored at leaves        |
| Internal nodes may contain data references                       | Internal nodes primarily guide searches            |
| Leaf nodes may or may not be linked, depending on implementation | Leaf nodes are typically linked                    |
| Supports search and range operations                             | Particularly convenient for sequential range scans |

**Interview detail:** PostgreSQL's B-Tree implementation is a balanced B-link-style tree. Learn the B+ Tree model for general database interview explanations, but know that database implementations differ.

## **A4. Types of indexes**

You should understand these six.

| Index type | Primary purpose                                                         |
| ---------- | ----------------------------------------------------------------------- |
| B-Tree     | Equality, range and ordered queries                                     |
| Hash       | Equality lookups                                                        |
| GIN        | Arrays, JSONB and full-text search                                      |
| GiST       | Geometric, range and extensible search                                  |
| BRIN       | Large tables with physically correlated data                            |
| SP-GiST    | Partitioned search structures such as certain spatial and text patterns |

### **1. B-Tree**

```
CREATE INDEX idx_employee_salary
ON employees(salary);
```

Useful for:

```
WHERE salary = 75000
```

```
WHERE salary > 70000
```

```
ORDER BY salary DESC
```

### **2. Hash index**

```
CREATE INDEX idx_employee_hash
ON employees USING HASH(emp_id);
```

Useful for equality comparisons.

```
SELECT *
FROM employees
WHERE emp_id = 101;
```

Hash indexes do not provide ordinary ordered range scanning.

B-Tree is usually the default choice unless you have a measured reason to use another index type.

### **3. GIN index**

GIN stands for Generalized Inverted Index.

It is useful when a single row contains multiple searchable components, such as:

- Arrays.
- JSONB documents.
- Full-text search tokens.

Example:

```
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    attributes JSONB
);
```

A row might contain:

```
{
  "brand": "Acme",
  "color": "blue",
  "size": "large"
}
```

Create an index:

```
CREATE INDEX idx_product_attributes
ON products USING GIN(attributes);
```

A supported JSONB containment query:

```
SELECT *
FROM products
WHERE attributes @> '{"color": "blue"}';
```

### **4. GiST index**

Generalized Search Tree.

Useful for search methods involving:

- Geometric data.
- Spatial searches.
- Range operators.
- Certain similarity operations.

For example, with PostGIS, GiST indexes are widely used for spatial queries.

### **5. BRIN index**

BRIN stands for Block Range Index.

It stores summaries for ranges of physical table blocks rather than individual row references.

Consider a huge table of timestamped sensor measurements inserted in chronological order.

```
CREATE INDEX idx_sensor_timestamp
ON sensor_data USING BRIN(recorded_at);
```

When the physical order of data correlates with timestamp values, BRIN can help identify relevant block ranges with very little index storage.

It is not equivalent to B-Tree and can require scanning many rows within matching ranges.

### **6. SP-GiST**

Space-Partitioned Generalized Search Tree.

Useful for certain data structures and partitioning strategies, such as:

- Quadtrees.
- Radix trees.
- Some spatial data.
- Text-prefix search structures.

For most entry-level SQL interviews, knowing the purpose of this index type is sufficient.

## **A5. Composite indexes**

A composite index contains multiple columns.

```
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date);
```

Imagine this table:

| order_id | customer_id | order_date | amount |
| -------- | ----------- | ---------- | ------ |
| 1        | 101         | Jan 01     | 500    |
| 2        | 102         | Jan 02     | 800    |
| 3        | 101         | Jan 04     | 300    |
| 4        | 103         | Jan 05     | 700    |

The index is ordered by:

1. customer_id
2. order_date within customer_id

### **Which queries can benefit?**

| Query                                           | Suitability                     |
| ----------------------------------------------- | ------------------------------- |
| `WHERE customer_id = 101`                       | Good                            |
| `WHERE customer_id = 101 AND order_date >= ...` | Good                            |
| `WHERE customer_id = 101 ORDER BY order_date`   | Good                            |
| `WHERE order_date = ...`                        | Not a good leading-prefix match |

### **Why does the order matter?**

Consider two indexes:

```
CREATE INDEX idx1
ON orders(customer_id, order_date);
```

```
CREATE INDEX idx2
ON orders(order_date, customer_id);
```

They are not interchangeable.

The first is designed primarily around customer ID. The second is designed primarily around order date.

**Practical rule:** Design the column order around your actual filter, join and sort patterns, then verify using EXPLAIN.

## **A6. Partial indexes**

A partial index indexes only rows satisfying a condition.

Example:

```
CREATE INDEX idx_pending_orders
ON orders(order_id)
WHERE status = 'Pending';
```

This index contains only pending orders.

Query:

```
SELECT *
FROM orders
WHERE status = 'Pending';
```

Benefits:

- Smaller index.
- Less index maintenance for rows outside the predicate.
- Useful for frequently accessed subsets.

Another example:

```
CREATE INDEX idx_unprocessed_events
ON events(created_at)
WHERE processed = FALSE;
```

Useful when a background worker repeatedly searches for unprocessed events.

## **A7. Expression indexes**

Suppose users' emails are stored with inconsistent capitalization.

```
SELECT *
FROM users
WHERE LOWER(email) = 'parth@gmail.com';
```

Create:

```
CREATE INDEX idx_email_lower
ON users(LOWER(email));
```

Now an appropriate lookup can use the expression index.

This is useful for case-insensitive search.

## **A8. Covering indexes and INCLUDE**

Suppose your query is:

```
SELECT name, salary
FROM employees
WHERE department_id = 101;
```

Create:

```
CREATE INDEX idx_department_cover
ON employees(department_id)
INCLUDE (name, salary);
```

The index stores:

- department_id as the search key.
- name and salary as included payload.

PostgreSQL may be able to use an index-only scan, avoiding heap access when visibility information permits it.

Remember:

`INCLUDE` columns are not search keys in the same way as the indexed key columns. They are stored to cover retrieval requirements.

## **A9. Unique indexes**

```
CREATE UNIQUE INDEX idx_users_email
ON users(email);
```

Ensures uniqueness of indexed non-NULL values under standard PostgreSQL unique-index semantics.

Useful for:

- Usernames.
- Emails.
- External transaction IDs.
- Idempotency keys.

For NULL values, PostgreSQL permits multiple NULLs by default in a normal unique index. `NULLS NOT DISTINCT` changes this behaviour.

## **A10. Clustered vs non-clustered indexes**

This depends on the DBMS.

**MySQL InnoDB:**

- Primary key is the clustered index.
- Table records are organised around the primary key.
- Secondary indexes contain primary-key values to identify the corresponding records.

**PostgreSQL:**

- Indexes are separate from the heap table.
- A primary key creates a unique index, but doesn't automatically organise the heap in primary-key order.
- `CLUSTER` can reorder the table according to an index once, but later writes do not automatically maintain that physical order.

Example:

```
CLUSTER employees USING employees_pkey;
```

## **A11. When NOT to create an index**

Do not create indexes blindly.

Potential disadvantages:

1. Extra storage.
2. Slower INSERT operations.
3. More expensive UPDATE operations for indexed columns.
4. More expensive DELETE operations.
5. Additional maintenance.
6. Cache and memory pressure.
7. More work for the query optimiser.

Indexes are particularly useful when they support frequent, selective queries.

A low-selectivity column such as a Boolean column may not justify a normal full index by itself, although a partial index can still be useful.

### **Section A — Interview questions**

| Interview question                                | Expected answer                                      |
| ------------------------------------------------- | ---------------------------------------------------- |
| Why use indexes?                                  | Reduce lookup work                                   |
| Why B-Tree?                                       | Balanced ordered access path                         |
| B-Tree vs Hash?                                   | Ordered/range support vs equality-oriented access    |
| What is a composite index?                        | Index on multiple columns                            |
| What is leftmost-prefix behaviour?                | Leading indexed columns determine lookup suitability |
| What is a covering index?                         | Index contains required query columns                |
| What is a partial index?                          | Index contains only qualifying rows                  |
| What is an expression index?                      | Index built using an expression                      |
| Why do indexes slow writes?                       | Additional index maintenance                         |
| Does PostgreSQL primary key mean clustered table? | No                                                   |

# **SECTION B — QUERY OPTIMISATION**

## **B1. How a SQL query is executed**

Consider:

```
SELECT
    c.customer_name,
    SUM(o.amount) AS total_spent
FROM customers c
JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.status = 'Completed'
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.amount) > 10000
ORDER BY total_spent DESC;
```

The database does not simply read this SQL from top to bottom and execute each line.

Conceptually, query processing involves:

1. **Parsing**

   Check SQL syntax and construct a parse representation.
2. **Semantic analysis**

   Resolve tables, columns, functions, types and permissions.
3. **Query rewriting**

   Apply supported transformations, rules and view expansion.
4. **Query optimisation**

   Consider alternative access paths, join orders and execution strategies.
5. **Plan selection**

   Choose a plan based on estimated costs and statistics.
6. **Execution**

   Execute the selected plan and return results.

The component selecting a plan is the query optimiser.

## **B2. What is a query execution plan?**

An execution plan describes how the database intends to retrieve and process the requested data.

Let's start with:

```
SELECT *
FROM employees
WHERE salary > 80000;
```

Run:

```
EXPLAIN
SELECT *
FROM employees
WHERE salary > 80000;
```

PostgreSQL might return:

```
Seq Scan on employees
  (cost=0.00..35.50 rows=200 width=80)
  Filter: (salary > 80000)
```

This is an illustrative plan, not a guaranteed output.

Interpretation:

- Seq Scan: PostgreSQL scans the table.
- cost: estimated planner cost.
- rows: estimated output row count.
- width: estimated output row width.
- Filter: predicate applied to rows.

## **B3. EXPLAIN vs EXPLAIN ANALYZE**

| EXPLAIN                      | EXPLAIN ANALYZE                     |
| ---------------------------- | ----------------------------------- |
| Displays planned execution   | Executes and measures the statement |
| Estimated rows and costs     | Actual rows and timings             |
| Does not run ordinary SELECT | Runs the query                      |
| Useful for plan inspection   | Useful for performance diagnosis    |

Example:

```
EXPLAIN ANALYZE
SELECT *
FROM employees
WHERE salary > 80000;
```

For more detailed PostgreSQL output:

```
EXPLAIN (
    ANALYZE,
    BUFFERS,
    VERBOSE
)
SELECT *
FROM employees
WHERE salary > 80000;
```

### **Important EXPLAIN fields**

| Field          | Meaning                                |
| -------------- | -------------------------------------- |
| Startup cost   | Estimated cost before first output row |
| Total cost     | Estimated total cost                   |
| Plan rows      | Estimated output rows                  |
| Actual rows    | Observed output rows                   |
| Actual time    | Measured execution time                |
| Loops          | Number of times a plan node executes   |
| Shared hit     | Shared buffer pages found in cache     |
| Shared read    | Shared buffer pages read into memory   |
| Planning Time  | Plan generation time                   |
| Execution Time | Statement execution time               |

**Critical:** Cost is not milliseconds. It is an internal estimate used for comparing execution plans.

### **How to debug a poor plan**

Look for:

1. Estimated rows versus actual rows.
2. Unexpected sequential scans on large tables.
3. Large sort operations.
4. Expensive nested-loop iterations.
5. High buffer reads.
6. Repeated loops over large result sets.
7. Rows removed by filters.
8. Unnecessary table access.

A big difference between estimated and actual rows can indicate inaccurate statistics or data-distribution assumptions.

## **B4. ANALYZE and database statistics**

The optimiser needs statistics about the data.

Example:

```
ANALYZE employees;
```

PostgreSQL collects statistics such as:

- Estimated number of rows.
- Common values.
- Value distribution.
- NULL fraction.
- Histograms for certain columns.
- Correlations.

Why?

Imagine a table of 10 million users.

Suppose only 100 users have `status = 'Inactive'`.

An index may be very effective.

But if 9 million users are active and the query requests active users, scanning most of the table may be cheaper than going through an index and fetching almost every row.

The optimiser uses statistics to estimate these costs.

**Interview question: Why might an existing index not be used?**

Possible answers:

- The table is small.
- The predicate selects a large percentage of rows.
- Statistics are stale or inaccurate.
- A function prevents direct use of an ordinary index.
- The index doesn't match the query's leading columns.
- A sequential scan is estimated to be cheaper.

## **B5. Sequential Scan, Index Scan and Bitmap Scan**

### **Sequential Scan**

Reads the table's pages.

```
Seq Scan on employees
```

Good for:

- Small tables.
- Queries requiring many rows.
- Cases where index access would be more expensive.

### **Index Scan**

Uses the index to find matching records and fetches corresponding heap tuples.

```
Index Scan using idx_employee_salary
```

Good for:

- Selective predicates.
- Small matching result sets.

### **Index Only Scan**

Retrieves query-required values from the index without visiting the heap, when visibility information permits.

```
Index Only Scan using idx_department_cover
```

### **Bitmap Index Scan + Bitmap Heap Scan**

PostgreSQL can:

1. Use an index to find matching tuple locations.
2. Build a bitmap of matching locations.
3. Fetch relevant heap pages in a more efficient order.

Useful when multiple table rows match.

## **B6. Join algorithms**

Very important for backend interviews

A database can use different algorithms to execute a JOIN.

The three important ones are:

1. Nested Loop Join.
2. Hash Join.
3. Merge Join.

### **1. Nested Loop Join**

Concept:

For every row in the outer input, find matching rows in the inner input.

Pseudocode:

```
for employee in employees:    for department in departments:        if employee.department_id == department.department_id:            output(employee, department)
```

Illustration:

**Nested Loop Join**

**Outer rows**

**Inner search**

Repeat the inner lookup for each outer row.

When useful:

- One input is small.
- The other input has an appropriate index.
- The join is selective.

With an index on the inner side, the cost can be attractive.

Without an index, naive repeated scanning can be expensive.

### **2. Hash Join**

The database generally:

1. Builds a hash table using one input.
2. Reads rows from the other input.
3. Probes the hash table for matching keys.

Illustration:

**Hash Join**

**Build input**

Create hash table using join key

**Hash table**

**Probe input**

Find matching keys

Useful for equality joins with sufficiently large inputs.

PostgreSQL can also use batch processing and disk spill when the hash table does not fit in memory.

### **3. Merge Join**

Both input streams are sorted on the join key.

The database walks through the sorted inputs, matching equal keys.

```
Input A: 10, 20, 30, 40
Input B: 10, 30, 50

Matches: 10, 30
```

Useful when:

- Inputs are already suitably ordered.
- An appropriate index provides ordered access.
- Sorting costs are acceptable.

### **Comparison**

| Algorithm   | Main idea                  | Suitable situation                          |
| ----------- | -------------------------- | ------------------------------------------- |
| Nested Loop | Repeated matching lookup   | Small outer side or indexed inner side      |
| Hash Join   | Build and probe hash table | Large equality joins                        |
| Merge Join  | Match sorted inputs        | Ordered inputs and suitable join conditions |

**Interview tip:** Don't say PostgreSQL always picks Hash Join for large tables. The planner chooses based on estimated cost and available plan alternatives.

## **B7. Query optimisation strategies**

Now let's learn the practical techniques.

### **Technique 1: Avoid SELECT \***

Poor:

```
SELECT *
FROM employees
WHERE department_id = 101;
```

Better:

```
SELECT emp_id, name, salary
FROM employees
WHERE department_id = 101;
```

Benefits:

- Less unnecessary data transfer.
- Lower memory usage.
- Potential for covering/index-only scans.
- Easier maintenance when the schema changes.

### **Technique 2: Make predicates index-friendly**

Less suitable for an ordinary date index:

```
WHERE EXTRACT(YEAR FROM order_date) = 2026
```

Better:

```
WHERE order_date >= DATE '2026-01-01'
  AND order_date < DATE '2027-01-01'
```

### **Technique 3: Use EXISTS for existence checks**

Suppose you only need customers who have orders.

```
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

This expresses existence without requiring you to retrieve the order details.

It also avoids accidental row multiplication from a one-to-many JOIN.

### **Technique 4: Avoid unnecessary DISTINCT**

Poorly designed JOINs can multiply rows.

A developer may add DISTINCT just to hide the duplicates.

Instead, understand the relationship and use the correct join or EXISTS pattern.

### **Technique 5: Filter before expensive processing where logically valid**

```
SELECT department_id, AVG(salary)
FROM employees
WHERE status = 'Active'
GROUP BY department_id;
```

The WHERE filter logically restricts input rows before grouping.

The optimiser may also push predicates through suitable query structures automatically.

### **Technique 6: Avoid unnecessary repeated calculations**

Use a CTE or derived table when it improves query clarity and helps structure calculations.

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
WHERE avg_salary > 70000;
```

Do not assume every CTE is physically materialised or every CTE improves performance. Inspect the plan.

### **Technique 7: Choose pagination carefully**

Offset pagination:

```
SELECT *
FROM orders
ORDER BY order_id
LIMIT 20 OFFSET 100000;
```

The database may need to traverse many rows before reaching that page.

Keyset pagination:

```
SELECT *
FROM orders
WHERE order_id > 100000
ORDER BY order_id
LIMIT 20;
```

For a non-unique sort column, use a unique tiebreaker:

```
SELECT *
FROM orders
WHERE (created_at, order_id) >
      (TIMESTAMP '2026-11-01 12:00:00', 500)
ORDER BY created_at, order_id
LIMIT 20;
```

This is useful for large feeds and transaction history.

### **Technique 8: Avoid N+1 queries**

Bad application approach:

```
Query all customers
Query their orders individually
Query their orders individually
Query their orders individually
...
```

Better:

```
SELECT
    c.customer_id,
    c.name,
    o.order_id,
    o.amount
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id;
```

Alternatively, batch-fetch related records.

### **Technique 9: Use appropriate data types**

For financial values:

Prefer:

```
NUMERIC(12, 2)
```

rather than binary floating-point types when exact decimal arithmetic is required.

Use DATE for date-only information and appropriate timestamp types for events.

Correct types can simplify comparisons and avoid unnecessary conversions.

### **Technique 10: Keep transactions short**

Long transactions can:

- Hold locks longer.
- Increase contention.
- Retain old MVCC snapshots.
- Delay cleanup of obsolete row versions.
- Increase the possibility of deadlocks.

Do database work efficiently and avoid waiting for unnecessary external calls while holding a transaction open.

## **B8. CTE materialisation and optimisation**

PostgreSQL supports:

```
WITH department_stats AS MATERIALIZED (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT *
FROM department_stats;
```

MATERIALIZED tells PostgreSQL to calculate and store the CTE result for reuse within the statement.

You can also specify:

```
WITH department_stats AS NOT MATERIALIZED (
    SELECT department_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT *
FROM department_stats;
```

`NOT MATERIALIZED` allows eligible CTE references to be folded into the surrounding query.

Why does it matter?

- Materialisation may avoid repeating an expensive calculation.
- It may also prevent predicate pushdown and cause unnecessary intermediate work.
- Inlining can expose more optimisation opportunities.
- The best choice depends on the query.

Modern PostgreSQL can inline many non-recursive CTEs automatically, so don't assume that every CTE is an optimisation barrier.

### **Section B — Interview questions**

| Question                             | Expected understanding                                 |
| ------------------------------------ | ------------------------------------------------------ |
| What is query optimisation?          | Selecting an efficient execution plan                  |
| What is EXPLAIN?                     | Displays planned execution                             |
| What is EXPLAIN ANALYZE?             | Executes and measures a statement                      |
| Why sequential scan?                 | May be cheaper for large result sets or small tables   |
| What is Hash Join?                   | Hash-based matching                                    |
| What is Merge Join?                  | Matching ordered inputs                                |
| What is Nested Loop?                 | Repeated lookups for input rows                        |
| What is SARGable?                    | Index-friendly search predicate                        |
| What is N+1?                         | One primary query followed by repeated related queries |
| What is keyset pagination?           | Pagination using the last seen sort key                |
| Why update statistics?               | Better row-count and cost estimates                    |
| Does CTE guarantee faster execution? | No                                                     |

# **SECTION C — TRANSACTIONS AND CONCURRENCY**

Critical for backend / SDE interviews

This is arguably the most important part of database internals for backend engineering.

We will understand not only what transactions are, but what happens when multiple requests try to modify the same data simultaneously.

## **C1. What is a transaction?**

A transaction is a logical unit of database work consisting of one or more operations.

Example: Transfer ₹5,000 from Aman to Rahul.

Initial balances:

| Account | Balance |
| ------- | ------- |
| Aman    | ₹20,000 |
| Rahul   | ₹10,000 |

Operations:

```
1. Deduct ₹5,000 from Aman.
2. Add ₹5,000 to Rahul.
```

Expected result:

| Account | Balance |
| ------- | ------- |
| Aman    | ₹15,000 |
| Rahul   | ₹15,000 |

What happens if the first operation succeeds but the second fails?

Without transaction management, money could disappear from Aman's account.

With a transaction, both operations are committed together or rolled back together.

### **SQL transaction syntax**

```
BEGIN;

UPDATE accounts
SET balance = balance - 5000
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 5000
WHERE account_id = 2;

COMMIT;
```

To undo uncommitted changes:

```
ROLLBACK;
```

## **C2. Autocommit**

What happens when you execute a query without BEGIN?

In PostgreSQL, statements executed outside an explicit transaction block run in their own implicit transactions.

For example:

```
UPDATE employees
SET salary = 80000
WHERE emp_id = 1;
```

The statement is committed automatically if it succeeds.

You can group multiple statements together using:

```
BEGIN;
-- multiple SQL operations
COMMIT;
```

This is important because two independent SQL statements are not automatically one atomic business transaction.

## **C3. ACID — in-depth understanding**

Frequently asked in every DBMS interview

### **A — Atomicity**

All operations in a transaction succeed together or are rolled back together.

Example:

```
BEGIN;

UPDATE accounts
SET balance = balance - 5000
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 5000
WHERE account_id = 2;

COMMIT;
```

If the second operation fails, the application must roll back the transaction rather than commit the first operation independently.

Atomicity is supported by transaction management and recovery mechanisms.

### **C — Consistency**

The database moves from one valid state to another valid state, provided the transaction and database rules are correctly defined.

Example:

```
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    balance NUMERIC(12, 2)
        CHECK (balance >= 0)
);
```

The CHECK constraint prevents a committed negative balance.

Another invariant:

\\[ \sum \text{Account balances} \\]

should remain unchanged during an internal transfer, assuming there are no fees or other adjustments.

Consistency is not magic. The database enforces declared constraints, and the application or database logic must correctly enforce the remaining business invariants.

### **I — Isolation**

Concurrent transactions should not interfere in ways that violate the guarantees of the selected isolation level.

We will cover the isolation levels and anomalies in detail below.

### **D — Durability**

Once the database confirms a successful commit, the transaction must survive a crash according to the database's durability configuration.

PostgreSQL uses Write-Ahead Logging (WAL).

Before modified data pages need to be written to the main data files, the corresponding WAL records are written to durable storage according to the commit and configuration guarantees.

This supports recovery after a crash.

### **Interview-ready answer**

ACID represents Atomicity, Consistency, Isolation and Durability. Atomicity ensures all-or-nothing execution, Consistency maintains database invariants, Isolation controls concurrent execution, and Durability preserves committed changes against failures.

## **C4. Write-Ahead Logging (WAL)**

Important DBMS internals

WAL stands for Write-Ahead Logging.

It is a logging mechanism used by PostgreSQL to support crash recovery and durability.

Imagine updating:

```
UPDATE accounts
SET balance = 15000
WHERE account_id = 1;
```

PostgreSQL does not need to immediately write the modified table page to its final location on disk before acknowledging the transaction.

Conceptually:

**Write-Ahead Logging**

**Transaction modifies data**

**WAL records generated**

Describe the changes needed for recovery

**WAL flushed according to commit settings**

**COMMIT acknowledged**

Durable under configured guarantees

Data pages may be written to their main files later. WAL enables redo-based recovery after crashes.

### **Why is WAL useful?**

- Crash recovery.
- Durability.
- Replication.
- Point-in-time recovery when combined with suitable WAL archiving and base backups.

### **WAL vs Data Files**

| WAL                                    | Data files                                              |
| -------------------------------------- | ------------------------------------------------------- |
| Records database changes for recovery  | Store actual database pages                             |
| Written ahead of data-page persistence | Can be written later                                    |
| Important for crash recovery           | Hold the database's persistent table and index contents |

**Interview question: Why does PostgreSQL use WAL instead of immediately writing every modified data page?**

Writing a sequential WAL stream is generally more efficient than requiring every modified data page to be synchronously written before committing. WAL also provides the information needed to redo changes after crashes.

## **C5. Isolation levels — complete understanding**

The four standard isolation levels are:

1. READ UNCOMMITTED
2. READ COMMITTED
3. REPEATABLE READ
4. SERIALIZABLE

First, understand the problems that isolation is designed to control.

### **1. Dirty Read**

Transaction A changes a salary from 50,000 to 80,000 but hasn't committed.

Transaction B reads 80,000.

Transaction A rolls back.

Transaction B has read a value that was never committed.

That is a dirty read.

### **2. Non-Repeatable Read**

Transaction A reads:

```
Employee salary = 50000
```

Transaction B:

```
UPDATE employees
SET salary = 80000
WHERE emp_id = 1;

COMMIT;
```

Transaction A reads again and sees 80,000.

The same row produced different values in one transaction.

### **3. Phantom Read**

Transaction A:

```
SELECT *
FROM employees
WHERE salary > 70000;
```

It receives 10 records.

Transaction B inserts another qualifying employee and commits.

Transaction A repeats the query and receives 11 records.

The set of matching rows has changed.

### **4. Lost Update**

Two transactions read the same old value and calculate updates independently.

One transaction overwrites another's modification.

Example:

```
Initial inventory = 10

T1 reads 10
T2 reads 10

T1 writes 8
T2 writes 7

Final = 7
Expected = 5
```

The second update has lost the first transaction's intended deduction.

### **5. Write Skew — advanced topic**

This is particularly important for understanding snapshot isolation.

Suppose a hospital must always have at least one doctor on call.

Two doctors are on call:

| Doctor | On call |
| ------ | ------- |
| Aman   | TRUE    |
| Rahul  | TRUE    |

Two concurrent transactions run:

- T1 checks that Rahul is on call, then switches Aman off.
- T2 checks that Aman is on call, then switches Rahul off.

Both transactions read the original state.

Both updates affect different rows, so they may both commit under snapshot isolation.

Result:

| Doctor | On call |
| ------ | ------- |
| Aman   | FALSE   |
| Rahul  | FALSE   |

The business invariant is violated.

This is write skew: concurrent transactions make individually valid decisions using a shared snapshot, but their combined result is invalid.

Serializable isolation is designed to prevent this type of serialization anomaly.

## **C6. Isolation level comparison**

| Isolation        | Dirty reads                | Non-repeatable reads | Phantom reads    |
| ---------------- | -------------------------- | -------------------- | ---------------- |
| Read Uncommitted | Possible in standard model | Possible             | Possible         |
| Read Committed   | Prevented                  | Possible             | Possible         |
| Repeatable Read  | Prevented                  | Prevented            | Standard permits |
| Serializable     | Prevented                  | Prevented            | Prevented        |

### **PostgreSQL implementation**

This distinction matters in interviews:

| PostgreSQL level | Behaviour                                                              |
| ---------------- | ---------------------------------------------------------------------- |
| READ UNCOMMITTED | Behaves as READ COMMITTED                                              |
| READ COMMITTED   | New statement snapshot for each statement                              |
| REPEATABLE READ  | Transaction-level snapshot isolation; prevents phantom reads           |
| SERIALIZABLE     | Adds Serializable Snapshot Isolation checks and may abort transactions |

PostgreSQL REPEATABLE READ is stronger than the minimum standard SQL guarantee regarding phantom reads.

However, REPEATABLE READ can still permit write-skew-style serialization anomalies.

SERIALIZABLE may produce serialization failures. Your application needs to retry the entire transaction when such a failure occurs.

### **Setting isolation levels**

```
BEGIN TRANSACTION
ISOLATION LEVEL READ COMMITTED;

SELECT *
FROM accounts;

COMMIT;
```

Repeatable Read:

```
BEGIN TRANSACTION
ISOLATION LEVEL REPEATABLE READ;

SELECT *
FROM accounts;

COMMIT;
```

Serializable:

```
BEGIN TRANSACTION
ISOLATION LEVEL SERIALIZABLE;

SELECT *
FROM accounts;

COMMIT;
```

### **How do you choose?**

| Situation                                         | Potential choice                     |
| ------------------------------------------------- | ------------------------------------ |
| Normal web application reads and writes           | READ COMMITTED                       |
| Consistent multi-statement analytical transaction | REPEATABLE READ                      |
| Complex concurrent business invariants            | SERIALIZABLE or explicit locking     |
| Critical inventory changes                        | Atomic conditional UPDATE or locking |

The right level depends on the data invariant and the concurrency requirements, not simply on whether the application is important.

## **C7. MVCC — Multi-Version Concurrency Control**

Important for PostgreSQL and backend engineering

MVCC allows transactions to work with different visible versions of the same row.

Suppose an employee's salary is initially 50,000.

Transaction A starts and reads the employee.

Transaction B changes salary to 80,000 and commits.

Under an appropriate transaction snapshot, A can continue seeing the older committed version while a new transaction sees 80,000.

**MVCC visualisation**

**Version 1**

# **₹50,000**

Older committed version

**Transaction B updates**

# **₹80,000**

Creates a new row version

**Transaction A**

May continue seeing ₹50,000 under its snapshot.

**New transaction**

Can see ₹80,000 after commit.

### **PostgreSQL-specific details**

PostgreSQL stores tuple visibility information, including transaction IDs associated with row creation and deletion/update.

Updates generally create a new tuple version rather than modifying the existing version in place in the MVCC sense.

Dead or obsolete tuple versions can remain physically present until cleanup.

This is why PostgreSQL needs VACUUM and autovacuum.

### **What is VACUUM?**

VACUUM reclaims space occupied by dead tuples so that it can be reused by the database.

```
VACUUM employees;
```

Update planner statistics:

```
ANALYZE employees;
```

Both together:

```
VACUUM ANALYZE employees;
```

Important:

- VACUUM generally makes dead-tuple space reusable; it does not normally shrink the file and return all space to the operating system.
- VACUUM FULL rewrites the table, requires a stronger lock and can return space to the operating system.
- Long-running transactions can delay cleanup because older versions might still be visible to their snapshots.

### **Interview answer**

MVCC maintains multiple versions of data so transactions can read consistent snapshots while concurrent writes happen. In PostgreSQL, tuple visibility and transaction snapshots determine which row versions are visible.

## **C8. Locking — complete understanding**

Locks coordinate conflicting database operations.

### **Shared vs Exclusive locks**

Conceptually:

| Lock       | Behaviour                                   |
| ---------- | ------------------------------------------- |
| Shared     | Compatible with other readers               |
| Exclusive  | Restricts conflicting access                |
| Row lock   | Protects selected rows                      |
| Table lock | Controls conflicting table-level operations |

PostgreSQL has specific lock modes, so the shared/exclusive terminology should be understood conceptually rather than treated as a complete description of every PostgreSQL lock.

### **PostgreSQL row locking**

```
BEGIN;

SELECT *
FROM accounts
WHERE account_id = 101
FOR UPDATE;

-- Perform modifications

COMMIT;
```

`FOR UPDATE` obtains a row-level lock that conflicts with other operations attempting incompatible row locks or updates.

Normal SELECT statements can generally still read the last committed row version because PostgreSQL uses MVCC.

### **Other useful clauses**

```
SELECT *
FROM accounts
WHERE account_id = 101
FOR NO KEY UPDATE;
```

```
SELECT *
FROM accounts
WHERE account_id = 101
FOR SHARE;
```

```
SELECT *
FROM accounts
WHERE account_id = 101
FOR KEY SHARE;
```

There are also:

`NOWAIT`:

```
SELECT *
FROM accounts
WHERE account_id = 101
FOR UPDATE NOWAIT;
```

It returns an error instead of waiting for a conflicting lock.

`SKIP LOCKED`:

```
SELECT *
FROM jobs
WHERE status = 'Pending'
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

It skips rows that are already locked by other transactions.

This is particularly useful for database-backed job queues.

## **C9. Pessimistic vs Optimistic locking**

### **Pessimistic locking**

Assume conflicts may happen and acquire locks before performing modifications.

```
BEGIN;

SELECT *
FROM products
WHERE product_id = 101
FOR UPDATE;

UPDATE products
SET price = 1200
WHERE product_id = 101;

COMMIT;
```

Other transactions attempting conflicting modifications may wait.

### **Optimistic locking**

Assume conflicts are relatively uncommon.

Use a version number.

Initial row:

| product_id | price | version |
| ---------- | ----- | ------- |
| 101        | 1000  | 1       |

Application reads version 1.

It then performs:

```
UPDATE products
SET price = 1200,
    version = version + 1
WHERE product_id = 101
  AND version = 1;
```

If one row is updated, success.

If zero rows are updated, a concurrent change occurred or the record didn't exist.

The application should inspect the reason and decide whether to retry or report a conflict.

### **Comparison**

| Pessimistic                            | Optimistic                       |
| -------------------------------------- | -------------------------------- |
| Locks before operation                 | Validates during update          |
| Can cause blocking                     | Can cause version conflicts      |
| Suitable for high-contention resources | Suitable when conflicts are rare |
| Lock release required                  | Version validation required      |

**Real-world example:**

- Last available product in inventory: pessimistic locking or an atomic conditional update.
- Editing a user profile with occasional conflicts: optimistic locking.

## **C10. Deadlocks**

A deadlock occurs when transactions wait in a cycle for locks held by one another.

Example:

```
Transaction A:
Lock account 1
Wait for account 2

Transaction B:
Lock account 2
Wait for account 1
```

Neither can continue.

### **How databases resolve deadlocks**

Databases detect the cycle and abort one of the transactions.

The aborted transaction receives an error, and the application can retry if appropriate.

### **Prevention**

1. Acquire locks in a consistent order.
2. Keep transactions short.
3. Avoid unnecessary locks.
4. Avoid user interaction or slow network calls inside transactions.
5. Create appropriate indexes so operations can access targeted rows efficiently.
6. Retry deadlock-victim transactions where safe.

### **Interview question**

**What is the difference between deadlock and blocking?**

Blocking occurs when one transaction waits for another to release a conflicting resource. A deadlock is a circular waiting condition that cannot resolve without aborting at least one transaction.

# **SECTION C — INTERVIEW CHECKLIST**

Make sure you can explain these without reading:

| Topic                  | Required understanding                      |
| ---------------------- | ------------------------------------------- |
| ACID                   | All four properties and a practical example |
| WAL                    | Crash recovery and durability               |
| Dirty read             | Reading uncommitted data                    |
| Non-repeatable read    | Same row changes between reads              |
| Phantom read           | Matching row set changes                    |
| Lost update            | Concurrent overwrite                        |
| Write skew             | Snapshot-based invariant violation          |
| Isolation levels       | Guarantees and tradeoffs                    |
| MVCC                   | Multiple visible row versions               |
| VACUUM                 | Cleanup of obsolete PostgreSQL tuples       |
| Shared/exclusive locks | Conflicting access                          |
| FOR UPDATE             | Pessimistic row locking                     |
| Optimistic locking     | Version-based validation                    |
| Deadlock               | Circular waiting                            |
| Serializable           | Strong isolation with possible retries      |

# **SECTION D — REAL-WORLD BACKEND ENGINEERING APPLICATIONS**

Most useful for practical SDE interviews

This section connects everything from A, B and C.

A technical interviewer may give you a real-world situation and ask:

- How would you prevent two users from purchasing the same last item?
- How would you implement a money transfer?
- How would you prevent duplicate payments?
- How would you prevent double booking?
- How would you process thousands of background jobs?
- How would you maintain data consistency across multiple services?

Let's solve these from a database and backend engineering perspective.

## **D1. Banking system — money transfer**

**Interview problem:**

Aman wants to transfer ₹5,000 to Rahul. How would you implement the transfer safely?

Database:

```
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    holder_name VARCHAR(100),
    balance NUMERIC(12, 2) NOT NULL
        CHECK (balance >= 0)
);
```

Initial data:

| account_id | holder_name | balance |
| ---------- | ----------- | ------- |
| 1          | Aman        | 20000   |
| 2          | Rahul       | 10000   |

### **Step 1: Begin transaction**

```
BEGIN;
```

### **Step 2: Debit Aman's account conditionally**

```
UPDATE accounts
SET balance = balance - 5000
WHERE account_id = 1
  AND balance >= 5000
RETURNING account_id, balance;
```

Why use a conditional UPDATE?

Because the balance check and deduction happen as one database operation.

You avoid the unsafe pattern of:

1. Reading the balance.
2. Checking it in application memory.
3. Updating it later without revalidation.

### **Step 3: Credit Rahul**

```
UPDATE accounts
SET balance = balance + 5000
WHERE account_id = 2
RETURNING account_id, balance;
```

### **Step 4: Commit**

```
COMMIT;
```

### **What if something fails?**

Suppose the debit succeeds but the credit operation fails.

```
ROLLBACK;
```

The earlier debit is reversed.

### **Complete application-level algorithm**

```
def transfer_money(source_id, target_id, amount):    if amount <= 0:        raise ValueError("Amount must be positive")    begin_transaction()    try:        debit = execute("""            UPDATE accounts            SET balance = balance - %s            WHERE account_id = %s              AND balance >= %s            RETURNING account_id        """, (amount, source_id, amount))        if not debit:            raise Exception("Insufficient funds or invalid account")        credit = execute("""            UPDATE accounts            SET balance = balance + %s            WHERE account_id = %s            RETURNING account_id        """, (amount, target_id))        if not credit:            raise Exception("Target account not found")        commit_transaction()    except Exception:        rollback_transaction()        raise
```

Illustrative pseudocode: begin_transaction, execute, commit_transaction and rollback_transaction represent your database driver's transaction API.

### **What concepts did we use?**

| Concept            | Application                     |
| ------------------ | ------------------------------- |
| Transaction        | Debit and credit together       |
| Atomicity          | Both succeed or neither commits |
| Consistency        | Balance cannot become negative  |
| Isolation          | Coordinate concurrent transfers |
| Durability         | Preserve confirmed transfers    |
| Conditional UPDATE | Prevent overdrawing             |
| Rollback           | Undo failed transfers           |

### **Advanced consideration: Lock ordering**

Suppose two transactions transfer money in opposite directions.

- T1 transfers A → B.
- T2 transfers B → A.

Both may try to lock accounts in different orders.

To reduce deadlocks, acquire account locks in a consistent account-ID order.

For example:

```
BEGIN;

SELECT *
FROM accounts
WHERE account_id IN (1, 2)
ORDER BY account_id
FOR UPDATE;

-- Validate balances and perform debit/credit

COMMIT;
```

This is useful when you need to validate and modify several account records together.

**Interview answer:** I would perform the transfer within a single transaction, use conditional updates or row locks to prevent overdrawing, validate both accounts, and commit only after both operations succeed. I would also use a consistent lock order and implement safe retries.

## **D2. Inventory management — prevent overselling**

Classic backend interview problem

Problem:

An e-commerce platform has one product remaining.

Two customers attempt to purchase it simultaneously.

Inventory:

| product_id | quantity |
| ---------- | -------- |
| 101        | 1        |

### **The incorrect implementation**

```
quantity = get_inventory(product_id)if quantity > 0:    update_inventory(product_id, quantity - 1)
```

What's wrong?

Both requests may read quantity = 1.

Both conclude that stock is available.

Both try to purchase the product.

This is a race condition.

### **Solution 1: Atomic conditional UPDATE**

```
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity >= 1
RETURNING quantity;
```

Interpretation:

- One returned row means success.
- Zero returned rows means stock is insufficient or the product doesn't exist.

The database coordinates concurrent modifications of the same row.

### **Solution 2: Pessimistic locking**

```
BEGIN;

SELECT quantity
FROM inventory
WHERE product_id = 101
FOR UPDATE;

-- Application checks the returned quantity.

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity >= 1;

COMMIT;
```

A conflicting transaction must wait for the row lock.

### **Solution 3: Optimistic concurrency control**

You can add a version column:

```
ALTER TABLE inventory
ADD COLUMN version INT NOT NULL DEFAULT 1;
```

Then:

```
UPDATE inventory
SET quantity = quantity - 1,
    version = version + 1
WHERE product_id = 101
  AND quantity >= 1
  AND version = 1;
```

If zero rows are updated, the application knows the expected version was no longer current or the stock condition failed.

For repeated operations, the application retrieves the latest state and handles retries.

### **Which approach should you choose?**

| Situation                                    | Approach                                      |
| -------------------------------------------- | --------------------------------------------- |
| Simple stock decrement                       | Atomic conditional UPDATE                     |
| Multiple rows require coordinated validation | Pessimistic locking                           |
| Low-conflict business records                | Optimistic locking                            |
| High-contention inventory                    | Carefully designed locking and retry strategy |

My preferred starting point for a simple stock decrement is an atomic conditional UPDATE. It is simpler and avoids unnecessary read-then-write logic.

## **D3. Ticket booking — prevent double booking**

Common system design + DBMS question

Imagine an event has 100 seats.

Two users try to book seat A10 simultaneously.

How can we prevent both from getting the same seat?

### **Step 1: Database schema**

```
CREATE TABLE seats (
    seat_id INT PRIMARY KEY,
    event_id INT NOT NULL,
    seat_number VARCHAR(10) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'Available',

    UNIQUE(event_id, seat_number)
);
```

### **Step 2: Reservations table**

```
CREATE TABLE reservations (
    reservation_id INT PRIMARY KEY,
    seat_id INT NOT NULL REFERENCES seats(seat_id),
    customer_id INT NOT NULL,
    booked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### **Step 3: Atomic seat acquisition**

One approach is a conditional update:

```
UPDATE seats
SET status = 'Booked'
WHERE event_id = 101
  AND seat_number = 'A10'
  AND status = 'Available'
RETURNING seat_id;
```

Only one competing request can successfully change that available seat to Booked.

Then insert the reservation in the same transaction:

```
BEGIN;

-- Perform the conditional seat update.
-- Application checks that exactly one row was returned.

INSERT INTO reservations (
    reservation_id,
    seat_id,
    customer_id
)
VALUES (1001, 501, 25);

COMMIT;
```

If the reservation insert fails, roll back the seat update.

### **Alternative: UNIQUE constraint**

You can also model active reservations with a unique constraint.

```
CREATE UNIQUE INDEX idx_active_seat_booking
ON reservations(seat_id)
WHERE status = 'Confirmed';
```

This guarantees that two confirmed reservations cannot own the same seat.

The status and uniqueness logic must match the business rules, particularly when cancellations and temporary holds are allowed.

### **What if we have a 10-minute payment window?**

This introduces a booking state machine:

```
AVAILABLE
    |
    v
HELD
    |
    v
PAYMENT_PENDING
    |
    +----------> CONFIRMED
    |
    +----------> EXPIRED
```

You need:

- Hold expiration time.
- Reservation status.
- A mechanism to expire abandoned holds.
- Safe concurrency control.
- Payment confirmation handling.

**Interview lesson:** Never depend only on frontend validation. The database must enforce the critical booking invariant.

## **D4. E-commerce order processing**

Suppose a customer places an order containing three products.

The backend needs to:

1. Validate stock.
2. Decrease inventory.
3. Create an order.
4. Create order items.
5. Record payment state.

These database changes should be coordinated.

### **Suggested transaction structure**

```
BEGIN;

-- 1. Lock required inventory rows.
SELECT product_id, quantity
FROM inventory
WHERE product_id IN (101, 102, 103)
ORDER BY product_id
FOR UPDATE;

-- 2. Validate stock quantities.

-- 3. Deduct inventory using conditional updates.

-- 4. Insert order.
INSERT INTO orders (
    order_id,
    customer_id,
    status
)
VALUES (
    1001,
    25,
    'Pending'
);

-- 5. Insert order items.
INSERT INTO order_items (
    order_id,
    product_id,
    quantity,
    unit_price
)
VALUES
    (1001, 101, 2, 500),
    (1001, 102, 1, 700),
    (1001, 103, 1, 300);

COMMIT;
```

This is a structural example. In production, each stock deduction must be checked for success, all required product IDs must be validated, and errors must cause rollback.

### **What happens if the third product is unavailable?**

Rollback the transaction.

All inventory deductions, order inserts and order-item inserts within that transaction are undone.

### **What about payment?**

This is where the problem gets more interesting.

Payment may involve an external gateway such as a payment processor.

A database transaction cannot automatically include that external network request in the same ACID transaction.

You generally should not hold inventory row locks and a database transaction open while waiting for a payment gateway.

A practical approach is:

1. Create the order in a pending state.
2. Reserve inventory within a database transaction.
3. Commit the reservation and order.
4. Initiate payment outside the database transaction.
5. Process the payment result.
6. Confirm the order or release the reservation on failure.

This brings us to two major distributed-systems patterns.

## **D5. Idempotency — prevent duplicate payments**

Extremely important backend concept

Imagine a customer clicks Pay.

The server processes the payment, but the response is lost due to a network timeout.

The customer clicks Pay again.

What happens?

Without protection, the customer may be charged twice.

We need idempotency.

**Idempotency means that repeating the same logical request produces the same intended effect rather than repeating the operation.**

### **Step 1: Create idempotency table**

```
CREATE TABLE payment_requests (
    idempotency_key VARCHAR(100) PRIMARY KEY,
    customer_id INT NOT NULL,
    request_hash TEXT NOT NULL,
    status VARCHAR(30) NOT NULL,
    payment_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

The primary key prevents duplicate idempotency keys.

### **Step 2: Client sends a unique key**

Example:

```
Idempotency-Key: payment_abc_123
```

### **Step 3: Server checks the key**

If the key has already been processed:

- Do not initiate a duplicate logical operation.
- Return or reconstruct the existing result.

If it is new:

- Register the request.
- Process the operation.
- Save the outcome.

### **Important production details**

- Same key + same request should return the existing result.
- Same key + different request should be rejected.
- Concurrent requests with the same key must be handled safely using a unique constraint and conflict handling.
- Pending or failed operations need carefully defined retry semantics.
- Idempotency records need a retention policy.

Example of atomic registration:

```
INSERT INTO payment_requests (
    idempotency_key,
    customer_id,
    request_hash,
    status
)
VALUES (
    'payment_abc_123',
    25,
    'request_hash_value',
    'Processing'
)
ON CONFLICT (idempotency_key)
DO NOTHING
RETURNING idempotency_key;
```

If a row is returned, the request registered successfully.

If no row is returned, the key already exists. Retrieve the existing record and verify the request hash before returning an existing result.

**Interview question: Why is idempotency required even when we have ACID transactions?**

Because ACID controls transactions within a database. Network retries can cause the same logical request to be submitted more than once. Idempotency prevents duplicate business effects across repeated requests.

## **D6. Transactional Outbox Pattern**

Important for system design and backend interviews

Imagine an order service must:

1. Save an order in PostgreSQL.
2. Publish an `OrderCreated` event to Kafka.

What if the database insert succeeds but Kafka is unavailable?

Or what if Kafka receives the event but the database transaction fails?

This is the dual-write problem.

### **Solution: Transactional Outbox**

Instead of publishing directly during the database operation, write both the order and an event record in the same transaction.

Schema:

```
CREATE TABLE outbox_events (
    event_id BIGSERIAL PRIMARY KEY,
    aggregate_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    published_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

Transaction:

```
BEGIN;

INSERT INTO orders (
    order_id,
    customer_id,
    status
)
VALUES (
    1001,
    25,
    'Created'
);

INSERT INTO outbox_events (
    aggregate_id,
    event_type,
    payload
)
VALUES (
    1001,
    'OrderCreated',
    '{"order_id": 1001, "customer_id": 25}'
);

COMMIT;
```

Now the order and the event record either both commit or neither commits.

A separate background worker publishes unpublished outbox events to Kafka.

**Transactional Outbox Architecture**

**Order Service**

**Single Database Transaction**

Orders table

Write

Outbox table

Write

Background outbox publisher

**Kafka / Message Broker**

**Downstream Consumers**

Inventory, notification, analytics

### **Important: Is the Outbox exactly-once?**

No.

A publisher might send an event successfully and crash before marking it published.

It can send the same event again after restarting.

Therefore, the usual design offers at-least-once delivery and requires consumers to process events idempotently.

Useful supporting concepts:

- Unique event IDs.
- Consumer deduplication.
- Retry handling.
- Poison message handling.
- Dead-letter queues.
- Event ordering.

## **D7. Job queues using SKIP LOCKED**

Suppose you have 100,000 jobs in a PostgreSQL table.

Multiple workers need to process jobs concurrently.

You don't want workers to pick the same job.

Table:

```
CREATE TABLE jobs (
    job_id BIGSERIAL PRIMARY KEY,
    payload JSONB,
    status VARCHAR(20) DEFAULT 'Pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

A worker can claim jobs like this:

```
BEGIN;

SELECT job_id, payload
FROM jobs
WHERE status = 'Pending'
ORDER BY created_at, job_id
FOR UPDATE SKIP LOCKED
LIMIT 10;

-- Mark the selected jobs as processing.

COMMIT;
```

This is the basic locking pattern.

A more atomic implementation is:

```
WITH selected_jobs AS (
    SELECT job_id
    FROM jobs
    WHERE status = 'Pending'
    ORDER BY created_at, job_id
    FOR UPDATE SKIP LOCKED
    LIMIT 10
)
UPDATE jobs j
SET status = 'Processing'
FROM selected_jobs s
WHERE j.job_id = s.job_id
RETURNING j.*;
```

The transaction locks available job rows and marks them Processing.

Other workers skip the rows that are already locked.

For a production job queue, you'll additionally need:

- Worker ownership or lease information.
- A processing timeout.
- Retry count.
- Failure status.
- Recovery of abandoned Processing jobs.
- Idempotent job execution.

## **D8. Production-grade concurrency patterns**

| Problem                              | Suitable technique                         |
| ------------------------------------ | ------------------------------------------ |
| Duplicate payment request            | Idempotency key + UNIQUE constraint        |
| Overselling inventory                | Atomic conditional UPDATE                  |
| Double booking                       | Conditional update + UNIQUE constraint     |
| Concurrent profile editing           | Optimistic locking                         |
| Conflicting balance modifications    | Transaction + row locks                    |
| Concurrent job processing            | FOR UPDATE SKIP LOCKED                     |
| Database + message broker dual-write | Transactional Outbox                       |
| Write skew                           | SERIALIZABLE or invariant-protecting locks |
| Deadlocks                            | Consistent lock order + retry              |
| Stale data in application            | Versioning or concurrency checks           |

### **D9. Important system design distinction: Local vs distributed transactions**

A normal PostgreSQL transaction can atomically modify records inside one database.

But imagine your system has:

- Order Service.
- Payment Service.
- Inventory Service.
- Shipping Service.

Each has its own database.

A PostgreSQL transaction cannot automatically make all four databases one ACID transaction.

There are two broad approaches:

**Distributed transaction approach:** Two-Phase Commit (2PC), where supported and appropriate.

**Saga approach:** A sequence of local transactions, each followed by the next action, with compensating operations when necessary.

Example:

```
Create Order
    ↓
Reserve Inventory
    ↓
Process Payment
    ↓
Create Shipment
    ↓
Order Confirmed
```

Suppose payment fails:

```
Payment Failed
    ↓
Release Inventory Reservation
    ↓
Cancel Order
```

This is a compensating workflow.

The compensation does not necessarily erase history; it creates a new operation that reverses the business effect.

**Interview answer:** A distributed transaction coordinates multiple participants to achieve a transaction-like outcome. In microservice architectures, Saga is often used to coordinate local transactions with compensating actions, while the transactional outbox supports reliable event publication.

# **SECTION D — Interview Problem-Solving Framework**

## **D10. How to approach database engineering scenarios in interviews**

When an interviewer presents a database problem, follow this sequence:

1. **Identify the invariant**

   What must always remain true? Example: account balance cannot become negative.
2. **Identify concurrent operations**

   Can multiple users modify the same records simultaneously?
3. **Choose transaction boundaries**

   Which database operations must succeed or fail together?
4. **Choose concurrency control**

   Atomic UPDATE, locks, version checking or SERIALIZABLE?
5. **Enforce database constraints**

   Primary keys, foreign keys, UNIQUE and CHECK constraints.
6. **Plan failure handling**

   Rollback, retry, idempotency and compensation.
7. **Consider performance**

   Indexes, query plans, locking duration and transaction size.

This framework applies to almost every practical database design and concurrency question.

# **SECTION D — Interview questions**

These are questions you should prepare to answer without executing SQL.

| #  | Interview question                                              | Core concept                        |
| -- | --------------------------------------------------------------- | ----------------------------------- |
| 1  | How would you implement a safe money transfer?                  | ACID                                |
| 2  | How would you prevent two users buying the last item?           | Atomic UPDATE / locking             |
| 3  | How would you prevent a seat from being double-booked?          | UNIQUE constraint                   |
| 4  | What if the payment gateway times out after processing payment? | Idempotency                         |
| 5  | How would you reliably publish an order event to Kafka?         | Transactional Outbox                |
| 6  | How would you process jobs with 10 workers?                     | SKIP LOCKED                         |
| 7  | What happens when two transactions update the same row?         | MVCC / row locks                    |
| 8  | How do you prevent a lost update?                               | Atomic updates / optimistic locking |
| 9  | What if two transactions deadlock?                              | Detection and retry                 |
| 10 | Why might an index not improve query performance?               | Selectivity / execution plan        |
| 11 | How would you optimise a slow JOIN?                             | Join plans / indexes                |
| 12 | How do you handle transactions spanning multiple services?      | Saga / 2PC                          |

# **Comprehensive Section 9 Practical Assessment**

Now combine Sections A, B, C and D.

These are the exercises I'd use to determine whether someone has actually understood database engineering rather than simply memorised DBMS definitions.

## **A. Indexing assignments**

| #  | Task                                                            | Difficulty |
| -- | --------------------------------------------------------------- | ---------- |
| 1  | Create a B-Tree index on employee salary.                       | Easy       |
| 2  | Create a composite index on customer ID and order date.         | Easy       |
| 3  | Create a partial index for pending orders.                      | Medium     |
| 4  | Create an expression index for case-insensitive email search.   | Medium     |
| 5  | Create a covering index for department-wise employee retrieval. | Medium     |
| 6  | Explain the leftmost-prefix behaviour of composite indexes.     | Medium     |
| 7  | Compare B-Tree and Hash indexes.                                | Medium     |
| 8  | Explain PostgreSQL vs InnoDB clustered index behaviour.         | Medium     |
| 9  | Explain when a sequential scan is preferable to an index scan.  | Hard       |
| 10 | Identify unnecessary indexes from a workload description.       | Hard       |

## **B. Query optimisation assignments**

| #  | Task                                                              | Difficulty |
| -- | ----------------------------------------------------------------- | ---------- |
| 1  | Use EXPLAIN to inspect a SELECT query.                            | Easy       |
| 2  | Use EXPLAIN ANALYZE and interpret actual rows and execution time. | Medium     |
| 3  | Compare sequential scan and index scan.                           | Medium     |
| 4  | Identify an incorrect row-count estimate.                         | Medium     |
| 5  | Rewrite a non-SARGable date condition.                            | Medium     |
| 6  | Optimise a JOIN across three tables.                              | Hard       |
| 7  | Explain Nested Loop, Hash Join and Merge Join.                    | Medium     |
| 8  | Rewrite an N+1 application query.                                 | Medium     |
| 9  | Replace deep OFFSET pagination with keyset pagination.            | Hard       |
| 10 | Identify why an existing index isn't being used.                  | Hard       |

## **C. Transactions and concurrency assignments**

| #  | Task                                                         | Difficulty |
| -- | ------------------------------------------------------------ | ---------- |
| 1  | Implement COMMIT and ROLLBACK.                               | Easy       |
| 2  | Demonstrate a SAVEPOINT.                                     | Easy       |
| 3  | Explain all ACID properties using banking.                   | Medium     |
| 4  | Demonstrate dirty read and non-repeatable read conceptually. | Medium     |
| 5  | Explain phantom read and write skew.                         | Hard       |
| 6  | Compare PostgreSQL isolation levels.                         | Hard       |
| 7  | Explain MVCC and VACUUM.                                     | Hard       |
| 8  | Implement SELECT FOR UPDATE.                                 | Medium     |
| 9  | Implement optimistic locking using version numbers.          | Medium     |
| 10 | Construct and diagnose a deadlock.                           | Hard       |
| 11 | Explain WAL and crash recovery.                              | Hard       |
| 12 | Handle serialization failures with safe retries.             | Hard       |

## **D. Backend engineering assignments**

| #  | Task                                                               | Difficulty |
| -- | ------------------------------------------------------------------ | ---------- |
| 1  | Build a transactional bank transfer.                               | Medium     |
| 2  | Prevent overselling inventory.                                     | Medium     |
| 3  | Implement safe seat booking.                                       | Hard       |
| 4  | Design an order checkout transaction.                              | Hard       |
| 5  | Implement payment idempotency.                                     | Hard       |
| 6  | Design a transactional outbox.                                     | Hard       |
| 7  | Implement concurrent database job processing.                      | Hard       |
| 8  | Handle payment timeout and duplicate callbacks.                    | Hard       |
| 9  | Design a multi-service order workflow using Saga.                  | Hard       |
| 10 | Explain how to maintain data integrity during concurrent requests. | Hard       |

# **Final Section 9 Interview Test**

Try to answer these as though you are sitting in an SDE technical interview.

**Section 9 final assessment**

10 questions

**Q1. A query returns 90% of a table's rows. Should you automatically force an index scan?**

Yes, indexes are always faster

No, a sequential scan may be cheaper

Always create a Hash index

Delete the index

**Q2. Which PostgreSQL feature lets multiple readers see consistent row versions during concurrent updates?**

Hash indexing

MVCC

DISTINCT

GROUP BY

**Q3. Two customers try to buy the last product. Which approach is a good starting point?**

Read stock in Python, then update later

Atomic conditional UPDATE

Use SELECT DISTINCT

Create a normal view

**Q4. A payment request times out and the customer retries. What prevents a duplicate logical payment?**

ORDER BY

Idempotency key with uniqueness enforcement

A B-Tree on amount only

OFFSET

**Q5. The order insert succeeds but publishing to Kafka fails. Which pattern addresses this dual-write problem?**

Transactional Outbox

CROSS JOIN

Clustered index

Dirty read

**Q6. Which PostgreSQL statement skips rows already locked by other workers?**

FOR UPDATE NOWAIT

FOR SHARE

FOR UPDATE SKIP LOCKED

VACUUM

**Q7. A transaction reads a snapshot and updates a different row from another concurrent transaction, potentially violating an invariant. Which anomaly could this represent?**

Write skew

Dirty read only

Index fragmentation

N+1

**Q8. Which statement about PostgreSQL REPEATABLE READ is correct?**

It always permits dirty reads

It prevents non-repeatable and phantom reads but can allow serialization anomalies

It is identical to READ COMMITTED

It automatically locks every table

**Q9. What happens when PostgreSQL detects a deadlock?**

It silently commits every transaction

It aborts a transaction to break the deadlock

It deletes the conflicting rows

It disables all indexes

**Q10. Which database mechanism supports crash recovery through a log written ahead of data-page persistence?**

WAL

STRING_AGG

DENSE_RANK

CROSS JOIN

Submit assessment

# **Section 9 — Final mastery checklist**

**Your Section 9 progress**

# **0/20**

Explain B-Tree and B+ Tree

Choose an appropriate index type

Understand composite-index column order

Understand covering, partial and expression indexes

Explain clustered vs non-clustered indexes across PostgreSQL and MySQL

Read EXPLAIN ANALYZE output

Understand sequential, index and bitmap scans

Explain Nested Loop, Hash and Merge Join

Optimise SARGable queries

Solve N+1 and pagination problems

Explain ACID and WAL

Understand all four isolation levels

Explain dirty reads, phantom reads and write skew

Understand MVCC and VACUUM

Implement row-level locking

Compare optimistic and pessimistic locking

Explain and prevent deadlocks

Implement safe money transfer

Prevent inventory overselling and double booking

Explain idempotency, Outbox and Saga

## **What you should be able to do after Section 9**

You should be able to explain how an index works internally, inspect and optimise query execution plans, design safe transactions, prevent concurrent-update problems, and apply database consistency techniques to banking, inventory, booking and payment systems.

**The most important distinction:** Knowing SQL means knowing how to ask the database for data. Understanding DBMS and backend engineering means knowing how the database retrieves, stores, protects and changes that data under real-world workloads.

Next in our course is **Lesson 10 — Company SQL OAs and Mock Interviews**, where we'll combine everything from Lessons 1–9 into actual problem-solving rounds.