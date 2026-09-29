# **SQL Interview Mastery — Part 4**

## **Questions 51–75: Indexing, Transactions, Locking and Concurrency**

Advanced DBMS

SDE / Backend interviews

PostgreSQL

This is one of the most important sections for backend engineering interviews.

Interviewers use these topics to determine whether you understand what happens inside a database when multiple users access and modify the same data simultaneously.

# **Section 6 — Indexing and Query Performance**

## **Q51. What is an index in a database?**

**Answer:** An index is an additional data structure that helps the database locate records efficiently without scanning the entire table. It improves certain read operations but requires additional storage and maintenance during data modifications.

Example:

```
CREATE INDEX idx_employee_salary
ON employees(salary);
```

Without an index:

```
Scan table → Check salary → Return matching rows
```

With a suitable index:

```
Search index → Locate matching rows → Retrieve data
```

**Important:** An index does not guarantee faster execution. PostgreSQL may choose a sequential scan if that is cheaper.

## **Q52. What is a B-Tree index?**

**Answer:** A B-Tree index is a balanced tree structure that maintains sorted keys and supports efficient equality, range and ordering operations. It is PostgreSQL's default index type and is suitable for most general-purpose database queries.

Example:

```
CREATE INDEX idx_salary
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
ORDER BY salary DESC;
```

**Why is it called balanced?**

The tree maintains its structure so that searches, insertions and deletions can generally be performed in logarithmic time relative to the number of indexed entries.

### **Common B-Tree use cases**

- Employee lookup by salary.
- Finding orders within a date range.
- Searching by customer ID.
- Sorting records.
- Finding minimum and maximum values.

## **Q53. What is a Hash index?**

**Answer:** A Hash index uses a hash function to map indexed values to hash buckets. It is primarily designed for equality comparisons and does not provide the ordered traversal required for range searches or general sorting.

Example:

```
CREATE INDEX idx_employee_hash
ON employees
USING HASH(emp_id);
```

Suitable query:

```
SELECT *
FROM employees
WHERE emp_id = 101;
```

Not suitable for:

```
SELECT *
FROM employees
WHERE salary > 50000;
```

### **B-Tree vs Hash**

| B-Tree                 | Hash                              |
| ---------------------- | --------------------------------- |
| Equality searches      | Equality searches                 |
| Range searches         | Not supported                     |
| Supports ordered scans | Does not provide sorted key order |
| General-purpose        | Specialized                       |
| PostgreSQL default     | Must be explicitly selected       |

**Interview tip:** In most ordinary application scenarios, B-Tree is the starting point. Don't assume Hash is automatically faster for equality searches.

## **Q54. What is a GIN index?**

**Answer:** GIN stands for Generalized Inverted Index. It indexes composite values by maintaining entries for their components, making it useful for searching JSONB documents, arrays and full-text search data.

Example:

```
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    profile JSONB
);
```

Create index:

```
CREATE INDEX idx_profile_gin
ON user_profiles
USING GIN(profile);
```

Query:

```
SELECT *
FROM user_profiles
WHERE profile @> '{"skills":["Python"]}'::jsonb;
```

GIN is useful when you need to search within the contents of a structured value.

Other applications include:

- JSONB containment queries.
- Array membership.
- Full-text search.
- Trigram searches with suitable operator classes.

## **Q55. What is a GiST index?**

**Answer:** GiST stands for Generalized Search Tree. It is an extensible index framework supporting different search strategies, particularly spatial, geometric and range-based operations, as well as nearest-neighbour searches with suitable operator classes.

Example using a range type:

```
CREATE TABLE reservations (
    reservation_id INT PRIMARY KEY,
    booking_period TSRANGE
);

CREATE INDEX idx_booking_period
ON reservations
USING GIST(booking_period);
```

Find reservations overlapping a particular time interval:

```
SELECT *
FROM reservations
WHERE booking_period &&
      TSRANGE(
          '2026-11-21 10:00',
          '2026-11-21 12:00'
      );
```

The `&&` operator checks whether two ranges overlap.

**Common use cases:**

- Geographical and spatial searches.
- Date and timestamp range overlap.
- Geometric data.
- Some full-text and trigram operations.

## **Q56. What is a BRIN index?**

**Answer:** BRIN stands for Block Range Index. It stores summaries of values across groups of table pages rather than individual row locations, making it particularly useful for very large tables where indexed values correlate with physical row order.

Example:

```
CREATE INDEX idx_transactions_date
ON transactions
USING BRIN(transaction_date);
```

Imagine a transaction table populated chronologically.

| Physical page range | Approximate date range |
| ------------------- | ---------------------- |
| Pages 1–100         | January                |
| Pages 101–200       | February               |
| Pages 201–300       | March                  |

When searching for March records, PostgreSQL may skip page ranges whose summaries show they cannot contain matching values.

### **Comparison of PostgreSQL index types**

| Index   | Best suited for                                                      |
| ------- | -------------------------------------------------------------------- |
| B-Tree  | Equality, range and ordering                                         |
| Hash    | Equality comparisons                                                 |
| GIN     | Multi-component values, JSONB, arrays and full-text                  |
| GiST    | Spatial, geometric and range-oriented searches                       |
| BRIN    | Very large, physically correlated tables                             |
| SP-GiST | Certain partitionable search structures, such as quadtrees and tries |

**Interview takeaway:** Choose an index based on the operators and data distribution your application actually uses.

## **Q57. What is a composite index?**

**Answer:** A composite index is an index created on multiple columns. It is useful when queries frequently filter or sort by combinations of attributes, and the column order determines which search patterns it supports efficiently.

Example:

```
CREATE INDEX idx_department_salary
ON employees(department_id, salary);
```

Useful query:

```
SELECT *
FROM employees
WHERE department_id = 10
  AND salary > 60000;
```

### **Why does column order matter?**

Consider:

```
(department_id, salary)
```

It is organized primarily by department ID, then by salary within each department.

This is a natural fit for:

```
WHERE department_id = 10
```

and:

```
WHERE department_id = 10
  AND salary > 60000
```

But a query filtering only by salary may not benefit as much.

The conventional **leftmost-prefix principle** is a useful starting point for understanding B-Tree composite indexes. PostgreSQL may use additional strategies in some situations, so always inspect the actual execution plan.

## **Q58. What is a partial index?**

**Answer:** A partial index indexes only the rows satisfying a specified predicate. It reduces index size and maintenance overhead for workloads that frequently access a specific subset of a table.

Example:

```
CREATE INDEX idx_pending_orders
ON orders(order_date)
WHERE status = 'Pending';
```

Useful query:

```
SELECT *
FROM orders
WHERE status = 'Pending'
  AND order_date >= DATE '2026-11-01';
```

It is particularly useful when only a small portion of the table satisfies the indexed condition.

## **Q59. What is an expression index?**

**Answer:** An expression index stores the result of a specified expression rather than indexing a column's raw value. It is useful when queries repeatedly filter or sort using functions or calculations applied to columns.

Example:

```
CREATE INDEX idx_lower_email
ON users(LOWER(email));
```

Query:

```
SELECT *
FROM users
WHERE LOWER(email) = 'parth@example.com';
```

The indexed expression matches the query expression.

Other use cases:

```
CREATE INDEX idx_trimmed_name
ON employees(TRIM(emp_name));
```

## **Q60. What is a covering index?**

**Answer:** A covering index contains the indexed columns and all additional columns required by a query. It can help the database answer a query directly from the index without fetching the corresponding table rows.

PostgreSQL example:

```
CREATE INDEX idx_department_covering
ON employees(department_id)
INCLUDE (emp_id, salary);
```

Query:

```
SELECT emp_id, salary
FROM employees
WHERE department_id = 10;
```

This may allow an Index Only Scan.

**Important PostgreSQL detail:** An Index Only Scan can still perform heap fetches if the visibility map does not indicate that the relevant pages are all-visible.

## **Q61. What is index selectivity?**

**Answer:** Selectivity describes how effectively a search condition narrows down the rows being considered. A condition matching a very small proportion of rows is highly selective, while one matching most rows has low selectivity.

Example:

Suppose a table contains 1,000,000 employees.

| Condition         | Matching rows | Approximate selectivity |
| ----------------- | ------------- | ----------------------- |
| `emp_id = 101`    | 1             | Very high               |
| `city = 'Bhopal'` | 200,000       | Lower                   |
| `gender = 'Male'` | 500,000       | Lower                   |

A selective index lookup can be very efficient because relatively few rows need to be fetched.

**Don't confuse these terms:**

- Selectivity: How small the matching fraction is.
- Cardinality: The number of distinct values or, in a query plan, sometimes the number of result rows depending on context.

# **Section 7 — ACID Properties and Transactions**

## **Q62. What are ACID properties?**

**Answer:** ACID represents Atomicity, Consistency, Isolation and Durability. These properties describe how database transactions preserve correctness and reliability, especially when multiple users execute operations concurrently or failures occur.

# **A**

**Atomicity**

All or nothing.

# **C**

**Consistency**

Valid state to valid state.

# **I**

**Isolation**

Controlled interaction between transactions.

# **D**

**Durability**

Committed changes survive failures.

Let's understand each property with a real-world example.

## **Q63. What is Atomicity?**

**Answer:** Atomicity ensures that all operations within a transaction either complete successfully or are rolled back together. A partially completed transaction must not leave the database in an intermediate state.

### **Example: Bank transfer**

Account A has ₹10,000.

Account B has ₹5,000.

Transfer ₹2,000 from A to B.

```
BEGIN;

UPDATE accounts
SET balance = balance - 2000
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 2000
WHERE account_id = 2;

COMMIT;
```

Expected outcome:

| Account | Before | After |
| ------- | ------ | ----- |
| A       | 10,000 | 8,000 |
| B       | 5,000  | 7,000 |

What happens when the second update fails?

The transaction must be rolled back rather than leaving ₹2,000 deducted from A without being credited to B.

```
ROLLBACK;
```

## **Q64. What is Consistency?**

**Answer:** Consistency ensures that transactions preserve database-defined integrity constraints and required business invariants. A successful transaction moves the database from one valid state to another rather than violating established rules.

Example:

```
CREATE TABLE accounts (
    account_id INT PRIMARY KEY,
    balance NUMERIC CHECK (balance >= 0)
);
```

This constraint prevents a committed row from having a negative balance.

Other examples:

- Foreign-key validity.
- Unique email addresses.
- Valid product quantities.
- Required fields not being NULL.

**Important:** The database can enforce only the integrity rules that have actually been implemented. Business invariants that aren't represented by constraints or transactional application logic are not automatically guaranteed.

## **Q65. What is Isolation?**

**Answer:** Isolation controls how concurrent transactions interact with each other. It prevents or limits anomalies caused by simultaneous reads and writes, depending on the isolation level selected by the database.

Example:

Two users attempt to buy the last available ticket.

Without appropriate concurrency control, both requests might read a quantity of one and both proceed.

Isolation and locking mechanisms help prevent this.

## **Q66. What is Durability?**

**Answer:** Durability ensures that once a transaction is successfully committed, its changes survive subsequent failures within the database's durability guarantees. PostgreSQL uses Write-Ahead Logging to support crash recovery and durable commits.

Example:

A customer completes a payment.

The server crashes immediately afterward.

After database recovery, the committed payment record should still exist, assuming the configured durability guarantees are being honoured.

# **Section 8 — Transactions and Isolation Levels**

## **Q67. What are COMMIT, ROLLBACK and SAVEPOINT?**

**Answer:** COMMIT permanently commits the current transaction, ROLLBACK reverses its uncommitted changes, and SAVEPOINT creates an intermediate recovery point that allows partial rollback within a transaction.

Example:

```
BEGIN;

INSERT INTO customers
VALUES (101, 'Parth', 'Bhopal');

SAVEPOINT customer_inserted;

INSERT INTO orders
VALUES (1001, 101, CURRENT_DATE, 'Pending', 5000);

ROLLBACK TO SAVEPOINT customer_inserted;

COMMIT;
```

In this example:

- The customer insertion remains.
- The order insertion is rolled back.
- The customer insertion is committed.

## **Q68. What are transaction isolation levels?**

**Answer:** Transaction isolation levels define which effects of concurrent transactions a transaction is allowed to observe. Higher isolation generally offers stronger protection against anomalies but can increase contention or transaction retries.

The four standard SQL isolation levels are:

| Isolation level  | Dirty reads                | Non-repeatable reads | Phantom reads                        |
| ---------------- | -------------------------- | -------------------- | ------------------------------------ |
| Read Uncommitted | Possible in standard model | Possible             | Possible                             |
| Read Committed   | Not permitted              | Possible             | Possible                             |
| Repeatable Read  | Not permitted              | Not permitted        | Not permitted in standard guarantees |
| Serializable     | Not permitted              | Not permitted        | Not permitted                        |

### **PostgreSQL-specific behaviour**

PostgreSQL maps `READ UNCOMMITTED` to `READ COMMITTED`.

Its isolation behaviour is:

| PostgreSQL level | Behaviour                                                  |
| ---------------- | ---------------------------------------------------------- |
| Read Committed   | Each statement gets a new snapshot                         |
| Repeatable Read  | Transaction uses a consistent snapshot                     |
| Serializable     | Detects serialization anomalies and may abort transactions |

PostgreSQL's Repeatable Read is stronger than the minimum standard requirements regarding phantom reads, but it can still permit serialization anomalies such as certain write-skew situations.

## **Q69. Explain dirty reads, non-repeatable reads and phantom reads.**

These are extremely common DBMS viva and interview questions.

### **A. Dirty read**

**Answer:** A dirty read occurs when one transaction reads a value written by another transaction before that transaction commits. If the writing transaction rolls back, the first transaction has observed data that was never committed.

Example:

**Transaction A**

UPDATE balance to 500

Not committed yet

**Transaction B**

Reads balance = 500

**Transaction A**

ROLLBACK

Actual balance returns to 1,000.

Dirty read occurred

PostgreSQL does not permit dirty reads at any supported isolation level.

### **B. Non-repeatable read**

**Answer:** A non-repeatable read occurs when a transaction reads the same row twice and obtains different committed values because another transaction updated or deleted that row between the reads.

Example:

```
Transaction A:
SELECT balance → 1000

Transaction B:
UPDATE balance = 500
COMMIT

Transaction A:
SELECT balance → 500
```

The two reads return different values.

PostgreSQL's Repeatable Read prevents this for ordinary snapshot reads.

### **C. Phantom read**

**Answer:** A phantom read occurs when a transaction repeats a query returning a set of rows and observes additional or missing rows because another transaction inserted or deleted records matching the query condition.

Example:

Transaction A:

```
SELECT *
FROM employees
WHERE salary > 50000;
```

Returns 5 rows.

Transaction B:

```
INSERT INTO employees
VALUES (101, 'Parth', 10, 80000, 'Bhopal', CURRENT_DATE);

COMMIT;
```

Transaction A repeats the query and sees 6 rows under an isolation model that allows phantoms.

PostgreSQL Repeatable Read uses a stable transaction snapshot, so this ordinary phantom does not appear in the same transaction snapshot.

### **Quick comparison**

| Anomaly             | What changes?                            |
| ------------------- | ---------------------------------------- |
| Dirty read          | Uncommitted data is read                 |
| Non-repeatable read | Existing row value changes between reads |
| Phantom read        | Matching result-set membership changes   |

## **Q70. What is MVCC?**

**Answer:** MVCC stands for Multi-Version Concurrency Control. It maintains multiple row versions so transactions can read consistent snapshots while concurrent transactions modify data, reducing the need for readers and writers to block each other.

This is a fundamental PostgreSQL concept.

### **Understand without jargon**

Suppose:

Employee 101 has salary ₹50,000.

Transaction A reads the salary.

Transaction B updates it to ₹60,000.

With PostgreSQL MVCC, Transaction A can continue seeing the version visible to its snapshot, while a later transaction can see the updated version after it commits.

**Conceptual row versions**

**Version 1**

Salary: ₹50,000

Visible to an older snapshot

**Version 2**

Salary: ₹60,000

Visible to a later snapshot after commit

### **Why is MVCC useful?**

- Readers generally do not block writers.
- Writers generally do not block ordinary snapshot readers.
- Transactions can read consistent snapshots.
- Concurrent workload throughput can improve.

### **What is VACUUM?**

PostgreSQL updates typically create new row versions rather than overwriting old versions immediately.

Old versions that are no longer needed by any active transaction eventually become dead tuples.

`VACUUM` helps reclaim space for reuse and maintain visibility information.

```
VACUUM employees;
```

`VACUUM FULL` rewrites the table and requires a stronger lock, so it has different operational consequences.

**Interview takeaway:** MVCC allows concurrent access using row versions and snapshots. VACUUM maintains database health by cleaning up obsolete versions when they are no longer needed.

# **Section 9 — Locks and Concurrent Transactions**

## **Q71. What is a database lock?**

**Answer:** A lock is a concurrency-control mechanism that restricts conflicting operations on database objects or rows. It helps prevent inconsistent modifications and coordinates transactions accessing shared data.

### **Table-level locks in PostgreSQL**

PostgreSQL has several table-lock modes, including:

- ACCESS SHARE
- ROW SHARE
- ROW EXCLUSIVE
- SHARE UPDATE EXCLUSIVE
- SHARE
- SHARE ROW EXCLUSIVE
- EXCLUSIVE
- ACCESS EXCLUSIVE

You don't need to memorise every conflict relationship initially, but understand the purpose.

For example:

```
SELECT *
FROM employees;
```

Takes an `ACCESS SHARE` table lock.

```
UPDATE employees
SET salary = 50000
WHERE emp_id = 101;
```

Takes a `ROW EXCLUSIVE` table lock and row-level locks on modified rows.

An ordinary SELECT can generally coexist with an UPDATE because PostgreSQL's MVCC allows snapshot reads.

## **Q72. What is SELECT FOR UPDATE?**

**Answer:** SELECT FOR UPDATE retrieves selected rows and obtains row-level locks that prevent conflicting updates, deletes and row-locking operations until the transaction ends. It is useful for coordinating read-modify-write workflows.

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

The row lock helps prevent another transaction from modifying the same row concurrently while the transaction performs its operation.

### **Other PostgreSQL row-locking clauses**

| Clause              | Purpose                                                  |
| ------------------- | -------------------------------------------------------- |
| `FOR UPDATE`        | Strong row lock for modifications                        |
| `FOR NO KEY UPDATE` | Weaker update lock when key columns aren't being changed |
| `FOR SHARE`         | Shared row lock                                          |
| `FOR KEY SHARE`     | Protects key values against conflicting changes          |

### **NOWAIT**

```
SELECT *
FROM accounts
WHERE account_id = 101
FOR UPDATE NOWAIT;
```

If the required row lock cannot be acquired immediately, PostgreSQL raises an error rather than waiting.

### **SKIP LOCKED**

```
SELECT *
FROM jobs
WHERE status = 'Pending'
ORDER BY job_id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

This is useful for concurrent job workers.

A worker skips rows already locked by another worker rather than waiting.

## **Q73. What is optimistic locking vs pessimistic locking?**

**Answer:** Optimistic locking assumes conflicts are uncommon and checks whether data changed before committing an update. Pessimistic locking acquires locks early to prevent conflicting modifications while the operation is in progress.

### **Optimistic locking example**

Add a version column:

```
ALTER TABLE products
ADD COLUMN version INT NOT NULL DEFAULT 1;
```

Application reads:

```
product_id = 101
quantity = 10
version = 3
```

It attempts an update:

```
UPDATE products
SET
    quantity = 9,
    version = version + 1
WHERE product_id = 101
  AND version = 3;
```

Check the affected row count.

- One row updated: success.
- Zero rows updated: another transaction changed the version, so the operation must be retried or rejected.

### **Pessimistic locking example**

```
BEGIN;

SELECT *
FROM products
WHERE product_id = 101
FOR UPDATE;

UPDATE products
SET quantity = quantity - 1
WHERE product_id = 101;

COMMIT;
```

Here, we lock the row before modifying it.

### **Comparison**

| Optimistic                       | Pessimistic                        |
| -------------------------------- | ---------------------------------- |
| Detects conflicts at update time | Acquires locks before modification |
| Good when conflicts are uncommon | Good when contention is expected   |
| May require retries              | Can cause waiting                  |
| Version column commonly used     | Database locks commonly used       |

**Practical use:**

- Optimistic: profile editing, document updates, low-contention records.
- Pessimistic: inventory reservation, account balance modifications and high-contention workflows.

Neither is universally superior.

## **Q74. What is a deadlock?**

**Answer:** A deadlock occurs when two or more transactions wait indefinitely for resources locked by one another, creating a circular dependency. Databases detect deadlocks and typically abort one transaction to break the cycle.

### **Example**

Transaction A:

```
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE account_id = 1;
```

Transaction B:

```
BEGIN;

UPDATE accounts
SET balance = balance - 200
WHERE account_id = 2;
```

Now:

Transaction A:

```
UPDATE accounts
SET balance = balance + 100
WHERE account_id = 2;
```

This waits for Transaction B's lock.

Transaction B:

```
UPDATE accounts
SET balance = balance + 200
WHERE account_id = 1;
```

This waits for Transaction A's lock.

**Circular waiting**

**Transaction A**

Holds Account 1 lock.

Waiting for Account 2.

**Transaction B**

Holds Account 2 lock.

Waiting for Account 1.

Deadlock

Neither transaction can proceed without the other releasing its lock.

PostgreSQL detects the deadlock and aborts one transaction with a deadlock error.

### **How to prevent deadlocks?**

1. Acquire locks in a consistent order.
2. Keep transactions short.
3. Avoid unnecessary operations while holding locks.
4. Create appropriate indexes so updates don't scan unnecessarily large ranges.
5. Handle deadlock errors and retry suitable transactions.

**Important:** Deadlocks are not the same as lock contention.

- Lock contention: Transactions wait for each other, but a circular dependency doesn't necessarily exist.
- Deadlock: Transactions form a circular wait that cannot resolve without intervention.

# **Section 10 — WAL, Checkpoints and Database Recovery**

## **Q75. What is Write-Ahead Logging (WAL)?**

**Answer:** Write-Ahead Logging records database changes in a log before corresponding modified data pages are written to their final storage locations. It supports crash recovery and transaction durability by allowing committed changes to be recovered after a failure.

### **Understand the concept**

Suppose your database updates an account balance from ₹10,000 to ₹8,000.

Rather than relying only on the data page being immediately written to disk, PostgreSQL records the relevant changes in its WAL.

Conceptually:

**WAL process**

**Transaction modifies data**

**WAL records the changes**

Log must be durable before corresponding dirty data pages are written

**Transaction COMMIT**

**Data pages written later**

This is the principle of write-ahead logging:

> The log describing a data-page modification must be safely written before that modified page is written to its data file.

## **What happens after a crash?**

Imagine:

1. Transaction updates an account balance.
2. WAL is safely flushed.
3. Transaction commits.
4. Database crashes before the modified table page is written.

After restarting, PostgreSQL uses WAL to redo the necessary changes and recover the committed state.

## **What is a checkpoint?**

**Answer:** A checkpoint establishes a recovery point by ensuring that modified data pages up to a relevant point have been flushed to storage and recording checkpoint information in WAL. It reduces the amount of WAL that must be replayed during crash recovery.

PostgreSQL performs checkpoints automatically and can also execute them manually:

```
CHECKPOINT;
```

Checkpoints are important because replaying an enormous amount of WAL during recovery can take considerable time.

## **WAL vs Checkpoint**

| WAL                                   | Checkpoint                                    |
| ------------------------------------- | --------------------------------------------- |
| Records database changes              | Establishes a recovery point                  |
| Supports redo and crash recovery      | Reduces recovery work                         |
| Written as database changes occur     | Occurs periodically or when requested         |
| Fundamental to transaction durability | Helps manage recovery and dirty-page flushing |

### **Additional interview point: WAL archiving**

PostgreSQL can archive completed WAL segments.

This supports:

- Point-in-time recovery (PITR).
- Recovery using a base backup plus archived WAL.
- Restoring a database to an earlier time, subject to backup and archive availability.

# **High-priority interview revision table**

| Question | Key concept to remember                             |
| -------- | --------------------------------------------------- |
| Q51      | Index improves selected lookup operations           |
| Q52      | B-Tree: equality, range and ordering                |
| Q53      | Hash: equality-oriented                             |
| Q54      | GIN: inverted indexing for JSONB, arrays, full-text |
| Q55      | GiST: spatial, range and generalized search         |
| Q56      | BRIN: large, physically correlated tables           |
| Q57      | Composite index and leading columns                 |
| Q58      | Index only rows satisfying a predicate              |
| Q59      | Index on a computed expression                      |
| Q60      | Cover query fields with index attributes            |
| Q61      | Selectivity means how much a condition filters      |
| Q62      | ACID                                                |
| Q63      | All or nothing                                      |
| Q64      | Valid database state                                |
| Q65      | Concurrent transaction isolation                    |
| Q66      | Committed changes survive failures                  |
| Q67      | COMMIT, ROLLBACK, SAVEPOINT                         |
| Q68      | Four standard isolation levels                      |
| Q69      | Dirty, non-repeatable and phantom reads             |
| Q70      | MVCC and snapshots                                  |
| Q71      | Database locks                                      |
| Q72      | SELECT FOR UPDATE                                   |
| Q73      | Optimistic vs pessimistic locking                   |
| Q74      | Circular waiting                                    |
| Q75      | WAL and checkpoints                                 |

# **Test your understanding**

**DBMS interview checkpoint**

8 questions

**Q1. Which PostgreSQL index type is most commonly used for ordinary equality and range searches?**

GIN

B-Tree

BRIN

GiST

**Q2. Which index is particularly useful for JSONB containment queries?**

Hash

BRIN

GIN

B-Tree only

**Q3. What does Atomicity guarantee?**

Data is always sorted

All transaction operations succeed or roll back together

Transactions never wait

Every query uses an index

**Q4. Which PostgreSQL isolation level provides a consistent transaction snapshot across ordinary reads?**

READ COMMITTED

REPEATABLE READ

READ UNCOMMITTED

None

**Q5. What does SELECT FOR UPDATE do?**

Creates a permanent index

Locks selected rows against conflicting modifications

Deletes selected records

Prevents all SELECT operations globally

**Q6. Which technique is commonly used for optimistic locking?**

Version column

TRUNCATE

CROSS JOIN

BRIN index

**Q7. What is a deadlock?**

A slow sequential scan

Circular waiting between transactions

A missing index

A NULL column

**Q8. Why does PostgreSQL use WAL?**

To replace indexes

To support durability and crash recovery

To avoid all locks

To normalize tables

Check answers

## **What's next — Questions 76–100**

The final 25 questions will cover:

1. Query optimization and execution plans.
2. Nested Loop, Hash Join and Merge Join.
3. N+1 query problem.
4. Database connection pooling.
5. Views, materialized views and triggers.
6. Stored procedures and functions.
7. SQL injection and prepared statements.
8. Partitioning and sharding.
9. Database replication and read replicas.
10. Real backend scenarios: ticket booking, inventory overselling, banking transfers and concurrent requests.

After Question 100, we'll have completed the interview question bank. Then we'll move into a full mock SQL + DBMS interview with practical query-writing exercises, follow-up questions and scoring.

# **SQL Interview Mastery — Part 4**

## **Questions 76–100: Query Optimization, Distributed Databases and Backend Engineering**

Final 25 questions

SDE / Backend interviews

PostgreSQL

This is the final section of the 100-question SQL and DBMS interview bank.

The focus now shifts towards production engineering: how databases execute queries, how applications interact with databases, and how to prevent real-world data consistency problems.

# **Section 11 — Query Optimization and Execution Plans**

## **Q76. What is a query execution plan?**

**Answer:** A query execution plan describes the operations the database performs to execute a SQL query. It shows scans, joins, sorting, aggregation and other operations selected by the optimizer to retrieve the requested data.

Example:

```
EXPLAIN
SELECT *
FROM employees
WHERE department_id = 10;
```

Possible output:

```
Seq Scan on employees
  Filter: (department_id = 10)
```

Common execution-plan operations:

| Operation        | Meaning                                             |
| ---------------- | --------------------------------------------------- |
| Seq Scan         | Reads table pages sequentially                      |
| Index Scan       | Uses an index to locate rows                        |
| Index Only Scan  | Attempts to retrieve results directly from an index |
| Bitmap Heap Scan | Uses a bitmap of matching row locations             |
| Nested Loop      | Joins by repeatedly scanning an inner input         |
| Hash Join        | Joins using a hash table                            |
| Merge Join       | Joins ordered inputs                                |
| Sort             | Sorts rows                                          |
| Aggregate        | Calculates aggregate results                        |

## **Q77. Difference between EXPLAIN and EXPLAIN ANALYZE?**

**Answer:** EXPLAIN displays the estimated execution plan without executing the SELECT query. EXPLAIN ANALYZE executes the statement and reports actual execution statistics, allowing comparison between estimated and observed performance.

```
EXPLAIN
SELECT *
FROM employees
WHERE salary > 50000;
```

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM employees
WHERE salary > 50000;
```

Important measurements:

- Estimated rows versus actual rows.
- Execution time.
- Number of loops.
- Buffer hits and reads.
- Rows removed by filters.

**Interview follow-up:** What if estimated rows are 100 but actual rows are 100,000?

The optimizer's cardinality estimate is inaccurate. Investigate statistics, data distribution and correlated predicates because incorrect estimates can lead to poor execution plans.

## **Q78. What is a Nested Loop Join?**

**Answer:** A Nested Loop Join iterates through rows of one input and searches for matching rows in another input. It is particularly effective when the outer input is small and the inner input has a suitable index.

Conceptually:

```
For every row in Table A:
    Find matching rows in Table B
```

Example:

```
SELECT *
FROM customers c
JOIN orders o
    ON o.customer_id = c.customer_id;
```

Suppose:

- Customers returned: 10.
- Orders table: 1 million rows.
- An appropriate index exists on `orders.customer_id`.

An indexed nested loop can be efficient because it performs relatively few targeted lookups.

**When can it become expensive?**

When the outer input contains many rows and each iteration triggers a costly scan of the inner input.

## **Q79. What is a Hash Join?**

**Answer:** A Hash Join builds an in-memory hash table from one input using the join key, then probes it using rows from the other input. It is useful for equality joins when sufficiently efficient hash processing is possible.

Conceptually:

```
Build hash table from Table A
             ↓
Read Table B
             ↓
Find matching hash entries
             ↓
Return matching rows
```

Example:

```
SELECT *
FROM employees e
JOIN departments d
    ON e.department_id = d.department_id;
```

Hash Join is useful when the database is joining relatively large inputs using equality conditions.

**Important:** Hash Join does not require an index on the join key. If memory is insufficient, PostgreSQL may use multiple batches and temporary storage.

## **Q80. What is a Merge Join?**

**Answer:** A Merge Join combines two inputs sorted by their join keys. It advances through both ordered inputs and matches equal keys, making it useful when the inputs are already ordered or sorting them is cost-effective.

Conceptually:

```
Table A: 10, 20, 30, 40
Table B: 10, 30, 40, 50

Compare keys → Advance → Match
```

### **Join algorithm comparison**

| Nested Loop                                   | Hash Join                    | Merge Join                  |
| --------------------------------------------- | ---------------------------- | --------------------------- |
| Repeated lookups                              | Hash table                   | Ordered traversal           |
| Good for small outer inputs                   | Good for many equality joins | Good for ordered inputs     |
| Benefits from inner index                     | Does not require index       | Benefits from sorted inputs |
| Can become expensive with many repeated scans | Can spill to disk            | May require sorting         |

**Interview takeaway:** There is no universally best join algorithm. The optimizer selects one based on estimated cardinality, costs, indexes and available memory.

## **Q81. What is query optimization?**

**Answer:** Query optimization is the process of improving database query execution by reducing unnecessary scans, joins, sorting, data transfer and computational work. It involves examining execution plans, query structure, indexes and database statistics.

Example:

Less efficient:

```
SELECT *
FROM employees
WHERE salary * 12 > 600000;
```

More index-friendly equivalent:

```
SELECT *
FROM employees
WHERE salary > 50000;
```

Other optimization techniques:

- Avoid unnecessary columns with `SELECT *`.
- Use appropriate indexes.
- Filter data early where logically valid.
- Avoid duplicate-producing joins.
- Preaggregate one-to-many relationships when necessary.
- Use suitable pagination.
- Inspect execution plans.

## **Q82. What is a SARGable query?**

**Answer:** A SARGable query uses search predicates that allow the database to efficiently navigate an index. Direct comparisons and suitable range conditions are generally more index-friendly than applying functions to indexed columns.

Suppose an ordinary B-Tree index exists on `hire_date`.

Less index-friendly:

```
SELECT *
FROM employees
WHERE EXTRACT(YEAR FROM hire_date) = 2026;
```

Better:

```
SELECT *
FROM employees
WHERE hire_date >= DATE '2026-01-01'
  AND hire_date < DATE '2027-01-01';
```

The second query directly defines a range on the indexed column.

## **Q83. Why might a database ignore an available index?**

**Answer:** The optimizer may choose a sequential scan when it estimates that reading most table pages is cheaper than performing index lookups and fetching numerous heap rows. Index presence alone does not guarantee index usage.

Common reasons:

1. The table is small.
2. A large percentage of rows qualify.
3. The indexed expression does not match the query.
4. Statistics suggest a sequential scan is cheaper.
5. The query uses an unsupported or unsuitable operator.
6. The index is not selective enough for the workload.

Useful diagnostic:

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM employees
WHERE department_id = 10;
```

# **Section 12 — Backend Application and Database Performance**

## **Q84. What is the N+1 query problem?**

**Answer:** The N+1 query problem occurs when an application executes one query to fetch a collection and then executes another query for every individual record to retrieve related data. It increases round trips and can significantly degrade performance.

Example:

Suppose you need 100 customers and their orders.

**Problematic approach:**

```
customers = get_all_customers()  # 1 queryfor customer in customers:    orders = get_orders(customer.id)  # 100 queries
```

Total:

\\[ 1+100=101\text{ queries} \\]

### **Better approach 1 — JOIN**

```
SELECT
    c.customer_id,
    c.customer_name,
    o.order_id,
    o.total_amount
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id;
```

### **Better approach 2 — Batch loading**

First fetch customers, then fetch orders for all selected customer IDs:

```
SELECT *
FROM orders
WHERE customer_id = ANY(ARRAY[101, 102, 103]);
```

**Interview takeaway:** Use JOINs, batch loading or ORM eager loading to reduce unnecessary database round trips.

## **Q85. What is batch processing in SQL?**

**Answer:** Batch processing groups multiple database operations into fewer requests or transactions. It reduces communication overhead, improves throughput and can avoid repeatedly executing the same database operation for individual records.

Instead of:

```
INSERT INTO products VALUES (101, 'Laptop', 50000);
INSERT INTO products VALUES (102, 'Mouse', 1000);
INSERT INTO products VALUES (103, 'Keyboard', 2000);
```

Use:

```
INSERT INTO products (
    product_id,
    product_name,
    price
)
VALUES
    (101, 'Laptop', 50000),
    (102, 'Mouse', 1000),
    (103, 'Keyboard', 2000);
```

For large imports, PostgreSQL also supports `COPY`, which is often more efficient than individual INSERT statements.

```
COPY products
FROM '/path/products.csv'
WITH (FORMAT CSV, HEADER TRUE);
```

The file path in this example is interpreted by the database server, not necessarily your application machine.

## **Q86. What is connection pooling?**

**Answer:** Connection pooling maintains a controlled collection of reusable database connections. Applications borrow available connections, execute queries and return them to the pool instead of repeatedly establishing new connections for every request.

Example architecture:

**FastAPI / Flask application**

**Connection pool**

Controls available connections and waiters

**PostgreSQL server**

Benefits:

- Connection reuse.
- Lower connection-establishment overhead.
- Controlled concurrency.
- Better resource management.

A popular PostgreSQL connection pooler is PgBouncer.

**Interview follow-up:** Can too many connections reduce performance?

Yes. Excessive database connections consume memory and scheduling resources and can increase contention. Pool sizes should be configured according to application concurrency and database capacity.

## **Q87. What is a database view?**

**Answer:** A view is a named query that provides a virtual table interface over underlying database objects. It simplifies complex queries, supports controlled data exposure and creates reusable logical interfaces without storing a separate materialized result set.

Example:

```
CREATE VIEW completed_orders AS
SELECT
    order_id,
    customer_id,
    total_amount
FROM orders
WHERE status = 'Completed';
```

Usage:

```
SELECT *
FROM completed_orders;
```

Benefits:

- Query simplification.
- Reusability.
- Access control.
- Logical abstraction.

## **Q88. What is a materialized view?**

**Answer:** A materialized view physically stores the results of a query. It can accelerate expensive reporting and aggregation queries, but its contents become stale when source data changes and must be refreshed.

```
CREATE MATERIALIZED VIEW department_salary_summary AS
SELECT
    department_id,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;
```

Refresh:

```
REFRESH MATERIALIZED VIEW department_salary_summary;
```

### **View vs Materialized View**

| View                          | Materialized view                 |
| ----------------------------- | --------------------------------- |
| Stores query definition       | Stores query results              |
| Reads current underlying data | Reflects data at the last refresh |
| No separate result storage    | Requires additional storage       |
| No refresh operation          | Needs refresh                     |
| Useful for abstraction        | Useful for precomputed reporting  |

## **Q89. What is a database trigger?**

**Answer:** A trigger automatically executes a specified database function when a configured table or view event occurs. It is useful for auditing, validation and automatic timestamp updates, but excessive trigger logic can make application behaviour harder to understand.

Example:

```
CREATE FUNCTION update_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_update_timestamp
BEFORE UPDATE ON employees
FOR EACH ROW
EXECUTE FUNCTION update_timestamp();
```

## **Q90. Difference between a stored function and a stored procedure?**

**Answer:** A function has a return definition and can be used within SQL expressions, while a procedure is invoked with CALL and supports procedural database operations. PostgreSQL procedures can control transactions in supported invocation contexts.

Example function:

```
CREATE FUNCTION annual_salary(monthly NUMERIC)
RETURNS NUMERIC
LANGUAGE SQL
AS $$
    SELECT monthly * 12;
$$;

SELECT annual_salary(50000);
```

Example procedure:

```
CREATE PROCEDURE increase_salary(
    dept_id INT,
    increment NUMERIC
)
LANGUAGE SQL
AS $$
    UPDATE employees
    SET salary = salary + increment
    WHERE department_id = dept_id;
$$;

CALL increase_salary(10, 5000);
```

# **Section 13 — Security and Distributed Database Concepts**

## **Q91. What is SQL injection?**

**Answer:** SQL injection is a security vulnerability in which untrusted input changes the structure or meaning of an SQL statement. It can allow attackers to bypass authentication, access unauthorized records or modify database contents.

Dangerous Python example:

```
username = input("Username: ")query = (    "SELECT * FROM users "    + "WHERE username = '" + username + "'")
```

An attacker could supply:

```
' OR '1'='1
```

This can change the intended condition.

### **Correct approach: Parameterized queries**

```
cursor.execute(    "SELECT * FROM users WHERE username = %s",    (username,))
```

The database driver handles parameter binding rather than interpreting user input as part of the SQL syntax.

**Interview takeaway:** Use parameterized queries, validate inputs, follow least-privilege access and avoid constructing SQL by concatenating untrusted input.

## **Q92. What is a prepared statement?**

**Answer:** A prepared statement separates an SQL statement's structure from its input parameters. It allows safe parameter binding, helps prevent value-based SQL injection and can reduce repeated parsing or planning overhead.

PostgreSQL example:

```
PREPARE employee_by_id(INT) AS
SELECT *
FROM employees
WHERE emp_id = $1;
```

Execute:

```
EXECUTE employee_by_id(101);
```

Prepared statements are especially useful when the same query pattern is repeatedly executed with different values.

## **Q93. What is database partitioning?**

**Answer:** Partitioning divides one logical table into smaller physical partitions using a partition key. It can improve manageability and query performance through partition pruning, particularly for large time-series and transaction tables.

Example:

```
CREATE TABLE transactions (
    transaction_id BIGINT,
    transaction_date DATE NOT NULL,
    amount NUMERIC
)
PARTITION BY RANGE(transaction_date);
```

Create a partition:

```
CREATE TABLE transactions_2026_jan
PARTITION OF transactions
FOR VALUES FROM ('2026-01-01')
TO ('2026-02-01');
```

Useful for:

- Large transaction tables.
- Time-series data.
- Historical data archiving.
- Data lifecycle management.

## **Q94. What is database sharding?**

**Answer:** Sharding distributes a dataset across multiple independent database instances or nodes. Each shard stores only part of the overall data, allowing horizontal scaling of storage and workload beyond the capacity of a single database server.

Example:

**Application**

**Shard router**

Routes query based on user_id

**Shard 1**

Users 1–1000

**Shard 2**

Users 1001–2000

**Shard 3**

Users 2001–3000

Common sharding strategies:

- Range-based.
- Hash-based.
- Directory-based.

### **Partitioning vs Sharding**

| Partitioning                           | Sharding                                           |
| -------------------------------------- | -------------------------------------------------- |
| Splits a logical table into partitions | Distributes data across database nodes             |
| Can exist within one PostgreSQL server | Commonly spans multiple servers                    |
| Useful for data management and pruning | Useful for horizontal scaling                      |
| Usually managed by one database        | Requires distributed data routing and coordination |

**Interview takeaway:** Partitioning organizes large tables; sharding distributes data and workload across independent database systems.

## **Q95. What is database replication?**

**Answer:** Replication maintains copies of database data on additional servers. It supports availability, disaster recovery and read scaling, depending on the replication mode and architecture.

A common primary-replica architecture:

**Primary database**

Handles writes

**Replica 1**

Read queries

**Replica 2**

Read queries

### **Types of PostgreSQL replication**

**Physical replication:** Copies the database's physical changes through WAL.

**Logical replication:** Replicates logical changes to selected tables or data sets using publications and subscriptions.

Common uses:

- Disaster recovery.
- High availability.
- Read scaling.
- Data integration.

# **Section 14 — Real Backend Database Scenarios**

These five questions are especially valuable because they connect SQL with application engineering.

## **Q96. What are read replicas, and what is replication lag?**

**Answer:** Read replicas are secondary database instances that receive replicated changes from a primary database and serve read queries. Replication lag occurs when a replica has not yet applied the latest primary changes.

Example:

1. User updates their profile on the primary.
2. Application immediately reads from a replica.
3. The replica hasn't received the update yet.
4. The user sees their old profile.

This is called **stale reading**.

### **How do you handle it?**

- Route critical read-after-write queries to the primary.
- Use synchronous replication where appropriate.
- Track replication lag.
- Design the application to tolerate eventual consistency where possible.

**Interview takeaway:** Read replicas improve read scalability, but asynchronous replication can produce stale reads.

## **Q97. How would you prevent overselling in an inventory system?**

**Problem:** There are five available units of a product. Ten customers attempt to buy five units simultaneously.

A naive implementation:

```
SELECT quantity
FROM inventory
WHERE product_id = 101;
```

Then, in application code, check whether enough stock exists and later update it.

This creates a race condition.

Two requests can both read the same quantity before either update commits.

### **Better solution: Atomic conditional UPDATE**

```
UPDATE inventory
SET quantity = quantity - 5
WHERE product_id = 101
  AND quantity >= 5
RETURNING quantity;
```

Interpretation:

- If a row is returned, stock was successfully reserved.
- If no row is returned, sufficient stock was unavailable.

The condition and update are one database operation.

For a more complete order workflow:

```
BEGIN;

UPDATE inventory
SET quantity = quantity - 5
WHERE product_id = 101
  AND quantity >= 5
RETURNING quantity;

-- Application checks whether a row was returned.
-- If successful:

INSERT INTO orders (
    order_id,
    customer_id,
    status,
    total_amount
)
VALUES (
    5001,
    101,
    'Pending',
    25000
);

COMMIT;
```

If stock cannot be reserved, the application should not insert the order and should roll back or end the transaction without the purchase.

**Interview takeaway:** Protect inventory updates through atomic conditional operations, suitable constraints and transactions—not application-side read-then-write checks alone.

## **Q98. How would you prevent two users from booking the same seat?**

**Problem:** Two users simultaneously attempt to book seat A10 for the same show.

A robust solution involves a database-enforced uniqueness rule.

Assume:

```
CREATE TABLE seat_bookings (
    show_id INT,
    seat_id INT,
    customer_id INT,
    status VARCHAR(20),
    PRIMARY KEY (show_id, seat_id)
);
```

This ensures that the same seat for the same show cannot be represented by two separate rows.

Booking:

```
INSERT INTO seat_bookings (
    show_id,
    seat_id,
    customer_id,
    status
)
VALUES (
    100,
    10,
    501,
    'CONFIRMED'
)
ON CONFLICT (show_id, seat_id)
DO NOTHING
RETURNING *;
```

- One request successfully inserts.
- The competing request gets no inserted row.

The uniqueness constraint, not just an application check, prevents duplicate booking.

### **What about temporary seat holds?**

A production system may need reservation expiry.

One design is to model each seat's current state and lock its row during a booking transaction. The system checks whether the hold is expired, updates the state and commits.

For example:

```
BEGIN;

SELECT *
FROM show_seats
WHERE show_id = 100
  AND seat_id = 10
FOR UPDATE;

-- Check availability and expiry.

UPDATE show_seats
SET status = 'HELD',
    customer_id = 501,
    hold_expires_at = CURRENT_TIMESTAMP + INTERVAL '5 minutes'
WHERE show_id = 100
  AND seat_id = 10;

COMMIT;
```

The application must handle the expiry policy and check the locked row's actual state before making the update.

## **Q99. How would you implement a safe banking transfer?**

**Problem:** Transfer ₹2,000 from account A to account B.

Requirements:

- Source account must have sufficient funds.
- Both updates must succeed together.
- Balance must not become negative.
- Concurrent transfers must be handled safely.

### **Solution**

```
BEGIN;

UPDATE accounts
SET balance = balance - 2000
WHERE account_id = 1
  AND balance >= 2000
RETURNING account_id, balance;
```

The application checks that exactly one row was returned.

If not, it rolls back because the source account is invalid or has insufficient funds.

Then:

```
UPDATE accounts
SET balance = balance + 2000
WHERE account_id = 2
RETURNING account_id, balance;
```

Again, verify that exactly one row was returned. If the destination does not exist, roll back.

Record both sides of the ledger:

```
INSERT INTO transactions (
    account_id,
    amount,
    transaction_type
)
VALUES
    (1, -2000, 'TRANSFER_OUT'),
    (2,  2000, 'TRANSFER_IN');

COMMIT;
```

### **Additional production considerations**

1. Validate that source and destination accounts differ if required.
2. Lock accounts in a consistent order when explicitly locking both.
3. Enforce nonnegative balance with a CHECK constraint.
4. Record immutable ledger entries.
5. Use an idempotency key so retrying a request does not make the same transfer twice.
6. Handle transaction failures with appropriate retries.

**Important:** A financial system should maintain a reliable ledger and reconciliation process; updating account balances alone is not sufficient for production-grade accounting.

## **Q100. What is idempotency, and what is the transactional outbox pattern?**

This question combines two major backend engineering concepts.

### **Part A — Idempotency**

**Answer:** An operation is idempotent when repeating the same request produces the same intended result as executing it once. Idempotency keys prevent duplicate processing when clients retry requests after timeouts or uncertain responses.

Example:

A payment request contains:

```
idempotency_key = payment-abc-123
```

The application stores this key under a unique constraint.

```
CREATE TABLE payment_requests (
    idempotency_key VARCHAR(100) PRIMARY KEY,
    payment_id BIGINT,
    status VARCHAR(30)
);
```

A repeated request with the same key must return or resume the original result instead of creating a second payment.

A unique constraint is the database-level protection against duplicate keys.

### **Part B — Transactional outbox**

Suppose your application processes an order:

1. Insert the order in PostgreSQL.
2. Publish an `OrderCreated` event to Kafka or a message broker.

What if the database commit succeeds but message publishing fails?

Your order exists, but downstream services never receive the event.

This is the **dual-write problem**.

The transactional outbox pattern addresses it.

```
BEGIN;

INSERT INTO orders (
    order_id,
    customer_id,
    status,
    total_amount
)
VALUES (5001, 101, 'Pending', 5000);

INSERT INTO outbox_events (
    event_id,
    event_type,
    payload,
    created_at
)
VALUES (
    9001,
    'OrderCreated',
    '{"order_id":5001,"customer_id":101}'::jsonb,
    CURRENT_TIMESTAMP
);

COMMIT;
```

Both database operations commit together.

A background publisher reads unsent events from the outbox and publishes them to the message broker. It then marks them as processed.

The publisher needs retry handling and should tolerate duplicate delivery, because a crash can occur after publishing but before marking an event as processed.

**Transactional outbox flow**

**Application**

Create order and event

**Single PostgreSQL transaction**

orders + outbox_events

**Outbox publisher**

Reads and publishes pending events

**Message broker**

Kafka / RabbitMQ / other messaging system

**Interview takeaway:** Idempotency prevents duplicate business operations. The transactional outbox prevents database updates and event publication from becoming inconsistent dual writes. The publisher and consumers still need duplicate-delivery handling.

# **Final revision sheet — Questions 76–100**

| Questions | Area                     | Must remember                                   |
| --------- | ------------------------ | ----------------------------------------------- |
| 76–77     | Execution plans          | EXPLAIN, EXPLAIN ANALYZE                        |
| 78–80     | Join algorithms          | Nested Loop, Hash, Merge                        |
| 81–83     | Optimization             | SARGability, cardinality and indexes            |
| 84–86     | Backend performance      | N+1, batch processing, pools                    |
| 87–90     | Database programmability | Views, materialized views, triggers, procedures |
| 91–92     | Security                 | SQL injection, parameter binding                |
| 93–94     | Scaling                  | Partitioning vs sharding                        |
| 95–96     | Replication              | Replicas and replication lag                    |
| 97        | Inventory                | Atomic conditional update                       |
| 98        | Ticket booking           | Unique constraints and row locking              |
| 99        | Banking                  | ACID transaction and ledger                     |
| 100       | Distributed workflows    | Idempotency and transactional outbox            |

# **Final SQL and DBMS knowledge assessment**

**Final 8-question checkpoint**

Questions 76–100

**Q1. Which join algorithm is particularly effective with a small outer input and an indexed inner input?**

Hash Join

Nested Loop Join

Merge Join

Cross Join

**Q2. What is the N+1 query problem?**

One query executed eleven times

One collection query followed by one query for each record

One query containing eleven joins

An indexing problem

**Q3. Which mechanism prevents two customers booking the same unique seat?**

Application-side SELECT alone

Database UNIQUE constraint

ORDER BY

OFFSET

**Q4. What is the difference between partitioning and sharding?**

They are identical

Partitioning divides a logical table; sharding distributes data across independent database nodes

Sharding only means indexing

Partitioning requires Kafka

**Q5. What does replication lag mean?**

The primary cannot accept writes

A replica has not applied the latest primary changes

An index is missing

A transaction has rolled back

**Q6. What does an idempotency key help prevent?**

All SQL errors

Duplicate execution of the same intended business request

Slow joins

Deadlocks in every situation

**Q7. What is the core purpose of the transactional outbox?**

Replace foreign keys

Atomically save business data and an event record in one database transaction

Guarantee exactly-once delivery from any message broker

Create a read replica

**Q8. Why is an atomic conditional inventory UPDATE useful?**

It removes the need for transactions everywhere

It prevents two concurrent requests from both consuming unavailable stock

It creates an automatic index

It makes the database non-relational

Evaluate answers

# **Your 100-question SQL and DBMS interview bank is complete**

# **100 / 100**

**Interview question bank covered**

SQL fundamentals and querying

Covered

JOINs, subqueries and CTEs

Covered

Database design and normalization

Covered

Indexing and optimization

Covered

ACID, MVCC, isolation and locks

Covered

PostgreSQL and backend engineering

Covered

Production database scenarios

Covered

**Important:** Completing all 100 explanations means the question bank has been covered, not that you've demonstrated mastery of all 100 questions.

## **What I recommend next**

The best next step is a **Full SQL + DBMS Mock Interview**, conducted in interviewer mode.

It will have four rounds:

1. SQL theory — 15 questions.
2. Live SQL coding — 10 problems.
3. DBMS fundamentals — 15 questions.
4. Backend database scenarios — 5 problems.

I'll ask one question at a time, evaluate your answer, challenge incorrect assumptions, ask follow-up questions and maintain a performance scorecard.

The objective is to establish whether you can independently handle SQL and DBMS questions in an actual B.Tech CSE placement interview.