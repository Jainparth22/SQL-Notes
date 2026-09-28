# **SQL Interview Mastery — Part 4**

## **100 Frequently Asked SQL & DBMS Interview Questions**

Interview preparation

B.Tech CSE / SDE

PostgreSQL-focused

Now we'll convert everything you've studied into interview-ready answers.

The questions are grouped into four sections:

| Section | Questions | Focus                                            |
| ------- | --------- | ------------------------------------------------ |
| 1       | 1–25      | SQL fundamentals and DBMS                        |
| 2       | 26–50     | Joins, subqueries, keys and normalization        |
| 3       | 51–75     | Indexes, transactions and concurrency            |
| 4       | 76–100    | Advanced SQL, optimization and backend scenarios |

**Answering strategy:** Give a direct definition, explain the important difference, and provide an example when asked. Aim for approximately 30–40 words in your spoken interview answers.

# **Section 1 — SQL Fundamentals and DBMS**

## **A. Basic database concepts**

### **Q1. What is SQL?**

**Answer:** SQL (Structured Query Language) is a standard language used to communicate with relational databases. It supports creating database structures, inserting and modifying data, retrieving records, controlling access and managing transactions.

### **Q2. What is a DBMS?**

**Answer:** A Database Management System is software that allows users and applications to store, retrieve, update and manage data. It provides features such as data integrity, concurrency control, security, backup and recovery.

Examples: PostgreSQL, MySQL, Oracle Database and Microsoft SQL Server.

### **Q3. What is an RDBMS?**

**Answer:** An RDBMS is a database management system based on the relational model. It organizes data into tables containing rows and columns, and uses relationships, keys and constraints to maintain data integrity.

Examples: PostgreSQL, MySQL and Oracle.

### **Q4. Difference between DBMS and RDBMS?**

| DBMS                                            | RDBMS                                    |
| ----------------------------------------------- | ---------------------------------------- |
| General database management concept             | Based on the relational model            |
| May use different data models                   | Uses tables and relations                |
| Relationships depend on the system              | Supports relational constraints and keys |
| Examples include document and key-value systems | PostgreSQL, MySQL                        |

**Interview note:** An RDBMS is a type of DBMS. Don't describe all DBMS products as non-relational.

### **Q5. What is a database?**

**Answer:** A database is an organized collection of data stored electronically. It allows applications to maintain, retrieve and manipulate information efficiently while supporting requirements such as consistency, security and concurrent access.

### **Q6. What is a table?**

**Answer:** A table is a database structure that stores related data in rows and columns. Each row represents a record, while each column represents an attribute with an associated data type and constraints.

### **Q7. What is a schema?**

**Answer:** A schema is a logical namespace that organizes database objects such as tables, views, functions and sequences. In PostgreSQL, a database can contain multiple schemas, helping separate objects and manage permissions.

Example:

```
CREATE SCHEMA university;

CREATE TABLE university.students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100)
);
```

**Important PostgreSQL distinction:** A database contains schemas, and schemas contain objects. A schema is not the same as a database.

### **Q8. What are the different categories of SQL commands?**

There are five commonly taught categories:

| Category | Full form                    | Purpose             | Commands                      |
| -------- | ---------------------------- | ------------------- | ----------------------------- |
| DDL      | Data Definition Language     | Define structures   | CREATE, ALTER, DROP, TRUNCATE |
| DML      | Data Manipulation Language   | Modify data         | INSERT, UPDATE, DELETE        |
| DQL      | Data Query Language          | Retrieve data       | SELECT                        |
| DCL      | Data Control Language        | Manage privileges   | GRANT, REVOKE                 |
| TCL      | Transaction Control Language | Manage transactions | COMMIT, ROLLBACK, SAVEPOINT   |

The five-category classification is commonly used in interviews and textbooks, although SQL standards don't always present DQL as a separate category.

## **B. Important SQL command differences**

### **Q9. Difference between DELETE, TRUNCATE and DROP?**

This is one of the most frequently asked SQL interview questions.

| DELETE                             | TRUNCATE                                 | DROP                        |
| ---------------------------------- | ---------------------------------------- | --------------------------- |
| Removes selected rows              | Removes all rows                         | Removes the database object |
| Supports WHERE                     | Does not support WHERE                   | Removes the table itself    |
| DML classification                 | DDL classification in common terminology | DDL                         |
| Table structure remains            | Table structure remains                  | Table structure is removed  |
| Can fire row-level DELETE triggers | Does not fire row-level DELETE triggers  | Removes the object          |

Examples:

```
DELETE FROM employees
WHERE department_id = 10;
```

```
TRUNCATE TABLE employees;
```

```
DROP TABLE employees;
```

**PostgreSQL-specific details:**

- `TRUNCATE` can be rolled back inside a transaction.
- It takes an `ACCESS EXCLUSIVE` lock on the table.
- It can use `RESTART IDENTITY` to reset owned sequences.
- `CASCADE` may truncate referencing tables and must be used carefully.

### **Q10. Difference between WHERE and HAVING?**

**Answer:** WHERE filters individual rows before grouping, while HAVING filters groups after aggregation. WHERE is used for row-level conditions; HAVING is commonly used with aggregate conditions such as COUNT, SUM and AVG.

Example:

```
SELECT department_id, AVG(salary)
FROM employees
WHERE salary > 30000
GROUP BY department_id
HAVING AVG(salary) > 50000;
```

### **Q11. Difference between UNION and UNION ALL?**

**Answer:** UNION combines result sets and eliminates duplicate rows, whereas UNION ALL preserves duplicates. UNION generally requires additional deduplication work, so UNION ALL is preferable when duplicate removal is unnecessary.

### **Q12. Difference between CHAR and VARCHAR?**

| CHAR(n)                                  | VARCHAR(n)                                |
| ---------------------------------------- | ----------------------------------------- |
| Fixed-length character type              | Variable-length character type            |
| Space-pads values to the declared length | Stores variable-length strings            |
| Useful for genuinely fixed-width fields  | Useful for names, emails and descriptions |

In PostgreSQL, `CHAR(n)` is also called `CHARACTER(n)`. For most application text columns, `VARCHAR` or `TEXT` is more practical.

### **Q13. What is the difference between NULL and zero?**

**Answer:** NULL represents an unknown or missing value, whereas zero is a valid numeric value. SQL treats NULL using three-valued logic, so comparisons require IS NULL or IS NOT NULL rather than ordinary equality.

Example:

```
SELECT *
FROM employees
WHERE manager_id IS NULL;
```

### **Q14. What is the difference between COUNT(\*) and COUNT(column)?**

**Answer:** COUNT(\*) counts all rows, including rows containing NULL values. COUNT(column) counts only rows where that particular column is non-NULL. This distinction becomes important when using LEFT JOIN and aggregate functions.

```
SELECT
    COUNT(*) AS all_rows,
    COUNT(manager_id) AS non_null_managers
FROM employees;
```

### **Q15. What is a constraint in SQL?**

**Answer:** A constraint is a rule enforced by the database to maintain data validity and integrity. Common constraints include PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL and CHECK, which prevent invalid database states.

Example:

```
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    email VARCHAR(150) UNIQUE NOT NULL,
    age INT CHECK (age >= 18)
);
```

# **Section 2 — Keys and Relational Database Concepts**

## **C. Keys**

### **Q16. What is a primary key?**

**Answer:** A primary key uniquely identifies each row in a table. It must contain unique, non-NULL values, and a table can have only one primary-key constraint, although that constraint can contain multiple columns.

```
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

### **Q17. What is a foreign key?**

**Answer:** A foreign key establishes a relationship between tables by referencing a primary key or suitable unique key in another table. It enforces referential integrity by preventing invalid references, subject to the configured deletion and update actions.

```
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(customer_id)
);
```

### **Q18. Difference between primary key and unique key?**

| Primary key                          | Unique constraint                         |
| ------------------------------------ | ----------------------------------------- |
| Uniquely identifies a row            | Enforces uniqueness                       |
| Cannot contain NULL                  | NULL handling depends on DBMS             |
| One primary-key constraint per table | Multiple unique constraints allowed       |
| Creates a unique index in PostgreSQL | Also creates a unique index in PostgreSQL |

**PostgreSQL detail:** A normal unique constraint allows multiple NULL values by default. `UNIQUE NULLS NOT DISTINCT` treats NULL values as equal for uniqueness enforcement.

### **Q19. What is a candidate key?**

**Answer:** A candidate key is a minimal set of attributes that uniquely identifies every row in a relation. A table can have multiple candidate keys, but one is selected as the primary key.

Example: Both `student_id` and `email` might uniquely identify students.

### **Q20. What is a super key?**

**Answer:** A super key is any set of one or more attributes that uniquely identifies a row. Unlike a candidate key, it does not need to be minimal because it may contain unnecessary attributes.

Example:

- `{student_id}` can be a candidate key.
- `{student_id, name}` can be a super key, but is not minimal.

### **Q21. What is a composite key?**

**Answer:** A composite key consists of two or more columns that collectively identify a row uniquely. It is useful in relationship tables, where neither individual foreign-key column is unique.

Example:

```
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    enrollment_date DATE,
    PRIMARY KEY (student_id, course_id)
);
```

### **Q22. What is a surrogate key versus a natural key?**

**Answer:** A natural key comes from meaningful business data, such as an email or government-issued identifier. A surrogate key is an artificially generated identifier, such as a UUID or sequence-based ID, without business meaning.

| Natural key                          | Surrogate key                    |
| ------------------------------------ | -------------------------------- |
| Has business meaning                 | Has no inherent business meaning |
| Can change                           | Usually stable                   |
| May be lengthy                       | Often compact or standardized    |
| Examples: email, registration number | Examples: integer ID, UUID       |

A surrogate key doesn't eliminate the need for business uniqueness constraints.

### **Q23. What is referential integrity?**

**Answer:** Referential integrity ensures that relationships between related tables remain valid. A foreign key enforces this by requiring referenced values to exist in the parent table or be NULL when permitted.

Example: An order cannot reference a nonexistent customer if the foreign-key constraint is enforced.

# **Section 3 — Database Design and Normalization**

## **D. Normalization**

### **Q24. What is normalization?**

**Answer:** Normalization is the process of organizing relational tables to reduce data redundancy and prevent insertion, update and deletion anomalies. It uses keys and functional dependencies to structure data into appropriate relations.

### **Q25. Explain 1NF, 2NF, 3NF and BCNF.**

This is a very common DBMS interview question.

| Normal form | Main requirement                                                                         |
| ----------- | ---------------------------------------------------------------------------------------- |
| 1NF         | Attributes contain atomic values and there are no repeating groups                       |
| 2NF         | In 1NF, every non-prime attribute is fully functionally dependent on every candidate key |
| 3NF         | In 2NF, for every non-trivial FD \\(X \to A\\), X is a superkey or A is prime            |
| BCNF        | For every non-trivial FD \\(X \to Y\\), X must be a superkey                             |

### **Understand through an example**

Suppose:

`Student_Course`

| student_id | course_id | student_name | course_name | instructor  |
| ---------- | --------- | ------------ | ----------- | ----------- |
| 1          | 101       | Parth        | DBMS        | Professor A |
| 1          | 102       | Parth        | OS          | Professor B |
| 2          | 101       | Rahul        | DBMS        | Professor A |

Assume the primary key is:

```
(student_id, course_id)
```

Functional dependencies:

```
student_id → student_name

course_id → course_name, instructor
```

Problem:

`student_name` depends only on `student_id`, not the whole composite key.

Similarly, `course_name` and `instructor` depend only on `course_id`.

This violates 2NF.

Decompose:

**Students**

| student_id | student_name |
| ---------- | ------------ |
| 1          | Parth        |
| 2          | Rahul        |

**Courses**

| course_id | course_name | instructor  |
| --------- | ----------- | ----------- |
| 101       | DBMS        | Professor A |
| 102       | OS          | Professor B |

**Enrollments**

| student_id | course_id |
| ---------- | --------- |
| 1          | 101       |
| 1          | 102       |
| 2          | 101       |

Now student details and course details are stored separately.

**Interview tip:** Understand partial dependencies, transitive dependencies, candidate keys and prime attributes. Don't just memorise the names of normal forms.

# **How to practise these questions**

**Section 1 revision tracker**

## **0/4**

Q1–Q8: Database fundamentals

Q9–Q15: SQL command and constraint differences

Q16–Q23: Keys and referential integrity

Q24–Q25: Normalization and functional dependencies

Target: explain each answer in your own words without looking at the notes.

## **Interviewer challenge — five questions**

Before moving ahead, check whether you can answer these without memorising the explanations.

1. Can a table have multiple candidate keys but only one primary key?
2. Can a foreign key contain NULL?
3. Why is `UNIQUE` different from `PRIMARY KEY` in PostgreSQL?
4. Why is a composite key useful in an enrollment table?
5. Can a table be in 3NF but not BCNF?

Self-check answers

Reveal answers

## **Next: Questions 26–50**

The next installment will cover:

- INNER, LEFT, RIGHT and FULL JOIN.
- Self joins and cross joins.
- Subqueries versus correlated subqueries.
- EXISTS versus IN versus JOIN.
- CTEs versus derived tables.
- Functional dependencies and anomalies.
- Clustered versus nonclustered indexes.
- DELETE vs TRUNCATE deeper interview traps.

We'll continue building the question bank until all 100 questions are covered, then move to the final mock interview.

# **SQL Interview Mastery — Part 4**

## **Questions 26–50: JOINs, Subqueries, CTEs and Database Design**

Interview preparation

SDE / Backend

PostgreSQL

This section focuses on questions interviewers use to check whether you understand how tables relate, how SQL retrieves related data, and how database design affects correctness.

# **Section 2 — JOINs and Relationships**

## **Q26. What is an INNER JOIN?**

**Answer:** INNER JOIN returns only rows having matching values in both tables according to the join condition. Rows without a match in either table are excluded from the result.

```
SELECT
    e.emp_name,
    d.department_name
FROM employees e
INNER JOIN departments d
    ON e.department_id = d.department_id;
```

Example:

**INNER JOIN**

**Employees**

101 — Parth — Dept 10

102 — Rahul — Dept 20

103 — Amit — Dept 30

**Departments**

10 — IT

20 — HR

40 — Finance

**Result**

| Employee | Department |
| -------- | ---------- |
| Parth    | IT         |
| Rahul    | HR         |

Amit is excluded because department 30 does not match. Finance is excluded because it has no matching employee.

## **Q27. What is a LEFT JOIN?**

**Answer:** LEFT JOIN returns every row from the left table and matching rows from the right table. When no right-side match exists, the right-side columns contain NULL.

```
SELECT
    c.customer_id,
    c.customer_name,
    o.order_id
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id;
```

**Use cases:**

- Find all customers, including those without orders.
- Find all departments, including empty departments.
- Build reports where unmatched records must remain visible.

## **Q28. What is a RIGHT JOIN?**

**Answer:** RIGHT JOIN returns every row from the right table and matching rows from the left table. When no left-side match exists, the left-side columns contain NULL. It can generally be rewritten as a LEFT JOIN by reversing table order.

```
SELECT *
FROM employees e
RIGHT JOIN departments d
    ON e.department_id = d.department_id;
```

Equivalent:

```
SELECT *
FROM departments d
LEFT JOIN employees e
    ON e.department_id = d.department_id;
```

**Interview tip:** Most developers prefer LEFT JOIN because it makes the preserved table visually clear.

## **Q29. What is a FULL OUTER JOIN?**

**Answer:** FULL OUTER JOIN returns all matching and nonmatching rows from both tables. When a row has no corresponding match, columns from the missing side are populated with NULL.

```
SELECT
    e.emp_name,
    d.department_name
FROM employees e
FULL OUTER JOIN departments d
    ON e.department_id = d.department_id;
```

Use cases include reconciliation and comparing two datasets.

## **Q30. What is a CROSS JOIN?**

**Answer:** CROSS JOIN returns the Cartesian product of two tables. Every row from the first table is combined with every row from the second table. If the tables contain M and N rows, the result contains M × N rows.

```
SELECT *
FROM sizes
CROSS JOIN colors;
```

If there are four sizes and three colors:

\\[ 4\times3=12\text{ rows} \\]

Practical uses include generating combinations, calendars and test data.

## **Q31. What is a SELF JOIN?**

**Answer:** A self join joins a table with itself using different aliases. It is useful for representing hierarchical relationships, such as employees and their managers, where both entities exist in the same table.

```
SELECT
    e.emp_name AS employee,
    m.emp_name AS manager
FROM employees e
LEFT JOIN employees m
    ON e.manager_id = m.emp_id;
```

Other use cases:

- Employee-manager hierarchy.
- Comparing employees in the same department.
- Finding pairs of related records.

## **Q32. What is the difference between ON and WHERE in a JOIN?**

**Answer:** ON specifies the conditions used to match rows between tables, whereas WHERE filters rows from the joined result. In outer joins, moving a right-table condition from ON into WHERE can remove unmatched rows.

Example:

```
-- Preserves customers without completed orders
SELECT *
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
   AND o.status = 'Completed';
```

Compare:

```
-- Removes customers without completed orders
SELECT *
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.status = 'Completed';
```

## **Q33. What is the difference between JOIN and UNION?**

**Answer:** JOIN combines columns from related tables horizontally using matching conditions. UNION combines rows from compatible query results vertically. JOIN requires a relationship condition, while UNION requires compatible result-column counts and types.

**JOIN — horizontal combination**

**Employees**

101 — Parth

**Departments**

10 — IT

Result: employee and department columns in the same row.

**UNION — vertical combination**

Employee names

Parth

Rahul

Amit

Result: rows from multiple query results combined into one result set.

## **Q34. What is an anti-join?**

**Answer:** An anti-join returns rows from one table that have no matching rows in another table. It is commonly implemented using NOT EXISTS or LEFT JOIN with an IS NULL condition.

Example: Customers with no orders.

**Method 1 — NOT EXISTS**

```
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

**Method 2 — LEFT JOIN**

```
SELECT c.*
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

## **Q35. What is a semi-join?**

**Answer:** A semi-join returns rows from one table when at least one matching row exists in another table. It does not multiply the left-side rows when multiple matches exist, and is commonly expressed using EXISTS.

```
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

**Important difference:**

| Semi-join                   | Anti-join                |
| --------------------------- | ------------------------ |
| Finds matching rows         | Finds nonmatching rows   |
| Commonly uses EXISTS        | Commonly uses NOT EXISTS |
| At least one match required | No match allowed         |

# **Section 3 — Subqueries and CTEs**

## **Q36. What is a subquery?**

**Answer:** A subquery is a query nested inside another SQL statement. It can appear in SELECT, FROM, WHERE or HAVING and provides intermediate results used by the outer query.

Example: Find employees earning more than the overall average salary.

```
SELECT *
FROM employees
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
);
```

The inner query calculates the average salary. The outer query uses it as a filter.

## **Q37. What is a correlated subquery?**

**Answer:** A correlated subquery references columns from the outer query and depends on its current row. It is conceptually evaluated for each relevant outer-row context, although the optimizer may transform it into another execution strategy.

Example: Employees earning above their department's average.

```
SELECT *
FROM employees e
WHERE salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id
);
```

The inner query references `e.department_id` from the outer query.

### **Non-correlated vs correlated**

| Non-correlated                  | Correlated                              |
| ------------------------------- | --------------------------------------- |
| Independent of outer query      | References outer query                  |
| Can be evaluated independently  | Depends on outer-row values             |
| Example: overall average salary | Example: department-wise average salary |

**Important:** A correlated subquery is not necessarily slow. The optimizer may decorrelate or otherwise optimize it. Measure the actual plan.

## **Q38. Difference between IN and EXISTS?**

**Answer:** IN checks whether a value belongs to a set of returned values, while EXISTS checks whether a subquery returns at least one row. Both can express membership conditions, and the optimizer may produce similar execution plans.

IN example:

```
SELECT *
FROM customers
WHERE customer_id IN (
    SELECT customer_id
    FROM orders
);
```

EXISTS example:

```
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

### **When should you use each?**

- `IN`: Convenient when comparing a value against a set.
- `EXISTS`: Natural when checking whether a related record exists.
- `NOT EXISTS`: Often safer than `NOT IN` when the subquery may contain NULL.

There is no universal rule that EXISTS is always faster than IN. Performance depends on the query, database and execution plan.

## **Q39. What is a scalar subquery?**

**Answer:** A scalar subquery returns exactly one column and at most one row, producing a single value that can be used in expressions. In PostgreSQL, returning multiple rows in a scalar context produces a cardinality violation.

```
SELECT
    emp_name,
    salary,
    (SELECT AVG(salary) FROM employees) AS average_salary
FROM employees;
```

If the scalar subquery returns no rows, its result is NULL.

## **Q40. What is a derived table?**

**Answer:** A derived table is a subquery used in the FROM clause. It acts as an intermediate table within the outer query and must be given an alias in PostgreSQL.

```
SELECT *
FROM (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
) AS dept_stats
WHERE avg_salary > 50000;
```

Here, `dept_stats` is the derived table.

## **Q41. What is a CTE?**

**Answer:** A Common Table Expression is a named temporary result defined using WITH. It improves query readability, supports multi-step transformations and can be recursive. Its output exists only for the duration of the statement.

```
WITH dept_salary AS (
    SELECT
        department_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department_id
)
SELECT *
FROM dept_salary
WHERE avg_salary > 50000;
```

### **CTE vs subquery**

| CTE                              | Subquery                                               |
| -------------------------------- | ------------------------------------------------------ |
| Named result                     | Nested query                                           |
| Useful for multi-step logic      | Useful for localized calculations                      |
| Can be referenced multiple times | Usually written within a specific query location       |
| Supports recursion               | Ordinary subqueries do not express recursive iteration |

**PostgreSQL nuance:** A nonrecursive CTE is not necessarily materialized. PostgreSQL can inline eligible CTEs. `MATERIALIZED` and `NOT MATERIALIZED` can influence this behaviour.

## **Q42. What is a recursive CTE?**

**Answer:** A recursive CTE repeatedly executes a recursive query against previously generated rows until no new rows are produced. It is useful for hierarchical data, organizational charts, tree traversal and recursive relationships.

```
WITH RECURSIVE employee_tree AS (
    SELECT
        emp_id,
        emp_name,
        manager_id,
        0 AS depth
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    SELECT
        e.emp_id,
        e.emp_name,
        e.manager_id,
        t.depth + 1
    FROM employees e
    JOIN employee_tree t
        ON e.manager_id = t.emp_id
)
SELECT *
FROM employee_tree;
```

Two components:

1. **Anchor member:** Starts the recursion.
2. **Recursive member:** Finds the next level.

Use cycle protection for potentially cyclic data.

# **Section 4 — Functional Dependencies and Database Design**

## **Q43. What is a functional dependency?**

**Answer:** A functional dependency describes a relationship where one set of attributes uniquely determines another set of attributes. It is written as X → Y, meaning each value of X is associated with exactly one value of Y.

Example:

\\[ student\\\_id \rightarrow student\\\_name \\]

This means a student ID determines the student's name.

### **More examples**

```
employee_id → employee_name

course_id → course_name

(order_id, product_id) → quantity
```

Functional dependencies are important for normalization and identifying candidate keys.

## **Q44. What is a determinant?**

**Answer:** A determinant is the attribute or set of attributes on the left-hand side of a functional dependency. It determines the values of the attributes on the right-hand side.

Example:

\\[ A,B \rightarrow C \\]

Here, `{A, B}` is the determinant.

## **Q45. What are insertion, update and deletion anomalies?**

**Answer:** Anomalies are problems caused by redundant or poorly structured data. Insertion anomalies complicate adding records, update anomalies require modifying repeated information, and deletion anomalies may unintentionally remove valuable information.

Example: A single table stores both student and course information.

| student_id | student_name | course_id | course_name |
| ---------- | ------------ | --------- | ----------- |
| 1          | Parth        | 101       | DBMS        |
| 2          | Rahul        | 101       | DBMS        |
| 3          | Amit         | 102       | OS          |

- **Update anomaly:** Changing the DBMS course name requires updating multiple records.
- **Insertion anomaly:** A course cannot be stored independently if a student must also be entered.
- **Deletion anomaly:** Removing the last student enrolled in a course may also remove the only record containing the course information.

Normalization helps prevent these problems.

## **Q46. What is lossless decomposition?**

**Answer:** Lossless decomposition splits a relation into smaller relations while ensuring that joining them reconstructs exactly the original relation, without introducing spurious tuples or losing information.

Suppose:

```
Student(student_id, student_name, department_id)
```

We decompose into:

```
Student(student_id, student_name)
Department_Membership(student_id, department_id)
```

If the decomposition is correctly designed, joining on `student_id` reconstructs the original relation.

For a binary decomposition of relation R into R1 and R2, a useful losslessness condition is that the common attributes functionally determine all attributes of at least one decomposed relation.

## **Q47. What is dependency preservation?**

**Answer:** A decomposition is dependency-preserving when the original functional dependencies can be enforced by checking the decomposed relations individually, without requiring a join of those relations.

Why it matters:

- Makes integrity enforcement easier.
- Avoids repeatedly joining relations to validate constraints.
- Helps maintain consistency.

**Important distinction:** A decomposition can be lossless but fail to preserve all functional dependencies.

## **Q48. What is denormalization?**

**Answer:** Denormalization deliberately introduces controlled redundancy to improve read performance or simplify frequently used queries. It can reduce joins but increases storage usage and the complexity of maintaining consistent values.

Example:

Instead of joining orders with customers every time, an order table might retain a snapshot of the customer's shipping address.

This is useful because historical order addresses should often remain unchanged even if the customer's current address changes.

**Interview takeaway:** Normalization helps maintain correctness and reduce redundancy; denormalization can improve read performance when justified by workload requirements.

# **Section 5 — Indexing and Storage Differences**

## **Q49. Difference between clustered and nonclustered indexes?**

**Answer:** A clustered index determines or closely organizes the physical storage order of table records according to an index key, while a nonclustered index stores a separate search structure containing keys and references to rows. The terminology and implementation vary by database.

### **MySQL InnoDB**

- Table data is organized by the primary-key clustered index.
- Secondary indexes contain the secondary key and the primary-key value.

### **PostgreSQL**

PostgreSQL does not automatically maintain a clustered index like InnoDB.

It has:

```
CLUSTER employees
USING idx_employees_salary;
```

This physically rewrites the table in the index's order.

However, PostgreSQL does **not** continuously maintain this physical order as rows are inserted or updated.

It is not equivalent to an always-maintained clustered index.

| MySQL InnoDB                                       | PostgreSQL                                          |
| -------------------------------------------------- | --------------------------------------------------- |
| Primary key is clustered by default                | Primary key does not automatically cluster the heap |
| Secondary indexes include primary-key values       | Secondary indexes contain tuple identifiers         |
| Clustered ordering is maintained by storage engine | `CLUSTER` performs a one-time physical reordering   |

## **Q50. Why is TRUNCATE often faster than DELETE without WHERE?**

**Answer:** TRUNCATE avoids processing every row as an individual DELETE operation. It removes all table rows using a more efficient storage-level operation, although it takes a strong table lock and still performs transactional and metadata work.

Example:

```
DELETE FROM employees;
```

Compared with:

```
TRUNCATE TABLE employees;
```

### **Important interview follow-ups**

**Q. Does TRUNCATE support WHERE?**

No. It removes all rows from the specified table or tables.

**Q. Does TRUNCATE fire DELETE triggers in PostgreSQL?**

No. It fires TRUNCATE triggers, not row-level DELETE triggers.

**Q. Can TRUNCATE be rolled back in PostgreSQL?**

Yes, when executed in a transaction. PostgreSQL supports transactional TRUNCATE.

**Q. Does TRUNCATE reset sequences?**

Not by default. Use:

```
TRUNCATE TABLE employees RESTART IDENTITY;
```

**Q. What is CASCADE?**

```
TRUNCATE TABLE departments CASCADE;
```

It can also truncate tables that have foreign-key references to the specified table. Use it carefully because it may affect more tables than expected.

# **Revision tracker — Questions 26–50**

**Section 2 progress**

# **0/4**

Q26–Q35: JOINs and relationships

Q36–Q42: Subqueries and CTEs

Q43–Q48: Functional dependencies and design

Q49–Q50: Indexing and storage

# **Test your understanding**

These are common interviewer follow-up questions. Try answering before revealing the explanations.

**Six-question checkpoint**

6 questions

**Q1. Which JOIN returns every left-table row, even when no right-table match exists?**

INNER JOIN

LEFT JOIN

CROSS JOIN

SELF JOIN

**Q2. What is the purpose of a semi-join?**

Return only unmatched records

Return left-side records that have at least one match

Multiply matching records

Remove duplicate table definitions

**Q3. Which subquery references a column from the outer query?**

Scalar subquery

Non-correlated subquery

Correlated subquery

Derived table

**Q4. Can PostgreSQL always materialize every CTE?**

Yes

No, eligible CTEs can be inlined

CTEs are always stored permanently

CTEs are equivalent to tables

**Q5. What does lossless decomposition guarantee?**

Every table has an index

The original relation can be reconstructed without spurious tuples or loss of information

All dependencies are automatically preserved

There are no foreign keys

**Q6. Does PostgreSQL automatically maintain CLUSTER ordering after future inserts?**

Yes

No

Only for primary keys

Only with a unique index

Check answers

## **What's next — Questions 51–75**

The next 25 questions will focus on:

- B-Tree, Hash, GIN, GiST and BRIN indexes.
- Composite, partial, expression and covering indexes.
- ACID properties and transaction states.
- Dirty reads, non-repeatable reads and phantom reads.
- MVCC and PostgreSQL isolation levels.
- Shared locks, exclusive locks and row-level locks.
- Optimistic vs pessimistic locking.
- Deadlocks and deadlock prevention.
- WAL, checkpoints and crash recovery.

**Placement advice:** Pay special attention to SQL JOINs, `EXISTS` versus `IN`, correlated subqueries, normalization and index behaviour. These concepts often appear in both coding assessments and technical interviews.

