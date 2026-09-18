# **Lesson 9 — SQL Indexes, Transactions and Query Optimisation**



Critical for SDE interviews

Advanced DBMS + Backend Engineering



This lesson takes us from writing SQL queries to understanding how databases actually execute them, maintain data consistency and handle multiple users accessing the same records.

There are three major areas:

1. **Indexing:** How databases find records efficiently.
2. **Query optimisation:** How to identify and improve slow queries.
3. **Transactions and concurrency:** How databases maintain correctness when multiple operations happen simultaneously.

By the end, you should be able to answer questions such as:

- Why is a database query slow?
- What is a B-Tree index?
- Why can adding indexes make writes slower?
- What is the difference between clustered and non-clustered indexes?
- What happens when two users update the same bank account?
- What are ACID properties?
- What is MVCC?
- What are dirty reads, phantom reads and lost updates?
- What is optimistic vs pessimistic locking?
- How do databases prevent deadlocks?

We'll use **PostgreSQL** for practical examples and explain differences in other database engines where important.

# **Part 1 — Database Indexing**

## **1. What is an index?**

Imagine you have a book with 1,000 pages.

You want to find a particular topic.

Without an index, you might need to scan every page until you find it.

With an index, you can directly locate the relevant page.

A database index works similarly.

An index is an additional data structure that helps the database locate records without necessarily scanning every row in a table.

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

Suppose you execute:

```
SELECT *
FROM employees
WHERE emp_id = 4;
```

Without a suitable index, the database may perform a sequential scan.

Conceptually:

```
Check row 1 → No
Check row 2 → No
Check row 3 → No
Check row 4 → Match
```

With a suitable index:

```
Search index
     |
     v
Locate emp_id = 4
     |
     v
Retrieve corresponding row
```

The database does not need to inspect every table row.

**Sequential scan vs Index scan**

**Sequential scan**

# **O(N)**

May inspect every row in the table.

Row 1

Row 2

Row 3

Row 4

Row 5

Row 6

**B-Tree index**

# **O(log N)**

Can locate matching keys through a tree traversal.

Root

Index nodes

Matching key

Illustrative complexity for a balanced B-Tree lookup. Actual query cost depends on selectivity, cache, table layout and the chosen execution plan.

## **2. Creating an index**

SQL syntax:

```
CREATE INDEX index_name
ON table_name(column_name);
```

Example:

```
CREATE INDEX idx_employee_salary
ON employees(salary);
```

Now the database has an index on the salary column.

This may help queries such as:

```
SELECT *
FROM employees
WHERE salary > 80000;
```

Or:

```
SELECT *
FROM employees
WHERE salary = 75000;
```

To remove an index:

```
DROP INDEX idx_employee_salary;
```

To inspect indexes in PostgreSQL:

```
SELECT *
FROM pg_indexes
WHERE tablename = 'employees';
```

**Important:** Creating an index does not guarantee that the database will use it. The query optimiser decides whether it is cheaper to use the index or scan the table.

## **3. Types of indexes**

### **A. Primary key index**

When you declare:

```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

PostgreSQL automatically creates a unique B-Tree index to enforce the primary key constraint.

You generally do not need to create another index on the same primary key.

### **B. Unique index**

A unique index prevents duplicate indexed values.

```
CREATE UNIQUE INDEX idx_employee_email
ON employees(email);
```

Now duplicate non-NULL email values are rejected.

A UNIQUE constraint is another way to enforce uniqueness:

```
ALTER TABLE employees
ADD CONSTRAINT unique_employee_email UNIQUE(email);
```

A UNIQUE constraint is backed by a unique index in PostgreSQL.

### **C. Composite index**

A composite index is built on multiple columns.

Example:

```
CREATE INDEX idx_department_salary
ON employees(department_id, salary);
```

This index stores information based on the combination of department ID and salary.

It can be useful for:

```
SELECT *
FROM employees
WHERE department_id = 101
  AND salary > 70000;
```

It may also support:

```
SELECT *
FROM employees
WHERE department_id = 101;
```

But an index on `(department_id, salary)` is generally less suitable for a query filtering only by salary:

```
SELECT *
FROM employees
WHERE salary > 70000;
```

Why?

Because column order matters.

## **4. The Leftmost Prefix Rule**

Very important interview concept

Suppose you create:

```
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date);
```

The index columns are:

```
(customer_id, order_date)
```

Think of the index as being primarily organised by customer ID, with order date as the next ordering key.

Typical usefulness:

| Query                                                | Index suitability       |
| ---------------------------------------------------- | ----------------------- |
| WHERE customer_id = 10                               | Good                    |
| WHERE customer_id = 10 AND order_date = '2026-01-01' | Good                    |
| WHERE customer_id = 10 AND order_date > '2026-01-01' | Good                    |
| WHERE order_date = '2026-01-01'                      | Generally less suitable |
| WHERE order_date > '2026-01-01'                      | Generally less suitable |

For a B-Tree index, the leading column is particularly important for locating matching ranges.

**Interview question: Does the order of columns matter in a composite index?**

Yes. The leading column determines how the B-Tree is organised. An index on `(A, B)` generally supports lookups beginning with A more effectively than lookups using only B.

## **5. B-Tree and B+ Tree**

Must understand for interviews

A B-Tree is a self-balancing search tree designed to keep data organised and allow efficient searching, insertion and deletion.

A database index should minimise expensive disk access.

B-Tree structures achieve this using nodes that can hold multiple keys and child pointers.

Conceptual diagram:

**B-Tree illustration**

**[30 | 60]**

**[10 | 20]**

**[40 | 50]**

**[70 | 80]**

Simplified tree structure showing how values are divided into searchable ranges.

### **Why B-Trees?**

- Balanced structure.
- Efficient searching.
- Efficient insertion and deletion.
- Support equality and range queries.
- Reduce unnecessary page reads.

Typical lookup complexity:

\\[ O(\log N) \\]

### **What is a B+ Tree?**

A B+ Tree is a related tree structure in which:

- Internal nodes primarily contain separator keys and child pointers.
- Actual indexed entries are stored at leaf level.
- Leaf pages are linked, helping sequential and range scans.

This structure is widely used in database indexing.

**PostgreSQL detail:** PostgreSQL's standard B-Tree implementation is a balanced, Lehman-Yao/B-link-style tree with linked pages. It is reasonable to learn B+ Tree concepts for database interviews, but avoid claiming that every database implements identical B+ Tree internals.

### **Why are B-Trees useful for range queries?**

Example:

```
SELECT *
FROM employees
WHERE salary BETWEEN 60000 AND 85000;
```

The index can locate the beginning of the required salary range and scan relevant entries.

Hash-based indexes are not designed for ordered range traversal.

## **6. Other PostgreSQL index types**

| Index type | Main use                                                       |
| ---------- | -------------------------------------------------------------- |
| B-Tree     | Equality, range queries, sorting                               |
| Hash       | Equality comparisons                                           |
| GIN        | Composite values such as arrays, JSONB and full-text search    |
| GiST       | Spatial, geometric and other extensible search types           |
| BRIN       | Very large tables with values correlated to physical row order |

You should know B-Tree extremely well. Learn the other types conceptually for interviews.

### **GIN example**

Suppose you have a JSONB column:

```
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    attributes JSONB
);
```

An example record could contain:

```
{
  "brand": "Acme",
  "color": "blue",
  "size": "large"
}
```

Create a GIN index:

```
CREATE INDEX idx_product_attributes
ON products
USING GIN(attributes);
```

This can help certain JSONB containment and key-existence queries.

# **Part 2 — Clustered and Non-Clustered Indexes**

## **7. What is a clustered index?**

A clustered index determines how table records are organised at the leaf level of an index structure.

In database systems that implement clustered indexes, the table's actual data rows are stored in the order of the clustered index.

Example:

Suppose employees are organised by employee ID.

| emp_id | name  | salary |
| ------ | ----- | ------ |
| 1      | Aman  | 60000  |
| 2      | Rahul | 80000  |
| 3      | Priya | 75000  |
| 4      | Neha  | 90000  |

Searching sequentially by employee ID can benefit from this organisation.

### **Non-clustered index**

A non-clustered index is a separate index structure that stores indexed values and references to the corresponding table records.

Conceptually:

```
NON-CLUSTERED INDEX

Salary       Row reference
  50000  ---> Karan's row
  60000  ---> Aman's row
  75000  ---> Priya's row
  80000  ---> Rahul's row
  90000  ---> Neha's row
```

The database uses the index to find the relevant row references and retrieve the underlying records.

### **Comparison**

| Clustered index                                | Non-clustered index                  |
| ---------------------------------------------- | ------------------------------------ |
| Determines data organisation at the leaf level | Separate index structure             |
| Often efficient for ordered range access       | Useful for additional search columns |
| Typically one clustering order per table       | Multiple secondary indexes possible  |
| Data and index are closely integrated          | Index references the underlying data |

### **Important: PostgreSQL vs MySQL**

This is a common interview trap.

**MySQL InnoDB:**

- The primary key is the clustered index.
- Table rows are stored in the clustered primary-key structure.
- Secondary indexes contain the primary-key value as the row reference.

**PostgreSQL:**

- Ordinary indexes are separate from the heap table.
- PostgreSQL does not maintain a clustered index automatically.
- `CLUSTER` can physically reorder a table according to an index, but this ordering is not automatically maintained after future modifications.

Example:

```
CLUSTER employees USING employees_pkey;
```

Do not give the unqualified answer that every PostgreSQL primary key is a clustered index.

# **Part 3 — Advanced Indexing**

## **8. Covering indexes and index-only scans**

Suppose this query runs frequently:

```
SELECT name, salary
FROM employees
WHERE department_id = 101;
```

One approach is:

```
CREATE INDEX idx_employee_department
ON employees(department_id);
```

The index finds the matching employees, but the database may need to visit the table to retrieve `name` and `salary`.

A covering index stores the required additional columns in the index.

In PostgreSQL:

```
CREATE INDEX idx_employee_department_cover
ON employees(department_id)
INCLUDE (name, salary);
```

Now the index contains:

- Search key: department_id
- Additional indexed payload: name, salary

This can allow an index-only scan.

**What is an index-only scan?**

An index-only scan retrieves the required columns directly from the index without needing to fetch the corresponding heap tuples, when PostgreSQL's visibility information allows it.

Important: `INCLUDE` doesn't guarantee an index-only scan. PostgreSQL must still verify tuple visibility, and the visibility map determines whether heap visits can be skipped.

## **9. Partial indexes**

A partial index contains entries only for rows satisfying a specified condition.

Example:

Suppose 95% of employees are active and your application frequently searches inactive employees.

```
CREATE INDEX idx_inactive_employees
ON employees(emp_id)
WHERE status = 'Inactive';
```

The index contains only inactive employees.

Query:

```
SELECT *
FROM employees
WHERE status = 'Inactive';
```

This may use the partial index.

Benefits:

- Smaller index.
- Less storage.
- Fewer index entries to maintain.
- Potentially faster targeted queries.

Another practical example:

```
CREATE INDEX idx_unpaid_orders
ON orders(order_id)
WHERE payment_status = 'Pending';
```

This is useful when queries frequently retrieve pending orders.

## **10. Expression indexes**

Sometimes applications search using a function on a column.

Example:

```
SELECT *
FROM users
WHERE LOWER(email) = 'parth@gmail.com';
```

An ordinary index on `email` may not directly support this expression.

Create an expression index:

```
CREATE INDEX idx_users_lower_email
ON users(LOWER(email));
```

Now queries using the same expression can use the index.

Other examples:

```
CREATE INDEX idx_users_upper_name
ON users(UPPER(name));
```

Expression indexes are useful for case-insensitive searches and other repeated computations.

## **11. When should you create an index?**

A good index candidate often has one or more of these properties:

- Frequently used in WHERE conditions.
- Frequently used in JOIN conditions.
- Used for common ORDER BY operations.
- Used for range queries.
- Helps enforce a uniqueness requirement.

Example:

```
SELECT *
FROM orders
WHERE customer_id = 101
ORDER BY order_date DESC;
```

A possible index:

```
CREATE INDEX idx_orders_customer_date
ON orders(customer_id, order_date DESC);
```

This can help the database find a customer's orders in date order.

### **Should we index every column?**

No.

Indexes introduce costs.

| Advantage                        | Cost                          |
| -------------------------------- | ----------------------------- |
| Faster lookups                   | Additional storage            |
| Faster selected joins            | Slower INSERT operations      |
| Faster selected ORDER BY queries | Slower DELETE operations      |
| Faster selected range queries    | Additional UPDATE maintenance |
| Can enforce uniqueness           | More memory/cache pressure    |

An index is an additional structure that must stay consistent with the table.

Every insert, delete or relevant update may require index maintenance.

**Interview answer:**

Indexes improve read performance by providing efficient access paths, but increase storage requirements and write overhead because database modifications must maintain the index structures.

# **Part 4 — Query Optimisation**

Extremely important for backend interviews

## **12. What is query optimisation?**

Query optimisation is the process of choosing and improving a database query's execution strategy to reduce resource consumption and execution time.

The database doesn't blindly execute your SQL exactly as written.

It generally performs several steps:

1. Parse the SQL.
2. Validate tables, columns and permissions.
3. Transform and optimise the query.
4. Generate possible execution plans.
5. Choose an execution plan.
6. Execute it.

Consider:

```
SELECT *
FROM employees
WHERE salary > 80000;
```

The database might choose:

- Sequential scan.
- Index scan.
- Bitmap index scan followed by heap access.

Its choice depends on the statistics, data distribution, indexes, row count and estimated cost.

## **13. EXPLAIN**

One of the most important tools for debugging slow queries.

```
EXPLAIN
SELECT *
FROM employees
WHERE salary > 80000;
```

EXPLAIN displays the planned execution strategy.

Example of a simplified plan:

```
Index Scan using idx_employee_salary
on employees

Index Cond: (salary > 80000)
```

This tells you the database intends to use the salary index.

### **EXPLAIN ANALYZE**

```
EXPLAIN ANALYZE
SELECT *
FROM employees
WHERE salary > 80000;
```

EXPLAIN ANALYZE actually executes the query and reports measured execution information.

Example of an illustrative output:

```
Index Scan using idx_employee_salary
(cost=0.15..12.30 rows=10)
(actual time=0.025..0.080 rows=12 loops=1)

Planning Time: 0.200 ms
Execution Time: 0.100 ms
```

Important fields:

| Field          | Meaning                                  |
| -------------- | ---------------------------------------- |
| cost           | Estimated planner cost, not milliseconds |
| rows           | Estimated or actual number of rows       |
| width          | Estimated average row size               |
| actual time    | Measured execution time                  |
| loops          | Number of node executions                |
| Planning Time  | Time spent creating the plan             |
| Execution Time | Time spent executing the query           |

**Very important:** EXPLAIN ANALYZE executes the statement. Be careful with UPDATE, DELETE and INSERT statements. For testing modifications, use a transaction and ROLLBACK where appropriate.

## **14. Sequential Scan vs Index Scan**

### **Sequential Scan**

The database reads table pages sequentially.

```
Sequential Scan on employees
```

Often appropriate when:

- The table is small.
- A large percentage of rows is needed.
- An index would cause too many random accesses.

### **Index Scan**

The database accesses an index and then retrieves matching table rows.

```
Index Scan using idx_employee_salary
```

Often appropriate when:

- The search is selective.
- Relatively few rows are required.
- A useful index exists.

### **Bitmap Index Scan**

PostgreSQL can combine index results into a bitmap and then retrieve relevant heap pages.

This is particularly useful when multiple rows match and fetching rows one at a time is less efficient.

**Important interview trap:** An index scan is not always better than a sequential scan. The optimiser may correctly choose a sequential scan because it estimates that reading the whole table is cheaper.

# **15. SARGable Queries**

Important performance concept

SARGable means that a search condition is structured so that the database can efficiently use an available index.

Consider this query:

```
SELECT *
FROM orders
WHERE EXTRACT(YEAR FROM order_date) = 2026;
```

The query applies a function to the indexed column.

An ordinary B-Tree index on `order_date` may not efficiently support this condition.

A better query:

```
SELECT *
FROM orders
WHERE order_date >= DATE '2026-01-01'
  AND order_date < DATE '2027-01-01';
```

Now the database can use a range scan on the ordinary date index.

### **Another example**

Less suitable for a normal B-Tree index:

```
SELECT *
FROM users
WHERE LOWER(email) = 'parth@gmail.com';
```

Possible solution 1: Expression index.

```
CREATE INDEX idx_email_lower
ON users(LOWER(email));
```

Possible solution 2: Store a normalised email value and index it.

### **Leading wildcard problem**

This query:

```
SELECT *
FROM users
WHERE name LIKE '%parth%';
```

Generally cannot use an ordinary B-Tree index for an efficient substring search.

For PostgreSQL, a trigram index is one option:

```
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_users_name_trgm
ON users
USING GIN(name gin_trgm_ops);
```

This can accelerate supported substring and similarity searches.

But a prefix search:

```
WHERE name LIKE 'Parth%'
```

can often use a suitable B-Tree index, depending on collation and operator class.

# **16. Common SQL Optimisation Techniques**

| Poor practice                                  | Better approach                               |
| ---------------------------------------------- | --------------------------------------------- |
| SELECT \* unnecessarily                        | Select required columns                       |
| Missing indexes on frequent selective searches | Evaluate suitable indexes                     |
| Applying functions to indexed columns          | Prefer range predicates or expression indexes |
| Repeating nested calculations                  | Consider CTEs or derived tables               |
| Retrieving unused rows                         | Filter early when logically valid             |
| Joining unrelated or unnecessary tables        | Join only what is required                    |
| Large OFFSET pagination                        | Consider keyset pagination                    |
| Unindexed join/filter columns                  | Review execution plans                        |
| Fetching each related record separately        | Consider JOINs or batch queries               |

### **Keyset pagination**

Traditional pagination:

```
SELECT *
FROM orders
ORDER BY order_id
LIMIT 20 OFFSET 100000;
```

The database may need to traverse a large number of rows before reaching the requested page.

Keyset pagination:

```
SELECT *
FROM orders
WHERE order_id > 100000
ORDER BY order_id
LIMIT 20;
```

This can be more efficient when the ordering key is indexed and unique.

In real applications, the last seen key is stored and passed as the next page cursor.

### **N+1 query problem**

Suppose you're displaying 100 orders with their customer names.

Bad application approach:

1. Query all 100 orders.
2. For every order, execute another query to get its customer.

Total queries:

\\[ 1 + 100 = 101 \\]

Better:

```
SELECT
    o.order_id,
    o.amount,
    c.name
FROM orders o
JOIN customers c
    ON o.customer_id = c.customer_id;
```

One query retrieves the required information.

This is the N+1 query problem, often discussed in ORM and backend interviews.

# **Part 5 — Transactions**

Critical DBMS interview topic

## **17. What is a transaction?**

A transaction is a logical unit of database work containing one or more operations that must be managed together.

Consider online banking.

Aman transfers ₹5,000 to Rahul.

There are two operations:

1. Deduct ₹5,000 from Aman's account.
2. Add ₹5,000 to Rahul's account.

Suppose the first operation succeeds but the second fails.

What happens?

Aman loses ₹5,000, but Rahul doesn't receive anything.

This is an inconsistent financial outcome.

A transaction ensures that both operations succeed together or fail together.

### **SQL implementation**

**accounts**

| account_id | account_holder | balance |
| ---------- | -------------- | ------- |
| 1          | Aman           | 20000   |
| 2          | Rahul          | 10000   |

Transaction:

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

If something fails:

```
ROLLBACK;
```

After a successful transfer:

| account_holder | balance |
| -------------- | ------- |
| Aman           | 15000   |
| Rahul          | 15000   |

The transaction groups the modifications into one unit of work.

## **18. ACID properties**

Must memorise and understand

ACID stands for:

- Atomicity
- Consistency
- Isolation
- Durability

**A — Atomicity**

All operations succeed or none of them take effect.

**C — Consistency**

A committed transaction preserves defined database constraints and invariants.

**I — Isolation**

Concurrent transactions are controlled so their execution behaves according to the selected isolation level.

**D — Durability**

Once committed, data survives crashes according to the database's durability guarantees.

### **A — Atomicity**

Imagine transferring ₹5,000.

Both the debit and credit must be part of the same transaction.

If one fails, the transaction should roll back.

### **C — Consistency**

Suppose the database enforces:

```
CHECK (balance >= 0)
```

A transaction cannot commit a balance that violates this constraint.

Consistency means preserving defined integrity rules and application invariants.

It does not mean every application-level business rule is automatically enforced by the database.

### **I — Isolation**

Suppose two users simultaneously update the same bank account.

Isolation controls how these concurrent operations interact.

We'll study isolation levels shortly.

### **D — Durability**

Suppose a transfer completes successfully and the database confirms COMMIT.

If the database subsequently crashes, the committed transaction must survive according to its configured durability guarantees.

PostgreSQL uses mechanisms including Write-Ahead Logging (WAL) to support crash recovery.

# **Part 6 — COMMIT, ROLLBACK and SAVEPOINT**

## **19. COMMIT**

Makes the transaction's changes permanent according to the database's durability guarantees.

```
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 1;

COMMIT;
```

## **20. ROLLBACK**

Reverses the uncommitted changes of the current transaction.

```
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 1;

ROLLBACK;
```

The balance returns to its previous committed state.

## **21. SAVEPOINT**

A SAVEPOINT creates an intermediate point inside a transaction.

```
BEGIN;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 1;

SAVEPOINT first_operation;

UPDATE accounts
SET balance = balance + 1000
WHERE account_id = 2;

ROLLBACK TO SAVEPOINT first_operation;

COMMIT;
```

This example reverses the second update while preserving the first update, assuming no other error occurred.

Useful commands:

| Command           | Purpose                               |
| ----------------- | ------------------------------------- |
| BEGIN             | Start transaction                     |
| COMMIT            | Commit transaction                    |
| ROLLBACK          | Roll back transaction                 |
| SAVEPOINT         | Create an intermediate recovery point |
| ROLLBACK TO       | Return to a savepoint                 |
| RELEASE SAVEPOINT | Remove a savepoint                    |

# **Part 7 — Transaction Isolation Levels**

Extremely important for SDE interviews

## **22. Why do we need isolation levels?**

Imagine two banking application requests accessing the same account simultaneously.

Transaction A reads a balance of ₹10,000.

Transaction B also reads ₹10,000.

Both transactions perform calculations using that same value.

Without proper concurrency control, their operations can produce incorrect results.

This is a concurrency problem.

There are four standard isolation levels defined by SQL.

1. READ UNCOMMITTED.
2. READ COMMITTED.
3. REPEATABLE READ.
4. SERIALIZABLE.

Let's understand the anomalies first.

## **23. Dirty Read**

A dirty read occurs when one transaction reads data written by another transaction that has not yet committed.

Example:

**Dirty read scenario**

**Transaction A**

UPDATE balance = 5000

Not committed

**Transaction B**

Reads balance = 5000

Dirty read

Transaction A rolls back.

**The value read by B was never committed.**

Transaction B has read an uncommitted value.

That is a dirty read.

## **24. Non-Repeatable Read**

A transaction reads the same row twice but gets different values because another transaction committed an update between the reads.

Scenario:

1. Transaction A reads balance = 10,000.
2. Transaction B updates balance to 8,000 and commits.
3. Transaction A reads the same row again.
4. Transaction A receives 8,000.

The same query returned different values during the same transaction.

This is a non-repeatable read.

## **25. Phantom Read**

A transaction executes the same condition-based query twice and receives a different set of rows because another transaction inserted or deleted matching rows and committed.

Example:

Transaction A:

```
SELECT *
FROM employees
WHERE salary > 70000;
```

It receives five rows.

Transaction B inserts another employee earning 90,000 and commits.

Transaction A repeats the query and receives six rows.

The additional matching row is a phantom row.

**Difference between non-repeatable and phantom reads:**

- Non-repeatable read: An existing row changes.
- Phantom read: The set of rows matching the query changes.

## **26. Lost Update**

A lost update occurs when concurrent transactions overwrite each other's modifications.

Suppose inventory quantity is 10.

Two transactions both read 10.

- Transaction A wants to subtract 2.
- Transaction B wants to subtract 3.

Both calculate their result based on the stale value 10.

Transaction A writes 8.

Transaction B writes 7.

The final quantity becomes 7, even though 5 units should have been deducted.

**Lost update**

**Initial inventory**

# **10**

**Transaction A**

10 - 2 = 8

Writes 8

**Transaction B**

10 - 3 = 7

Writes 7

**Incorrect final quantity: 7**

**Expected final quantity: 5**

A safer approach is to perform the calculation directly inside an atomic SQL UPDATE:

```
UPDATE inventory
SET quantity = quantity - 2
WHERE product_id = 101
  AND quantity >= 2;
```

The database coordinates concurrent modifications to the row.

This is preferable to blindly reading a value into application memory, calculating a new value and overwriting the row.

# **Part 8 — Comparing Isolation Levels**

## **27. Isolation level comparison**

| Isolation level  | Dirty reads                    | Non-repeatable reads | Phantom reads                                     |
| ---------------- | ------------------------------ | -------------------- | ------------------------------------------------- |
| Read Uncommitted | Possible in standard SQL model | Possible             | Possible                                          |
| Read Committed   | Prevented                      | Possible             | Possible                                          |
| Repeatable Read  | Prevented                      | Prevented            | Prevented in PostgreSQL; standard SQL allows them |
| Serializable     | Prevented                      | Prevented            | Prevented                                         |

There are important implementation differences between database engines.

### **PostgreSQL-specific behaviour**

This is particularly important for interviews.

PostgreSQL implements:

- READ UNCOMMITTED as READ COMMITTED.
- READ COMMITTED with a new snapshot for each statement.
- REPEATABLE READ using transaction-level snapshot isolation.
- SERIALIZABLE using Serializable Snapshot Isolation (SSI).

PostgreSQL's REPEATABLE READ also prevents phantom reads, but it can still allow serialization anomalies such as write skew.

SERIALIZABLE provides stronger guarantees, but some transactions may be aborted with serialization failures. Applications must retry those transactions.

### **MySQL InnoDB**

InnoDB's default isolation level is typically REPEATABLE READ.

It uses MVCC and locking mechanisms such as next-key locks to control certain phantom-producing operations.

**Interview tip:** Never assume all database engines implement isolation levels identically. State which database you're discussing.

# **Part 9 — MVCC (Multi-Version Concurrency Control)**

Important for backend engineering

## **28. What is MVCC?**

MVCC stands for Multi-Version Concurrency Control.

Instead of requiring every reader and writer to block each other, MVCC allows the database to maintain multiple versions of a row.

Each transaction reads the version appropriate to its database snapshot.

Let's understand the concept.

Initial record:

| account_id | balance |
| ---------- | ------- |
| 1          | 10000   |

Transaction A starts and reads the account.

Transaction B updates the balance to 8000 and commits.

Conceptually:

```
Version 1:
balance = 10000

Version 2:
balance = 8000
```

Depending on its snapshot:

- Transaction A may continue seeing the older version.
- A later transaction may see the new committed version.

This allows concurrent readers to access consistent snapshots while writes are happening.

**Conceptual MVCC model**

**Transaction A**

Snapshot sees balance = 10,000

**Transaction B**

Creates updated version: balance = 8,000

Commits

**Transaction A**

Can continue seeing its snapshot's earlier version under an appropriate isolation level.

Conceptual illustration. Actual visibility depends on the database engine and transaction isolation level.

### **Benefits of MVCC**

- Readers generally don't block writers.
- Writers generally don't block ordinary snapshot readers.
- Transactions can read consistent versions of data.
- Concurrency can improve.

### **What is the disadvantage?**

Old versions cannot be discarded immediately if active transactions may still need them.

In PostgreSQL:

- Updates and deletes can leave obsolete tuple versions.
- VACUUM reclaims or makes space reusable.
- Autovacuum performs routine maintenance.
- Long-running transactions can delay cleanup and contribute to table bloat.

**Interview answer:**

MVCC is a concurrency-control technique that maintains multiple versions of records so transactions can access consistent snapshots without requiring ordinary reads to block concurrent writes.

# **Part 10 — Locks**

## **29. What is a database lock?**

A lock is a concurrency-control mechanism used to coordinate access to shared database resources.

Locks help prevent conflicting operations from violating database correctness.

Common conceptual lock categories:

| Lock             | Purpose                         |
| ---------------- | ------------------------------- |
| Shared lock      | Allows compatible readers       |
| Exclusive lock   | Restricts conflicting access    |
| Row-level lock   | Protects particular rows        |
| Table-level lock | Protects a table-level resource |

Actual lock modes and compatibility rules vary by database.

PostgreSQL provides several row-level lock modes, including:

- FOR UPDATE
- FOR NO KEY UPDATE
- FOR SHARE
- FOR KEY SHARE

## **30. SELECT FOR UPDATE**

Suppose two customers attempt to purchase the last available unit of a product.

Inventory:

| product_id | quantity |
| ---------- | -------- |
| 101        | 1        |

We want to prevent both transactions from purchasing the same unit.

```
BEGIN;

SELECT quantity
FROM inventory
WHERE product_id = 101
FOR UPDATE;
```

This obtains a row-level lock.

Another transaction trying to acquire a conflicting row lock must wait until the current transaction releases it.

Then:

```
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity > 0;

COMMIT;
```

The second transaction can continue after the first finishes, but must still check whether stock remains.

### **A more compact implementation**

For inventory deduction, a conditional UPDATE can perform the check and modification atomically:

```
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity >= 1
RETURNING quantity;
```

If a row is returned, the deduction succeeded.

If no row is returned, the product was unavailable.

This is a particularly practical backend engineering pattern.

# **Part 11 — Pessimistic vs Optimistic Locking**

Very common SDE interview topic

## **31. Pessimistic locking**

Pessimistic locking assumes conflicts are likely.

It acquires a lock before modifying a record.

Example:

```
BEGIN;

SELECT *
FROM accounts
WHERE account_id = 101
FOR UPDATE;

UPDATE accounts
SET balance = balance - 1000
WHERE account_id = 101;

COMMIT;
```

Another transaction attempting a conflicting modification must wait.

Applications:

- Banking.
- Inventory management.
- Critical resource allocation.
- Scenarios where conflicts are expensive.

## **32. Optimistic locking**

Optimistic locking assumes conflicts are relatively uncommon.

Instead of locking a record throughout the operation, the application checks whether the record has changed before updating it.

A common implementation uses a version number.

**products**

| product_id | price | version |
| ---------- | ----- | ------- |
| 101        | 1000  | 1       |

Suppose an application reads version 1.

It prepares a price update.

```
UPDATE products
SET price = 1200,
    version = version + 1
WHERE product_id = 101
  AND version = 1;
```

If one row is updated, the operation succeeds.

If zero rows are updated, the version was changed by another transaction or the record did not exist.

The application can fetch the latest state and retry or report a conflict.

### **Comparison**

| Pessimistic locking                 | Optimistic locking                          |
| ----------------------------------- | ------------------------------------------- |
| Locks records before modification   | Checks for conflicts during modification    |
| Conflicting transactions may wait   | Conflicting transactions may fail and retry |
| Good for high-contention operations | Good when conflicts are relatively uncommon |
| Can reduce concurrency              | Often allows more concurrency               |
| Deadlocks may occur                 | Version conflicts may occur                 |

### **Interview answer**

Pessimistic locking acquires locks before making changes to prevent conflicting modifications. Optimistic locking allows concurrent work but validates a version or condition during the update, rejecting stale modifications.

# **Part 12 — Deadlocks**

## **33. What is a deadlock?**

A deadlock occurs when two or more transactions wait for resources held by one another, creating a circular waiting dependency.

Consider two accounts:

- Account A
- Account B

Transaction 1:

1. Locks Account A.
2. Attempts to lock Account B.

Transaction 2:

1. Locks Account B.
2. Attempts to lock Account A.

Now both are waiting.

**Deadlock scenario**

**Transaction 1**

Holds A

Waits for B

**Transaction 2**

Holds B

Waits for A

Circular wait

### **How do databases handle deadlocks?**

Databases detect deadlocks and select a transaction to abort.

The victim transaction receives an error. Its uncommitted work is rolled back.

The application may retry the transaction where appropriate.

### **How can we prevent deadlocks?**

One important strategy:

**Acquire locks in a consistent order.**

For example, always lock the lower account ID first.

Transaction 1:

```
BEGIN;

SELECT *
FROM accounts
WHERE account_id IN (101, 102)
ORDER BY account_id
FOR UPDATE;

-- Perform transfer
COMMIT;
```

Transaction 2 should use the same ordering.

This reduces circular waiting.

Other strategies:

- Keep transactions short.
- Avoid unnecessary locks.
- Access tables and rows in consistent order.
- Use suitable indexes so statements do not scan and lock more rows than necessary.
- Retry deadlock victims safely.

**Interview distinction:**

- Deadlock: Transactions wait on one another in a cycle.
- Starvation: A transaction keeps waiting because other transactions repeatedly get priority.
- Blocking: One transaction waits for another to release a conflicting resource; blocking alone is not necessarily a deadlock.

# **Part 13 — Advanced Concurrency Scenario**

## **34. Two users purchasing the last product**

This is an excellent practical interview problem.

Initial inventory:

```
product_id = 101
quantity = 1
```

Two requests arrive at approximately the same time.

A poorly written application might:

1. Read stock = 1.
2. Check stock > 0 in application code.
3. Deduct one unit later.

Both requests can read 1 before either performs its update.

This creates an overselling risk.

### **Better approach: Atomic conditional UPDATE**

```
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity >= 1
RETURNING quantity;
```

The database coordinates concurrent updates to the row.

Only one request can successfully decrement the final unit.

The other request will find no qualifying row after the competing update is committed.

### **Interview follow-up**

**What if the order also requires inserting an order record and a payment record?**

Use a transaction.

```
BEGIN;

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101
  AND quantity >= 1
RETURNING quantity;

-- Application checks whether a row was returned.

-- If successful:
INSERT INTO orders (...);

INSERT INTO payments (...);

COMMIT;
```

If stock is unavailable or any required database operation fails:

```
ROLLBACK;
```

For real applications, the application must handle the conditional UPDATE result, perform the remaining operations in the same transaction, and implement idempotency for payment-related retries. External payment gateways cannot be made atomic merely by wrapping SQL statements in a database transaction.

# **Part 14 — Interview Revision Sheet**

| Topic               | One-line understanding                                          |
| ------------------- | --------------------------------------------------------------- |
| Index               | Additional data structure for efficient data retrieval          |
| B-Tree              | Balanced search tree supporting equality and range searches     |
| Composite index     | Index containing multiple columns                               |
| Covering index      | Index containing all columns required by a query                |
| Partial index       | Index containing only rows satisfying a condition               |
| Expression index    | Index built on an expression                                    |
| Clustered index     | Determines data organisation in a clustered-index storage model |
| EXPLAIN             | Shows the planned execution strategy                            |
| EXPLAIN ANALYZE     | Executes the query and reports measured execution information   |
| SARGable            | Search condition that can efficiently use an index              |
| Transaction         | Logical unit of database operations                             |
| ACID                | Atomicity, Consistency, Isolation, Durability                   |
| Dirty read          | Reading another transaction's uncommitted data                  |
| Non-repeatable read | Same row gives different values within a transaction            |
| Phantom read        | Same predicate returns a different set of rows                  |
| Lost update         | Concurrent updates overwrite one another                        |
| MVCC                | Multiple row versions for concurrency control                   |
| Pessimistic locking | Lock before modifying                                           |
| Optimistic locking  | Validate changes when updating                                  |
| Deadlock            | Circular waiting for resources                                  |

# **Part 15 — Interactive Interview Assessment**

**Indexes, transactions and concurrency**

12 questions

**Q1. What is the primary purpose of a database index?**

Reduce table size

Improve data retrieval efficiency

Automatically normalise tables

Eliminate transactions

**Q2. Which index is generally appropriate for range queries on salary?**

B-Tree

Hash only

No index can support ranges

UNIQUE only

**Q3. For an index on (department_id, salary), which lookup is generally supported by the leading prefix?**

WHERE salary = 80000 only

WHERE department_id = 101

WHERE name = 'Aman'

None

**Q4. Which is a cost of maintaining additional indexes?**

No longer needing transactions

Higher write-maintenance and storage costs

Automatic ACID violations

Removal of primary keys

**Q5. Which ACID property ensures committed changes survive a crash?**

Atomicity

Consistency

Isolation

Durability

**Q6. What is a dirty read?**

Reading a deleted table

Reading another transaction's uncommitted data

Reading an old index

Reading duplicate rows

**Q7. Which PostgreSQL isolation level gives a new snapshot for each statement?**

READ COMMITTED

REPEATABLE READ

SERIALIZABLE

READ UNCOMMITTED as a distinct level

**Q8. What does MVCC stand for?**

Multi-Value Column Control

Multi-Version Concurrency Control

Maximum Version Cache Control

Multiple View Consistency Check

**Q9. Which technique is commonly used for optimistic locking?**

A version column

A full table scan

A permanent table lock

A CROSS JOIN

**Q10. What happens when a database detects a deadlock?**

It always crashes

It typically aborts one transaction to break the cycle

It deletes both tables

It commits both transactions immediately

**Q11. What is a SARGable condition?**

A condition designed for efficient index-assisted searching

A condition that always performs a table scan

A database backup

A transaction isolation level

**Q12. Which is an appropriate way to prevent overselling the last inventory unit?**

Read stock in the application and update much later

Use an atomic conditional UPDATE with quantity >= 1

Remove the primary key

Use SELECT DISTINCT

Submit assessment

# **Part 16 — Practical Assignment**

Solve these problems without referring to the solutions.

| #  | Problem                                                                    | Difficulty |
| -- | -------------------------------------------------------------------------- | ---------- |
| 1  | Create a B-Tree index on employee salary.                                  | Easy       |
| 2  | Create a composite index on customer ID and order date.                    | Easy       |
| 3  | Create a partial index for pending orders.                                 | Medium     |
| 4  | Create an expression index for case-insensitive email searches.            | Medium     |
| 5  | Explain why indexing every column is not recommended.                      | Easy       |
| 6  | Use EXPLAIN ANALYZE to inspect a query.                                    | Medium     |
| 7  | Rewrite a non-SARGable date query using a date range.                      | Medium     |
| 8  | Perform a bank transfer using a transaction.                               | Medium     |
| 9  | Demonstrate ROLLBACK and SAVEPOINT.                                        | Medium     |
| 10 | Explain dirty reads and non-repeatable reads with concurrent transactions. | Medium     |
| 11 | Solve the lost-update problem using atomic UPDATE.                         | Hard       |
| 12 | Demonstrate optimistic locking using a version number.                     | Hard       |
| 13 | Demonstrate pessimistic locking using SELECT FOR UPDATE.                   | Hard       |
| 14 | Create a deadlock scenario using two transactions.                         | Hard       |
| 15 | Explain how MVCC helps concurrent readers and writers.                     | Hard       |

# **Part 17 — SQL Course Progress**

**SQL + DBMS Mastery**

9 / 10 lessons

01\. SQL Fundamentals

Covered

02\. Aggregate Functions, GROUP BY, HAVING

Covered

03\. SQL JOINs

Covered

04\. Subqueries

Covered

05\. CTEs and Recursive CTEs

Covered

06\. Window Functions

Covered

07\. Advanced SQL Functions

Covered

08\. Database Design and Normalisation

Covered

09\. Indexes, Transactions and Optimisation

Covered

**10. Company OAs and Mock Interviews**

Next

# **Next Lesson — Lesson 10: SQL Company OAs and Interview Mastery**

Final course module

This lesson will bring all the concepts together.

**Section A — OA problem patterns**

1. Second and Nth-highest salary.
2. Top N per department.
3. Employees above department average.
4. Consecutive login streaks.
5. Duplicate detection and deduplication.
6. Customer retention and churn.
7. Running totals and moving averages.
8. Monthly revenue growth.
9. Gaps and islands.
10. Anti-join and existence problems.

**Section B — LeetCode SQL practice**

- SELECT and filtering.
- JOINs.
- Aggregation.
- Subqueries.
- Window functions.
- Advanced SQL.

**Section C — Interview preparation**

- SQL query writing rounds.
- DBMS theoretical questions.
- Query debugging.
- Schema design questions.
- Query optimisation scenarios.
- Transactions and concurrency scenarios.

**Section D — Mock assessments**

- Easy questions: syntax and filtering.
- Medium questions: joins, grouping and subqueries.
- Hard questions: window functions, recursive queries and multi-step analysis.

**My recommendation:** Before advancing, make sure you can explain ACID, isolation levels, MVCC, indexes, B-Trees, locking and deadlocks without relying on memorised definitions. These are particularly important for backend/SDE interviews.