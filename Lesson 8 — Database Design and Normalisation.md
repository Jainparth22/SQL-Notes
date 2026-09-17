# **Lesson 8 — Database Design and Normalisation**



Important for technical interviews

DBMS + SQL + System Design



Till now, we've focused on writing SQL queries. Now we're going one level deeper: **how should we actually design the database in which these queries operate?**

A good developer doesn't just know how to retrieve data. They understand how to store it correctly, avoid inconsistencies, maintain relationships and design schemas that can scale.

This lesson connects SQL with DBMS interview concepts.

By the end, you should be able to:

- Identify primary, foreign, candidate and composite keys.
- Understand functional dependencies.
- Explain database anomalies.
- Normalise a database through 1NF, 2NF, 3NF and BCNF.
- Design one-to-one, one-to-many and many-to-many relationships.
- Understand when denormalisation is appropriate.
- Design relational schemas for real-world applications.

We'll use PostgreSQL-compatible examples.

# **Part 1 — Database design fundamentals**

## **1. What is database design?**

Imagine you're building a university management system.

You need to store:

- Student details.
- Course details.
- Faculty details.
- Student enrolments.
- Examination results.
- Course fees.

One approach is to put everything into a single massive table.

**student_records**

| student_id | student_name | course_id | course_name | faculty_name |
| ---------- | ------------ | --------- | ----------- | ------------ |
| 1          | Aman         | 101       | DBMS        | Sharma       |
| 1          | Aman         | 102       | OS          | Verma        |
| 2          | Rahul        | 101       | DBMS        | Sharma       |
| 3          | Priya        | 103       | CN          | Gupta        |

This works for a tiny dataset, but as the database grows, problems emerge:

- Student information is repeated.
- Course names are repeated.
- Faculty names are repeated.
- Updating information requires changing multiple records.
- Inserting a course without enrolling a student becomes difficult.
- Deleting the last enrolment might accidentally remove other information.

These are database anomalies.

Database design and normalisation help solve these problems.

## **2. Important database keys**

Keys are attributes used to identify records and establish relationships between tables.

Consider:

**students**

| student_id | name  | email            |
| ---------- | ----- | ---------------- |
| 1          | Aman  | aman\@gmail.com  |
| 2          | Rahul | rahul\@gmail.com |
| 3          | Priya | priya\@gmail.com |

### **A. Super Key**

A super key is any set of one or more attributes that uniquely identifies a row.

For the students table, possible super keys include:

- `{student_id}`
- `{email}`
- `{student_id, name}`
- `{student_id, email}`

Assuming student IDs and emails are unique.

A super key may contain unnecessary attributes.

### **B. Candidate Key**

A candidate key is a minimal super key.

Minimal means that removing any attribute makes it unable to uniquely identify a row.

For the students table:

- `student_id`
- `email`

Both can be candidate keys, assuming both are unique and non-NULL.

`{student_id, email}` is not a candidate key because `student_id` alone already uniquely identifies the row.

### **C. Primary Key**

The primary key is the candidate key selected as the main identifier for the table.

```
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(255) UNIQUE
);
```

Here:

- `student_id` is the primary key.
- `email` is an alternate candidate key enforced using UNIQUE.

Primary key properties:

1. Must be unique.
2. Cannot contain NULL.
3. A table has only one primary key constraint.
4. It may consist of multiple columns.

### **D. Alternate Key**

A candidate key that has not been chosen as the primary key.

Here, email is the alternate key.

### **E. Foreign Key**

A foreign key establishes a relationship between two tables.

```
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100)
);

CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id)
        REFERENCES departments(department_id)
);
```

The `department_id` in employees references the primary key in departments.

A foreign key ensures referential integrity: a non-NULL foreign key value must reference an existing parent row, subject to the constraint definition.

### **F. Composite Key**

A composite key consists of two or more columns.

Consider a student enrolling in multiple courses.

**enrollments**

| student_id | course_id | enrollment_date |
| ---------- | --------- | --------------- |
| 1          | 101       | 2026-01-10      |
| 1          | 102       | 2026-01-11      |
| 2          | 101       | 2026-01-12      |
| 2          | 103       | 2026-01-14      |

Neither `student_id` nor `course_id` alone uniquely identifies an enrolment.

But their combination does.

```
CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    enrollment_date DATE,

    PRIMARY KEY (student_id, course_id),

    FOREIGN KEY (student_id)
        REFERENCES students(student_id),

    FOREIGN KEY (course_id)
        REFERENCES courses(course_id)
);
```

The primary key is:

\\[ (student\\\_id, course\\\_id) \\]

**Keys — interview revision**

| Key           | Meaning                               |
| ------------- | ------------------------------------- |
| Super Key     | Uniquely identifies a row             |
| Candidate Key | Minimal super key                     |
| Primary Key   | Selected candidate key                |
| Alternate Key | Unselected candidate key              |
| Foreign Key   | References a key in another table     |
| Composite Key | Key consisting of multiple attributes |

### **Common interview question: Primary key vs UNIQUE**

| Primary Key                          | UNIQUE                              |
| ------------------------------------ | ----------------------------------- |
| Identifies a row                     | Enforces uniqueness                 |
| Cannot contain NULL                  | NULL behaviour depends on database  |
| One primary key constraint per table | Multiple UNIQUE constraints allowed |
| Can consist of multiple columns      | Can consist of multiple columns     |

In PostgreSQL, a normal UNIQUE constraint permits multiple NULL values by default because NULLs are treated as distinct for uniqueness. PostgreSQL also supports `NULLS NOT DISTINCT` when a different behaviour is required.

# **Part 2 — Functional Dependencies**

Fundamental to normalisation

## **3. What is a Functional Dependency?**

A functional dependency describes a relationship between attributes.

Suppose:

| student_id | student_name | department |
| ---------- | ------------ | ---------- |
| 1          | Aman         | CSE        |
| 2          | Rahul        | ECE        |
| 3          | Priya        | CSE        |

Every student ID identifies exactly one student name and department.

We express this as:

\\[ student\\\_id \rightarrow student\\\_name \\]

\\[ student\\\_id \rightarrow department \\]

The symbol \\(\rightarrow\\) means "functionally determines".

In general:

\\[ X \rightarrow Y \\]

means that whenever two rows have the same value of X, they must also have the same value of Y.

Here:

- X is the determinant.
- Y is the dependent attribute.

### **Example**

Consider:

| employee_id | employee_name | department_id |
| ----------- | ------------- | ------------- |
| 1           | Aman          | 101           |
| 2           | Rahul         | 102           |
| 3           | Priya         | 101           |

Functional dependencies:

\\[ employee\\\_id \rightarrow employee\\\_name \\]

\\[ employee\\\_id \rightarrow department\\\_id \\]

But generally:

\\[ department\\\_id \not\rightarrow employee\\\_name \\]

Because multiple employees may belong to the same department.

### **Why are functional dependencies important?**

They help us identify:

- Candidate keys.
- Redundant data.
- Partial dependencies.
- Transitive dependencies.
- Whether a relation satisfies 2NF, 3NF or BCNF.

## **4. Types of Functional Dependencies**

### **A. Trivial Functional Dependency**

A dependency is trivial when the right-side attributes are already contained in the left-side attributes.

Example:

\\[ (A,B) \rightarrow A \\]

This is always true.

### **B. Non-trivial Functional Dependency**

A dependency is non-trivial when the right-side attributes are not contained in the left-side attributes.

Example:

\\[ A \rightarrow B \\]

where A and B are distinct attributes.

### **C. Full Functional Dependency**

An attribute is fully functionally dependent on a composite key when the entire composite key is required to determine it.

Example:

\\[ (student\\\_id, course\\\_id) \rightarrow grade \\]

Neither student ID alone nor course ID alone determines an individual student's grade in that course.

### **D. Partial Dependency**

An attribute depends on only part of a composite candidate key.

Suppose:

\\[ (student\\\_id, course\\\_id) \rightarrow student\\\_name \\]

but:

\\[ student\\\_id \rightarrow student\\\_name \\]

Then `student_name` depends on only part of the composite key.

This is called partial dependency.

Partial dependencies are important when understanding 2NF.

### **E. Transitive Dependency**

A non-key attribute depends on another non-key attribute through a key.

Consider:

\\[ student\\\_id \rightarrow department\\\_id \\]

\\[ department\\\_id \rightarrow department\\\_name \\]

Therefore:

\\[ student\\\_id \rightarrow department\\\_name \\]

through department ID.

This is a transitive dependency.

Transitive dependencies are important when understanding 3NF.

# **Part 3 — Database Normalisation**

Core DBMS interview topic

## **5. What is Normalisation?**

Normalisation is the process of organising data into properly structured relational tables to reduce redundancy and prevent data anomalies.

The main normal forms you must know for interviews are:

1. 1NF — First Normal Form.
2. 2NF — Second Normal Form.
3. 3NF — Third Normal Form.
4. BCNF — Boyce-Codd Normal Form.

Understand one important idea:

**Each normal form introduces additional conditions that the database schema must satisfy.**

## **Normalisation hierarchy**

**1NF**

Atomic values; no repeating groups

**2NF**

1NF + no partial dependency of non-prime attributes

**3NF**

2NF + no disallowed transitive dependency

**BCNF**

Every non-trivial determinant must be a superkey

Let's understand why each exists using an actual example.

## **6. First Normal Form (1NF)**

A relation is in First Normal Form when:

- Each cell contains an atomic value relative to the intended attribute.
- There are no repeating groups of columns.
- Rows can be distinguished through a key.

Consider this table:

**student_courses**

| student_id | name  | courses        |
| ---------- | ----- | -------------- |
| 1          | Aman  | DBMS, OS, CN   |
| 2          | Rahul | DBMS, Java     |
| 3          | Priya | OS, Python, CN |

The courses column contains multiple values in a single cell.

This structure is not in the intended 1NF representation.

### **Convert it to 1NF**

| student_id | name  | course |
| ---------- | ----- | ------ |
| 1          | Aman  | DBMS   |
| 1          | Aman  | OS     |
| 1          | Aman  | CN     |
| 2          | Rahul | DBMS   |
| 2          | Rahul | Java   |
| 3          | Priya | OS     |
| 3          | Priya | Python |
| 3          | Priya | CN     |

Now each row represents one student-course relationship.

A suitable primary key would be:

\\[ (student\\\_id, course) \\]

**Interview question: What is atomicity in 1NF?**

Atomicity means a column value represents one indivisible value according to the intended data model. For example, a course enrolment should be represented as a separate row rather than a comma-separated collection inside one cell.

Important nuance: What counts as atomic depends on the application's data model. 1NF doesn't mean that SQL cannot store JSON, arrays or structured values; it concerns how attributes are represented in a relational design.

## **7. Second Normal Form (2NF)**

Very important

A relation is in 2NF if:

1. It is already in 1NF.
2. Every non-prime attribute is fully functionally dependent on every candidate key.

In simpler interview language:

**There must be no partial dependency of a non-key attribute on a candidate key.**

Let's understand this properly.

Consider:

**student_enrollment**

| student_id | course_id | student_name | course_name | grade |
| ---------- | --------- | ------------ | ----------- | ----- |
| 1          | 101       | Aman         | DBMS        | A     |
| 1          | 102       | Aman         | OS          | B     |
| 2          | 101       | Rahul        | DBMS        | A     |
| 3          | 103       | Priya        | CN          | A     |

Assume:

- One student has one name.
- One course has one name.
- A student can enrol in multiple courses.
- Each student has one grade per course.

The candidate key is:

\\[ (student\\\_id, course\\\_id) \\]

Functional dependencies:

\\[ student\\\_id \rightarrow student\\\_name \\]

\\[ course\\\_id \rightarrow course\\\_name \\]

\\[ (student\\\_id, course\\\_id) \rightarrow grade \\]

Look carefully.

`student_name` depends only on `student_id`, not on the complete composite key.

Similarly, `course_name` depends only on `course_id`.

These are partial dependencies.

**Visualise partial dependency**

**Composite candidate key**

# **(student_id, course_id)**

**student_id**

**student_name**

Partial

**course_id**

**course_name**

Partial

**(student_id, course_id)**

**grade**

Full dependency

### **Convert to 2NF**

Split the original table into three tables.

**students**

| student_id (PK) | student_name |
| --------------- | ------------ |
| 1               | Aman         |
| 2               | Rahul        |
| 3               | Priya        |

**courses**

| course_id (PK) | course_name |
| -------------- | ----------- |
| 101            | DBMS        |
| 102            | OS          |
| 103            | CN          |

**enrollments**

| student_id (PK, FK) | course_id (PK, FK) | grade |
| ------------------- | ------------------ | ----- |
| 1                   | 101                | A     |
| 1                   | 102                | B     |
| 2                   | 101                | A     |
| 3                   | 103                | A     |

Now:

- Student name depends on student ID.
- Course name depends on course ID.
- Grade depends on the complete composite key.

The partial dependencies have been eliminated.

**Important interview trap:** 2NF is automatically satisfied by a 1NF relation whose every candidate key consists of only one attribute, because there cannot be a partial dependency on a single-attribute key.

## **8. Third Normal Form (3NF)**

Extremely important

A relation is in 3NF if:

1. It is already in 2NF.
2. It has no disallowed transitive dependencies.

Let's return to our university database.

Suppose we have:

**courses**

| course_id | course_name | instructor_id | instructor_name |
| --------- | ----------- | ------------- | --------------- |
| 101       | DBMS        | 501           | Sharma          |
| 102       | OS          | 502           | Verma           |
| 103       | CN          | 503           | Gupta           |
| 104       | Java        | 501           | Sharma          |

Functional dependencies:

\\[ course\\\_id \rightarrow instructor\\\_id \\]

\\[ instructor\\\_id \rightarrow instructor\\\_name \\]

Therefore:

\\[ course\\\_id \rightarrow instructor\\\_name \\]

This is a transitive dependency.

The course ID determines the instructor ID, which determines the instructor name.

The problem is that instructor names are repeated whenever the same instructor teaches multiple courses.

### **Convert to 3NF**

Separate instructors into their own table.

**courses**

| course_id (PK) | course_name | instructor_id (FK) |
| -------------- | ----------- | ------------------ |
| 101            | DBMS        | 501                |
| 102            | OS          | 502                |
| 103            | CN          | 503                |
| 104            | Java        | 501                |

**instructors**

| instructor_id (PK) | instructor_name |
| ------------------ | --------------- |
| 501                | Sharma          |
| 502                | Verma           |
| 503                | Gupta           |

Now instructor information is stored separately.

The resulting schema contains:

- Students
- Courses
- Instructors
- Enrollments

The relationships are maintained using foreign keys.

### **The formal definition of 3NF**

For every non-trivial functional dependency:

\\[ X \rightarrow A \\]

at least one of these must be true:

1. X is a superkey.
2. A is a prime attribute (an attribute belonging to at least one candidate key).

This is the rigorous definition to use in advanced DBMS interviews.

**Interview answer:**

3NF eliminates problematic transitive dependencies of non-key attributes on candidate keys. Formally, every non-trivial functional dependency must have a superkey determinant or a prime attribute on its right-hand side.

## **9. BCNF — Boyce-Codd Normal Form**

Advanced DBMS interview topic

BCNF is a stricter form of 3NF.

A relation is in BCNF if:

**For every non-trivial functional dependency X → Y, X must be a superkey.**

Compare:

| 3NF                                                                                       | BCNF                                          |
| ----------------------------------------------------------------------------------------- | --------------------------------------------- |
| Allows a dependency with a non-superkey determinant when the dependent attribute is prime | Requires every determinant to be a superkey   |
| Less restrictive                                                                          | More restrictive                              |
| Does not eliminate every possible redundancy                                              | Eliminates certain anomalies remaining in 3NF |

### **Understand using an example**

Suppose a university maintains the following relation:

**student_course_instructor**

| student | course | instructor |
| ------- | ------ | ---------- |
| Aman    | DBMS   | Sharma     |
| Rahul   | DBMS   | Sharma     |
| Priya   | OS     | Verma      |
| Aman    | OS     | Verma      |

Assume these business rules:

1. A student taking a course has exactly one instructor.
2. Each instructor teaches exactly one course.
3. Different instructors may teach the same course.

Functional dependencies:

\\[ (student, course) \rightarrow instructor \\]

\\[ instructor \rightarrow course \\]

Candidate keys are:

\\[ (student, course) \\]

and

\\[ (student, instructor) \\]

Why is `(student, instructor)` a candidate key?

Because instructor determines course, so together they determine all three attributes.

Now examine:

\\[ instructor \rightarrow course \\]

`instructor` alone is not a superkey because one instructor can teach many students.

Therefore, this dependency violates BCNF.

However, `course` is a prime attribute because it belongs to candidate key `(student, course)`.

So the relation satisfies the relevant 3NF condition but violates BCNF.

### **Decompose into BCNF**

**instructor_courses**

| instructor (PK) | course |
| --------------- | ------ |
| Sharma          | DBMS   |
| Verma           | OS     |

**student_instructors**

| student | instructor |
| ------- | ---------- |
| Aman    | Sharma     |
| Rahul   | Sharma     |
| Priya   | Verma      |
| Aman    | Verma      |

Now the determinant in each relation is a key.

This decomposition removes the BCNF violation.

**Remember the most important distinction**

# **BCNF ⊂ 3NF**

Every BCNF relation is in 3NF, but not every 3NF relation is in BCNF.

Advanced point: BCNF decomposition can sometimes fail to preserve all original functional dependencies directly. In database design, we also care about lossless decomposition and dependency preservation.

# **Part 4 — Database Anomalies**

## **10. What are database anomalies?**

Anomalies are undesirable problems caused by poorly structured or redundant data.

There are three major types.

### **A. Insertion anomaly**

Consider:

| student_id | student_name | course_id | course_name |
| ---------- | ------------ | --------- | ----------- |
| 1          | Aman         | 101       | DBMS        |
| 2          | Rahul        | 102       | OS          |

You want to add a new course called Computer Networks.

But nobody has enrolled in it.

If your table requires a student ID and enrolment record, you may not be able to insert the course independently without using NULLs or artificial data.

This is an insertion anomaly.

### **B. Update anomaly**

Suppose instructor Sharma changes their name to Sharma Sir.

If their name appears in 100 course or student records, you must update every occurrence.

Missing even one creates inconsistent data.

This is an update anomaly.

### **C. Deletion anomaly**

Suppose the only student enrolled in a particular course withdraws.

Deleting the student's enrolment might also remove the only stored information about that course.

This is a deletion anomaly.

**Three database anomalies**

**Insertion anomaly**

Cannot add some information independently.

**Update anomaly**

Same information must be updated in multiple locations.

**Deletion anomaly**

Deleting one record unintentionally removes other information.

Normalisation helps prevent these problems.

# **Part 5 — Relationships in Relational Databases**

## **11. One-to-One (1:1)**

One record in table A is associated with at most one record in table B, and vice versa.

Example:

- One person has one passport.
- One passport belongs to one person.

Tables:

**persons**

| person_id | name  |
| --------- | ----- |
| 1         | Aman  |
| 2         | Rahul |

**passports**

| passport_id | person_id |
| ----------- | --------- |
| 101         | 1         |
| 102         | 2         |

Implementation:

```
CREATE TABLE persons (
    person_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE passports (
    passport_id INT PRIMARY KEY,
    person_id INT UNIQUE REFERENCES persons(person_id)
);
```

Why UNIQUE?

Because without UNIQUE, multiple passport records could reference the same person.

A one-to-one relationship is generally implemented using a foreign key with a UNIQUE constraint. Depending on whether the relationship is mandatory or optional, you may also use NOT NULL.

## **12. One-to-Many (1:N)**

One record in table A can be associated with multiple records in table B.

But each record in B belongs to at most one record in A.

Example:

- One department has many employees.
- Each employee belongs to one department.

**One-to-many relationship**

**Department**

# **1**

**Employees**

# **N**

The foreign key goes on the many side.

Implementation:

```
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100)
);

CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT REFERENCES departments(department_id)
);
```

The foreign key goes into the `employees` table.

This is a very important database design principle.

## **13. Many-to-Many (M:N)**

One record in A can be connected to multiple records in B, and one record in B can be connected to multiple records in A.

Example:

- One student can enrol in many courses.
- One course can have many students.

You cannot normally represent this relationship using one foreign key in either table without introducing repeating data.

The solution is a junction table (also called a bridge or associative table).

**Many-to-many relationship**

**Students**

1, 2, 3...

**Courses**

101, 102, 103...

Implemented using Enrollments between them

Schema:

**students**

```
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100)
);
```

**courses**

```
CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100)
);
```

**enrollments**

```
CREATE TABLE enrollments (
    student_id INT REFERENCES students(student_id),
    course_id INT REFERENCES courses(course_id),
    enrollment_date DATE,

    PRIMARY KEY (student_id, course_id)
);
```

The junction table contains both foreign keys.

It can also contain attributes specific to the relationship, such as:

- Enrolment date.
- Grade.
- Attendance percentage.
- Payment status.

### **Query example**

Find all courses enrolled by Aman:

```
SELECT
    s.student_name,
    c.course_name
FROM students s
JOIN enrollments e
    ON s.student_id = e.student_id
JOIN courses c
    ON e.course_id = c.course_id
WHERE s.student_id = 1;
```

This connects your understanding of database design directly to JOINs.

# **Part 6 — Referential Integrity and Constraints**

## **14. What is referential integrity?**

Referential integrity ensures that relationships between tables remain valid.

For example, an employee should not refer to a department that does not exist when the foreign key is enforced.

```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,

    FOREIGN KEY (department_id)
        REFERENCES departments(department_id)
);
```

Attempting to insert:

```
INSERT INTO employees
VALUES (10, 'Aman', 999);
```

will fail if department 999 does not exist.

### **Foreign key delete behaviours**

Consider what happens when a department is deleted.

| Action      | Behaviour                                                    |
| ----------- | ------------------------------------------------------------ |
| NO ACTION   | Rejects a deletion that violates the constraint when checked |
| RESTRICT    | Rejects a referenced-row deletion                            |
| CASCADE     | Deletes dependent rows                                       |
| SET NULL    | Sets referencing foreign keys to NULL                        |
| SET DEFAULT | Sets foreign keys to their default value                     |

Example:

```
FOREIGN KEY (department_id)
REFERENCES departments(department_id)
ON DELETE SET NULL
```

If a department is deleted, affected employees have their department ID changed to NULL.

**Interview warning:** CASCADE can delete many dependent records. Use it only when that behaviour is intended.

## **15. Other SQL constraints**

| Constraint  | Purpose                      |
| ----------- | ---------------------------- |
| NOT NULL    | Prevents missing values      |
| UNIQUE      | Prevents duplicate values    |
| PRIMARY KEY | Uniquely identifies records  |
| FOREIGN KEY | Enforces relationships       |
| CHECK       | Enforces a logical condition |
| DEFAULT     | Supplies a default value     |

Example:

```
CREATE TABLE employees (
    emp_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    salary NUMERIC(10, 2) CHECK (salary >= 0),
    status VARCHAR(20) DEFAULT 'Active',
    email VARCHAR(255) UNIQUE
);
```

Here:

- Employee name is mandatory.
- Salary cannot be negative.
- Status defaults to Active.
- Email must be unique under the database's UNIQUE semantics.

# **Part 7 — Denormalisation**

## **16. What is denormalisation?**

Denormalisation is the deliberate introduction of some redundancy or precomputed data to improve read performance or simplify queries.

You might ask:

"If normalisation reduces redundancy, why would anyone introduce redundancy?"

Because highly normalised databases can require multiple JOINs to answer common queries.

Consider an e-commerce system.

Normalised tables:

- customers
- orders
- order_items
- products

To calculate total order information, you might need several joins and aggregations.

In some systems, you may maintain a precomputed order total or cached summary.

Example:

**orders**

| order_id | customer_id | total_amount |
| -------- | ----------- | ------------ |
| 101      | 1           | 1500         |
| 102      | 2           | 2500         |

The `total_amount` might be calculated from order items but stored as a maintained summary for faster reading.

### **Normalisation vs Denormalisation**

| Normalisation                      | Denormalisation                                     |
| ---------------------------------- | --------------------------------------------------- |
| Reduces redundancy                 | Introduces controlled redundancy                    |
| Helps prevent anomalies            | Can simplify and speed up reads                     |
| Encourages consistent updates      | Requires careful synchronisation                    |
| Often uses more joins              | May reduce joins                                    |
| Useful for transactional modelling | Useful for carefully optimised read-heavy workloads |

**Important:** Denormalisation is not automatically faster. It trades simpler or faster reads for additional storage and update complexity. You should measure the real workload before introducing it.

# **Part 8 — Design a Real-World Database**

## **17. E-commerce database design: Interview exercise**

Imagine an interviewer asks:

**Design a relational database for an e-commerce application like Amazon.**

You need to identify entities, relationships, primary keys and foreign keys.

Start by identifying the entities:

1. Customers
2. Products
3. Orders
4. Order items
5. Payments

### **A. Identify the relationships**

| Relationship          | Cardinality                                      |
| --------------------- | ------------------------------------------------ |
| Customer → Orders     | One-to-many                                      |
| Order → Order Items   | One-to-many                                      |
| Product → Order Items | One-to-many                                      |
| Order → Product       | Many-to-many through Order Items                 |
| Order → Payments      | One-to-many, when partial payments are supported |

### **B. Design the schema**

**Customers**

```
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL
);
```

**Products**

```
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    price NUMERIC(10, 2) NOT NULL
        CHECK (price >= 0)
);
```

**Orders**

```
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(30),

    FOREIGN KEY (customer_id)
        REFERENCES customers(customer_id)
);
```

**Order Items**

```
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL,

    PRIMARY KEY (order_id, product_id),

    FOREIGN KEY (order_id)
        REFERENCES orders(order_id),

    FOREIGN KEY (product_id)
        REFERENCES products(product_id)
);
```

Notice `unit_price`.

Why are we storing price in both products and order_items?

Because product prices can change.

Suppose a customer purchases a product for ₹1,000. Next month, the price becomes ₹1,500.

The historical order should still reflect ₹1,000.

This is intentional historical data preservation, not necessarily a normalisation mistake.

**Payments**

```
CREATE TABLE payments (
    payment_id INT PRIMARY KEY,
    order_id INT NOT NULL,
    amount NUMERIC(10, 2) NOT NULL,
    payment_status VARCHAR(30),
    payment_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    FOREIGN KEY (order_id)
        REFERENCES orders(order_id)
);
```

### **C. Visualise the database relationships**

## **E-commerce relational schema**

**CUSTOMERS**

customer_id (PK)

1 customer → many orders

**ORDERS**

order_id (PK), customer_id (FK)

1:N

**ORDER_ITEMS**

order_id (PK, FK)

product_id (PK, FK)

1:N

**PAYMENTS**

payment_id (PK)

order_id (FK)

Many order items can reference the same product

**PRODUCTS**

product_id (PK)

This design illustrates several concepts from this lesson:

- Primary and foreign keys.
- Composite primary keys.
- One-to-many relationships.
- Many-to-many relationships.
- Junction tables.
- Historical price preservation.
- Referential integrity.

**Interview tip:** Don't immediately start writing CREATE TABLE statements. First identify entities, relationships, cardinality, keys and business constraints. Then translate the design into SQL.

# **Part 9 — Normalisation Revision**

## **18. The four normal forms**

| Normal form | Main requirement                                                   | Prevents                               |
| ----------- | ------------------------------------------------------------------ | -------------------------------------- |
| 1NF         | Atomic values; no repeating groups                                 | Repeating groups                       |
| 2NF         | No partial dependency of non-prime attributes                      | Partial dependency                     |
| 3NF         | For each non-trivial FD, determinant is a superkey or RHS is prime | Disallowed transitive dependencies     |
| BCNF        | Every non-trivial determinant must be a superkey                   | Remaining determinant-based redundancy |

### **Quick memory technique**

- 1NF: Atomic.
- 2NF: No partial dependency.
- 3NF: No disallowed transitive dependency.
- BCNF: Every determinant is a superkey.

### **Frequently asked interview questions**

**Q1. Does every 2NF table satisfy 1NF?**

Yes. 2NF builds upon 1NF.

**Q2. Can a table be in 3NF but not BCNF?**

Yes. A 3NF relation can contain a non-trivial functional dependency whose determinant is not a superkey, provided the dependent attribute is prime.

**Q3. Does BCNF guarantee that all functional dependencies are preserved after decomposition?**

No. A BCNF decomposition can be lossless but may fail to preserve all functional dependencies.

**Q4. What is lossless decomposition?**

A decomposition is lossless when joining the decomposed relations reconstructs exactly the original relation, without introducing spurious tuples.

**Q5. What is dependency preservation?**

A decomposition is dependency-preserving when the original functional dependencies can be enforced by checking the decomposed relations individually, without requiring joins.

**Q6. Is a composite primary key mandatory for 2NF violations?**

Partial dependency problems arise with composite candidate keys. Remember that 2NF is defined against all candidate keys, not just the chosen primary key.

**Q7. Is normalisation always better than denormalisation?**

No. Normalisation improves integrity and reduces redundancy, while controlled denormalisation can improve read performance for specific workloads.

# **Part 10 — Interactive DBMS Assessment**

Test your understanding.

**Database design assessment**

8 questions

**Q1. Which normal form requires atomic values and no repeating groups?**

1NF

2NF

3NF

BCNF

**Q2. A non-prime attribute depends on only part of a composite candidate key. Which normal form is violated?**

1NF

2NF

3NF

BCNF only

**Q3. Which normal form removes disallowed transitive dependencies?**

1NF

2NF

3NF

None

**Q4. What is the BCNF requirement?**

All columns must be numeric

Every determinant of a non-trivial FD must be a superkey

Every table must have exactly one candidate key

Every attribute must be a foreign key

**Q5. What is the best representation of a many-to-many relationship?**

Only one foreign key in the first table

Only one foreign key in the second table

A junction table containing foreign keys

A single comma-separated column

**Q6. Which anomaly occurs when updating repeated information in multiple rows?**

Insertion anomaly

Update anomaly

Deletion anomaly

Join anomaly

**Q7. Where is the foreign key generally placed for a one-to-many relationship?**

On the one side

On the many side

In an unrelated table

Only in the primary key

**Q8. What does lossless decomposition mean?**

No data is ever deleted

Joining decomposed tables reconstructs the original relation without spurious tuples

All tables have only one column

All functional dependencies disappear

Submit assessment

# **Part 11 — Practical Assignment**

Complete these problems without looking at the previous explanations.

| #  | Problem                                                           | Difficulty |
| -- | ----------------------------------------------------------------- | ---------- |
| 1  | Identify primary and foreign keys for a university database.      | Easy       |
| 2  | Identify candidate keys and superkeys from a given relation.      | Medium     |
| 3  | Identify functional dependencies in an employee database.         | Medium     |
| 4  | Convert a repeating-group table into 1NF.                         | Easy       |
| 5  | Identify partial dependencies and convert a relation into 2NF.    | Medium     |
| 6  | Identify transitive dependencies and convert a relation into 3NF. | Medium     |
| 7  | Determine whether a given relation satisfies BCNF.                | Hard       |
| 8  | Explain insertion, update and deletion anomalies using examples.  | Easy       |
| 9  | Design an M:N relationship between students and courses.          | Medium     |
| 10 | Design an e-commerce database with five related entities.         | Hard       |
| 11 | Explain lossless decomposition and dependency preservation.       | Hard       |
| 12 | Explain when denormalisation is appropriate.                      | Medium     |

For questions 2, 3 and 7, practise with functional dependencies rather than merely memorising definitions.

# **Part 12 — Course Progress**

**SQL + DBMS Mastery**

8 / 10 lessons

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

**09. Indexes, Transactions and Query Optimisation**

Next

10\. Company OAs and Mock Interviews

# **Next Lesson — Lesson 9: Indexes, Transactions and Query Optimisation**

Extremely important for SDE interviews

This lesson connects SQL with database internals and performance engineering.

**Section A — Indexing**

1. What is an index?
2. Why databases use indexes.
3. Primary index vs secondary index.
4. Clustered vs non-clustered indexes.
5. B-Tree and B+ Tree.
6. Composite indexes.
7. Unique and partial indexes.
8. Covering indexes and index-only scans.
9. When indexes hurt performance.

**Section B — Query optimisation**

1. Query execution plans.
2. EXPLAIN and EXPLAIN ANALYZE.
3. Sequential scan vs index scan.
4. Index selectivity.
5. Sargable queries.
6. Optimising JOINs.
7. N+1 query problem.
8. Avoiding unnecessary queries and data retrieval.

**Section C — Transactions**

1. What is a transaction?
2. ACID properties.
3. COMMIT and ROLLBACK.
4. Transaction isolation levels.
5. Dirty reads, non-repeatable reads and phantom reads.
6. Locks and deadlocks.
7. Optimistic vs pessimistic locking.
8. MVCC (Multi-Version Concurrency Control).

**Section D — Interview applications**

1. How banking transactions work.
2. How databases prevent inconsistent updates.
3. How indexes improve query performance.
4. How to optimise a slow SQL query.
5. How to handle concurrent transactions.

**My recommendation:** Give Lesson 9 special attention. Query writing gets you through many OAs, but indexing, transactions, isolation, locking and query execution are what help you answer deeper database and backend engineering interview questions.