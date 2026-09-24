# SQL + DBMS Mastery Notes

Structured notes for learning SQL from scratch and preparing for SQL interviews and online assessments (OAs). They also cover the database theory and backend topics that SDE interviews ask about.

All examples use **PostgreSQL**. Dialect differences from MySQL (LeetCode's default) are called out where they matter.

## Who this is for

- Students preparing for placements and SDE / data engineer interviews
- Anyone who wants to learn SQL by solving problems instead of memorising syntax

## How to use these notes

1. Work through the lessons in order. Each builds on the previous one.
2. Write the practice queries yourself before reading the solutions.
3. Run the examples in PostgreSQL (or any online SQL editor) to see the output.
4. Finish with Lessons 10 and 11 for mock assessments and interview drills.

## Contents

| Lesson | Topics |
| --- | --- |
| [Lesson 1 — SQL fundamentals](Lesson%201%20%E2%80%94%20SQL%20fundamentals.md) | SELECT, WHERE, ORDER BY, LIMIT, NULL, CASE, logical execution order |
| [Lesson 2 — Aggregate Functions, GROUP BY and HAVING](Lesson%202%20%E2%80%94%20Aggregate%20Functions%2C%20GROUP%20BY%20and%20HAVING.md) | COUNT/SUM/AVG, GROUP BY, HAVING, conditional aggregation, duplicate detection |
| [Lesson 3 — SQL JOINs](Lesson%203%20%E2%80%94%20SQL%20JOINs.md) | INNER/LEFT/RIGHT/FULL/CROSS/SELF joins, ON vs WHERE, join multiplication |
| [Lesson 4 — SQL Subqueries and Nested Queries](Lesson%204%20%E2%80%94%20SQL%20Subqueries%20and%20Nested%20Queries.md) | Scalar and correlated subqueries, IN/EXISTS/ANY/ALL, Nth-highest salary |
| [Lesson 5 — SQL CTEs (Common Table Expressions)](Lesson%205%20%E2%80%94%20SQL%20CTEs%20%28Common%20Table%20Expressions%29.md) | WITH clause, multiple CTEs, recursive CTEs and hierarchies |
| [Lesson 6 — SQL Window Functions](Lesson%206%20%E2%80%94%20SQL%20Window%20Functions.md) | ROW_NUMBER/RANK/DENSE_RANK, LAG/LEAD, running totals, frames, gaps basics |
| [Lesson 7 — Advanced SQL Functions Strings, Dates, NULL Handling and Conditional Logic](Lesson%207%20%E2%80%94%20Advanced%20SQL%20Functions%20Strings%2C%20Dates%2C%20NULL%20Handling%20and%20Conditional%20Logic.md) | String, date and NULL functions, conditional aggregation, type casting |
| [Lesson 8 — Database Design and Normalisation](Lesson%208%20%E2%80%94%20Database%20Design%20and%20Normalisation.md) | Keys, functional dependencies, 1NF to BCNF, relationships, schema design |
| [Lesson 9 — Complete Advanced DBMS-Indexing, Query Optimisation, Transactions and Backend Engineering-Part-2](Lesson%209%20%E2%80%94%20Complete%20Advanced%20DBMS-Indexing%2C%20Query%20Optimisation%2C%20Transactions%20and%20Backend%20Engineering-Part-2.md) | Deep dive: B-Tree, join algorithms, WAL, write skew, idempotency, outbox, SKIP LOCKED, Saga |
| [Lesson 9 — SQL Indexes, Transactions and Query Optimisation](Lesson%209%20%E2%80%94%20SQL%20Indexes%2C%20Transactions%20and%20Query%20Optimisation.md) | Indexes, EXPLAIN, ACID, isolation levels, MVCC, locking, deadlocks |
| [Lesson 10 — SQL Company OAs, LeetCode Practice and Mock Interviews](Lesson%2010%20%E2%80%94%20SQL%20Company%20OAs%2C%20LeetCode%20Practice%20and%20Mock%20Interviews.md) | OA patterns (top N, gaps and islands, retention), LeetCode map, interview prep, 3 mock OAs with answer keys |
| [Lesson 11.1 — Advanced Drills Sets 1-2 and Alternative Solutions](Lesson%2011.1%20%E2%80%94%20Advanced%20Drills%20Sets%201-2%20and%20Alternative%20Solutions.md) | Practice drills sets 1-2 and solving one problem several ways |
| [Lesson 11.2 — Advanced Drills Sets 3-4 Query Rewriting](Lesson%2011.2%20%E2%80%94%20Advanced%20Drills%20Sets%203-4%20Query%20Rewriting.md) | Drill set 3 solutions and set 4 (query rewriting) |
| [Lesson 11.3 — Advanced Drills Sets 5-6](Lesson%2011.3%20%E2%80%94%20Advanced%20Drills%20Sets%205-6.md) | Drill sets 5-6 |
| [Lesson 11.4 — Final Placement Preparation Set 7](Lesson%2011.4%20%E2%80%94%20Final%20Placement%20Preparation%20Set%207.md) | Final placement preparation, set 7 |

## Learning path

1. **Query basics** (Lessons 1-3): filtering, aggregation, joins
2. **Advanced querying** (Lessons 4-7): subqueries, CTEs, window functions, built-in functions
3. **Database concepts** (Lessons 8-9): design, normalisation, indexes, transactions, concurrency
4. **Practice** (Lessons 10-11): OA patterns, mock tests, a 100-question interview bank

## Notes on accuracy

- The SQL has been checked by reading, not run against a live database. If you find a mistake, please open an issue or a pull request.
- Quizzes in the lessons were copied from an interactive format, so some show only the questions and options.
- Some lessons link to external articles for illustration.

## Contributing

Corrections, extra practice problems and new examples are welcome. Open an issue, or fork the repo and send a pull request.

## License

Add a license of your choice (for example [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) for notes) before publishing.
