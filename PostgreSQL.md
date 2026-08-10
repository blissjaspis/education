# PostgreSQL: From Beginner to Expert

This course teaches PostgreSQL with practical lessons. Run each example. Do each
exercise before you go to the next lesson.

The examples use PostgreSQL 17. Most examples also work with newer supported
versions.

## Course Goals

At the end of this course, you can:

- Design a relational database.
- Write safe and efficient SQL.
- Use transactions and control concurrent work.
- Read query plans and add useful indexes.
- Manage roles, backups, maintenance, and monitoring.
- Prepare PostgreSQL for production use.
- Store and search vector embeddings with `pgvector`.

## Contents

1. [Setup](#setup)
2. [Beginner Lessons](#beginner-lessons)
3. [Intermediate Lessons](#intermediate-lessons)
4. [Advanced Lessons](#advanced-lessons)
5. [Expert Lessons](#expert-lessons)
6. [Bonus: PostgreSQL and pgvector on Docker](#bonus-postgresql-and-pgvector-on-docker)
7. [Projects](#projects)
8. [Command Reference](#command-reference)
9. [Troubleshooting](#troubleshooting)
10. [Next Steps](#next-steps)

---

## Setup

### Required Tools

You need:

- Docker, or a local PostgreSQL server
- `psql`, DBeaver, DataGrip, or pgAdmin
- A text editor

The lessons use `psql`. GUI clients can run the same SQL.

### Option A: Start PostgreSQL with Docker

```bash
docker run --name postgres-learning \
  -e POSTGRES_USER=student \
  -e POSTGRES_PASSWORD=change_me \
  -e POSTGRES_DB=academy \
  -p 127.0.0.1:5432:5432 \
  -v postgres_learning_data:/var/lib/postgresql/data \
  -d postgres:17
```

Check the container:

```bash
docker ps
docker logs postgres-learning
```

Open `psql` in the container:

```bash
docker exec -it postgres-learning psql -U student -d academy
```

Connect from a local client:

```bash
psql "postgresql://student:change_me@localhost:5432/academy"
```

Stop and start the server:

```bash
docker stop postgres-learning
docker start postgres-learning
```

Do not use `change_me` as a real password.

### Option B: Use a Local PostgreSQL Server

Create the course user and database from an administrator account:

```sql
CREATE ROLE student WITH LOGIN PASSWORD 'change_me';
CREATE DATABASE academy OWNER student;
```

Connect to the database:

```bash
psql -U student -d academy
```

### Useful `psql` Commands

Commands that start with `\` are `psql` commands. They are not SQL.

```text
\?                 Show psql help
\h SELECT          Show SQL help for SELECT
\l                 List databases
\c academy         Connect to a database
\dn                List schemas
\dt                List tables
\d students        Describe a table
\du                List roles
\dx                List extensions
\timing            Show query run time
\x                 Use expanded output
\q                 Quit
```

Make scripts stop after the first error:

```sql
\set ON_ERROR_STOP on
```

### Create the Course Data

Run this script once:

```sql
DROP SCHEMA IF EXISTS learning CASCADE;
CREATE SCHEMA learning;
SET search_path TO learning, public;

CREATE TABLE students (
    student_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    full_name text NOT NULL,
    email text NOT NULL UNIQUE,
    birth_date date,
    created_at timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT students_email_check CHECK (position('@' IN email) > 1)
);

CREATE TABLE courses (
    course_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    course_code text NOT NULL UNIQUE,
    title text NOT NULL,
    fee numeric(10, 2) NOT NULL CHECK (fee >= 0),
    capacity integer NOT NULL CHECK (capacity > 0),
    published boolean NOT NULL DEFAULT false,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE enrollments (
    enrollment_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    student_id bigint NOT NULL REFERENCES students(student_id),
    course_id bigint NOT NULL REFERENCES courses(course_id),
    status text NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'completed', 'cancelled')),
    score numeric(5, 2) CHECK (score BETWEEN 0 AND 100),
    enrolled_at timestamptz NOT NULL DEFAULT now(),
    UNIQUE (student_id, course_id)
);

INSERT INTO students (full_name, email, birth_date) VALUES
    ('Amina Yusuf', 'amina@example.com', '1998-04-12'),
    ('Budi Santoso', 'budi@example.com', '2000-11-03'),
    ('Carla Silva', 'carla@example.com', '1996-07-21'),
    ('Dae Kim', 'dae@example.com', NULL);

INSERT INTO courses (course_code, title, fee, capacity, published) VALUES
    ('SQL-101', 'SQL Basics', 49.00, 30, true),
    ('PG-201', 'PostgreSQL Operations', 99.00, 20, true),
    ('DATA-301', 'Data Modeling', 129.00, 15, false);

INSERT INTO enrollments (student_id, course_id, status, score) VALUES
    (1, 1, 'completed', 91.50),
    (1, 2, 'active', NULL),
    (2, 1, 'active', 78.00),
    (3, 1, 'completed', 88.00),
    (3, 2, 'cancelled', NULL);
```

Use the course schema in each new session:

```sql
SET search_path TO learning, public;
```

---

## Beginner Lessons

## Lesson 1: Relational Database Basics

### Goal

Understand the main PostgreSQL objects.

### Main Ideas

- A PostgreSQL server contains databases.
- A database contains schemas.
- A schema contains tables, views, functions, and other objects.
- A table contains rows and columns.
- A primary key identifies one row.
- A foreign key connects rows in different tables.
- A constraint stops invalid data.
- A role is a user or a group of users.

PostgreSQL uses a client-server model. The server stores and processes data. A
client sends SQL to the server.

### Inspect the Course Database

```sql
SELECT current_database();
SELECT current_user;
SHOW server_version;
SHOW search_path;

SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema = 'learning'
ORDER BY table_name;
```

### Exercise

1. List the databases.
2. List the tables in the `learning` schema.
3. Describe the `students` table.

## Lesson 2: Data Types and Table Design

### Goal

Select data types that keep data correct.

### Common Data Types

- `smallint`, `integer`, and `bigint` store whole numbers.
- `numeric(p, s)` stores exact decimal values. Use it for money.
- `real` and `double precision` store approximate values.
- `text` and `varchar(n)` store text.
- `boolean` stores `true`, `false`, or `NULL`.
- `date` stores a calendar date.
- `time` stores a time of day.
- `timestamp` stores date and time without a time zone.
- `timestamptz` stores an instant in time. Use it for events.
- `uuid` stores a universally unique identifier.
- `jsonb` stores indexed JSON data.
- Arrays store a list of values of one type.

`NULL` means that a value is unknown or absent. It does not mean zero or an
empty string.

### Create a Table

```sql
CREATE TABLE instructors (
    instructor_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    full_name text NOT NULL,
    hourly_rate numeric(8, 2) NOT NULL CHECK (hourly_rate > 0),
    active boolean NOT NULL DEFAULT true,
    hired_on date NOT NULL DEFAULT current_date
);
```

### Change a Table

```sql
ALTER TABLE instructors ADD COLUMN biography text;
ALTER TABLE instructors RENAME COLUMN biography TO bio;
ALTER TABLE instructors ALTER COLUMN bio SET DEFAULT '';
```

Remove the practice table:

```sql
DROP TABLE instructors;
```

### Design Rules

- Use the smallest type that safely supports expected growth.
- Use `text` unless a strict maximum length is a business rule.
- Use `timestamptz` for real-world events.
- Use `numeric`, not floating-point types, for exact money values.
- Add `NOT NULL`, `CHECK`, `UNIQUE`, and foreign keys when the rule is stable.
- Do not store comma-separated values in one column.

### Exercise

Create an `instructors` table. Add a unique email and a nonnegative salary.

## Lesson 3: Create, Read, Update, and Delete Data

### Goal

Use the four basic data operations.

### Insert

```sql
INSERT INTO students (full_name, email, birth_date)
VALUES ('Elena Rossi', 'elena@example.com', '1999-02-15')
RETURNING student_id, created_at;
```

Insert more than one row:

```sql
INSERT INTO courses (course_code, title, fee, capacity, published) VALUES
    ('API-101', 'API Design', 79.00, 25, true),
    ('OPS-101', 'Operations Basics', 89.00, 20, false)
RETURNING *;
```

### Read

```sql
SELECT * FROM students;

SELECT full_name, email
FROM students
ORDER BY full_name;
```

### Update

Always test the `WHERE` condition with `SELECT` first.

```sql
SELECT * FROM courses WHERE course_code = 'API-101';

UPDATE courses
SET fee = 84.00,
    published = true
WHERE course_code = 'API-101'
RETURNING *;
```

### Delete

```sql
DELETE FROM courses
WHERE course_code = 'OPS-101'
RETURNING *;
```

An `UPDATE` or `DELETE` without `WHERE` changes all rows.

### Upsert

An upsert inserts a row or changes the existing conflicting row.

```sql
INSERT INTO students (full_name, email)
VALUES ('Amina Yusuf', 'amina@example.com')
ON CONFLICT (email)
DO UPDATE SET full_name = EXCLUDED.full_name
RETURNING *;
```

### Exercise

1. Add a new course.
2. Increase its fee by 10 percent.
3. Delete it and use `RETURNING` to show the deleted row.

## Lesson 4: Filter and Sort Data

### Goal

Get only the rows that you need.

```sql
SELECT *
FROM courses
WHERE published = true
  AND fee BETWEEN 40 AND 100
ORDER BY fee DESC, title ASC
LIMIT 10 OFFSET 0;
```

Useful conditions:

```sql
SELECT * FROM students WHERE birth_date IS NULL;
SELECT * FROM students WHERE full_name LIKE 'A%';
SELECT * FROM students WHERE full_name ILIKE '%silva%';
SELECT * FROM courses WHERE course_code IN ('SQL-101', 'PG-201');
SELECT * FROM courses WHERE NOT published;
```

Use `IS NULL` and `IS NOT NULL`. Do not use `= NULL`.

### Expressions and Functions

```sql
SELECT
    full_name,
    upper(full_name) AS uppercase_name,
    coalesce(birth_date::text, 'Not provided') AS birth_date_text
FROM students;

SELECT
    title,
    fee,
    round(fee * 1.11, 2) AS fee_with_tax
FROM courses;
```

Use a `CASE` expression for conditional output:

```sql
SELECT
    title,
    fee,
    CASE
        WHEN fee < 60 THEN 'low'
        WHEN fee < 100 THEN 'medium'
        ELSE 'high'
    END AS price_group
FROM courses;
```

### Exercise

Find all published courses that cost less than 100. Sort the result from the
highest fee to the lowest fee.

## Lesson 5: Join Tables

### Goal

Combine related data.

### Inner Join

An inner join returns only matching rows.

```sql
SELECT
    s.full_name,
    c.title,
    e.status,
    e.score
FROM enrollments AS e
JOIN students AS s ON s.student_id = e.student_id
JOIN courses AS c ON c.course_id = e.course_id
ORDER BY s.full_name, c.title;
```

### Left Join

A left join keeps all rows from the left table.

```sql
SELECT
    s.full_name,
    count(e.enrollment_id) AS enrollment_count
FROM students AS s
LEFT JOIN enrollments AS e ON e.student_id = s.student_id
GROUP BY s.student_id, s.full_name
ORDER BY s.full_name;
```

`Dae Kim` stays in the result, although that student has no enrollment.

### Other Join Types

- `RIGHT JOIN` keeps all rows from the right table.
- `FULL JOIN` keeps all rows from both tables.
- `CROSS JOIN` returns all combinations.
- A self join joins a table to itself.

Prefer `JOIN ... ON` syntax. It makes the join condition clear.

### Exercise

Show each course and its number of active enrollments. Include courses that
have zero active enrollments.

## Lesson 6: Aggregate Data

### Goal

Summarize groups of rows.

```sql
SELECT
    status,
    count(*) AS total,
    round(avg(score), 2) AS average_score,
    min(score) AS minimum_score,
    max(score) AS maximum_score
FROM enrollments
GROUP BY status
ORDER BY status;
```

Use `HAVING` to filter groups:

```sql
SELECT
    c.course_id,
    c.title,
    count(e.enrollment_id) AS student_count
FROM courses AS c
LEFT JOIN enrollments AS e ON e.course_id = c.course_id
GROUP BY c.course_id, c.title
HAVING count(e.enrollment_id) >= 2;
```

Important behavior:

- `count(*)` counts rows.
- `count(column)` counts non-`NULL` values.
- Most aggregate functions ignore `NULL`.
- `WHERE` filters rows before grouping.
- `HAVING` filters groups after grouping.

### Exercise

Show the average completed score for each course. Only show averages of 80 or
more.

---

## Intermediate Lessons

## Lesson 7: Constraints, Keys, and Relationships

### Goal

Make PostgreSQL enforce data rules.

Main constraints:

- `PRIMARY KEY` is unique and not null.
- `FOREIGN KEY` requires a matching parent row.
- `UNIQUE` stops duplicate values.
- `NOT NULL` requires a value.
- `CHECK` checks a condition.
- `EXCLUDE` stops conflicts that a normal unique constraint cannot express.

### Foreign Key Actions

```sql
CREATE TABLE course_notes (
    note_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    course_id bigint NOT NULL
        REFERENCES courses(course_id)
        ON UPDATE CASCADE
        ON DELETE CASCADE,
    body text NOT NULL
);
```

Common delete actions are:

- `RESTRICT` or `NO ACTION`: reject the parent deletion.
- `CASCADE`: delete related child rows.
- `SET NULL`: keep the child row and clear the reference.

Use `CASCADE` only when child data has no useful meaning without its parent.

### Composite Constraints

The course schema uses this rule:

```sql
UNIQUE (student_id, course_id)
```

It permits one enrollment for each student and course pair.

### Deferrable Constraints

A deferrable constraint can wait until transaction commit:

```sql
ALTER TABLE enrollments
    DROP CONSTRAINT enrollments_student_id_fkey,
    ADD CONSTRAINT enrollments_student_id_fkey
        FOREIGN KEY (student_id)
        REFERENCES students(student_id)
        DEFERRABLE INITIALLY IMMEDIATE;
```

Use this feature only when one transaction must temporarily break a relation.

### Exercise

Create an `assignments` table. Connect it to `courses`. Make assignment names
unique inside each course.

## Lesson 8: Schemas and Object Names

### Goal

Organize objects and control name resolution.

```sql
CREATE SCHEMA reporting;

CREATE VIEW reporting.published_courses AS
SELECT course_code, title, fee
FROM learning.courses
WHERE published = true;
```

The `search_path` controls how PostgreSQL resolves names without schema names:

```sql
SHOW search_path;
SET search_path TO learning, public;
```

In application code and security-sensitive functions, use schema-qualified
names such as `learning.students`.

Do not let untrusted users create objects in schemas that are in another
user's `search_path`.

### Exercise

Create an `audit` schema and an empty `audit.events` table.

## Lesson 9: Subqueries, CTEs, and Set Operations

### Goal

Build complex queries in clear parts.

### Scalar Subquery

```sql
SELECT title, fee
FROM courses
WHERE fee > (SELECT avg(fee) FROM courses);
```

### `EXISTS`

`EXISTS` is useful when you only need to know if a matching row exists:

```sql
SELECT s.student_id, s.full_name
FROM students AS s
WHERE EXISTS (
    SELECT 1
    FROM enrollments AS e
    WHERE e.student_id = s.student_id
      AND e.status = 'active'
);
```

### Common Table Expression

```sql
WITH course_totals AS (
    SELECT course_id, count(*) AS total
    FROM enrollments
    WHERE status = 'active'
    GROUP BY course_id
)
SELECT c.title, coalesce(ct.total, 0) AS active_students
FROM courses AS c
LEFT JOIN course_totals AS ct USING (course_id)
ORDER BY active_students DESC;
```

### Set Operations

```sql
SELECT student_id
FROM enrollments
WHERE course_id = 1

INTERSECT

SELECT student_id
FROM enrollments
WHERE course_id = 2;
```

- `UNION` combines and removes duplicate rows.
- `UNION ALL` combines and keeps duplicate rows.
- `INTERSECT` returns rows in both results.
- `EXCEPT` returns rows only in the first result.

### Recursive CTE

```sql
WITH RECURSIVE numbers(n) AS (
    SELECT 1
    UNION ALL
    SELECT n + 1
    FROM numbers
    WHERE n < 5
)
SELECT n FROM numbers;
```

Recursive CTEs are useful for trees, graphs, and generated sequences.

### Exercise

Find students who are not enrolled in any course. Write one query with
`NOT EXISTS` and one query with `LEFT JOIN`.

## Lesson 10: Window Functions

### Goal

Calculate values across related rows without grouping them into one row.

```sql
SELECT
    c.title,
    s.full_name,
    e.score,
    rank() OVER (
        PARTITION BY e.course_id
        ORDER BY e.score DESC NULLS LAST
    ) AS score_rank,
    round(avg(e.score) OVER (PARTITION BY e.course_id), 2) AS course_average
FROM enrollments AS e
JOIN courses AS c USING (course_id)
JOIN students AS s USING (student_id)
ORDER BY c.title, score_rank;
```

Common window functions:

- `row_number()` gives each row a unique sequence number.
- `rank()` leaves gaps after tied values.
- `dense_rank()` does not leave gaps.
- `lag()` reads a previous row.
- `lead()` reads a following row.
- `sum() OVER (...)` makes a running total.

Running total example:

```sql
SELECT
    enrolled_at,
    enrollment_id,
    count(*) OVER (
        ORDER BY enrolled_at, enrollment_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total
FROM enrollments;
```

### Exercise

Rank courses by fee. Also show the difference between each fee and the prior
fee.

## Lesson 11: Transactions and ACID

### Goal

Change related data as one safe unit.

ACID means:

- Atomicity: all changes succeed or all changes fail.
- Consistency: constraints stay valid.
- Isolation: concurrent transactions have controlled interaction.
- Durability: committed data survives a failure.

### Basic Transaction

```sql
BEGIN;

SELECT course_id
FROM courses
WHERE course_id = 1
FOR UPDATE;

INSERT INTO enrollments (student_id, course_id)
SELECT 4, c.course_id
FROM courses AS c
WHERE c.course_id = 1
  AND (
      SELECT count(*)
      FROM enrollments AS e
      WHERE e.course_id = c.course_id
        AND e.status = 'active'
  ) < c.capacity
RETURNING enrollment_id;

COMMIT;
```

The application must check that `RETURNING` gives one row. If it gives no row,
the course is full. All enrollment code must use the same lock rule.

Use `ROLLBACK` when a step fails:

```sql
BEGIN;
UPDATE courses SET fee = -1 WHERE course_id = 1;
ROLLBACK;
```

### Savepoint

```sql
BEGIN;
UPDATE courses SET fee = fee + 5;
SAVEPOINT before_publish;
UPDATE courses SET published = true;
ROLLBACK TO SAVEPOINT before_publish;
COMMIT;
```

### Isolation Levels

PostgreSQL supports:

- `READ COMMITTED`: each statement sees committed data at its start. This is
  the default.
- `REPEATABLE READ`: the transaction uses one stable snapshot.
- `SERIALIZABLE`: the result is as if transactions ran one at a time.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Read and write data.
COMMIT;
```

A serializable transaction can fail with a serialization error. The
application must retry the complete transaction.

### Exercise

Write a transaction that creates a student and enrolls the new student in a
course. Return the new identifiers.

## Lesson 12: Locks and Concurrent Work

### Goal

Prevent race conditions without blocking more rows than necessary.

Lock one row before a dependent change:

```sql
BEGIN;

SELECT capacity
FROM courses
WHERE course_id = 2
FOR UPDATE;

INSERT INTO enrollments (student_id, course_id)
SELECT 2, c.course_id
FROM courses AS c
WHERE c.course_id = 2
  AND (
      SELECT count(*)
      FROM enrollments AS e
      WHERE e.course_id = c.course_id
        AND e.status = 'active'
  ) < c.capacity;

COMMIT;
```

Queue workers can skip rows that another worker has locked:

```sql
SELECT enrollment_id
FROM enrollments
WHERE status = 'active'
ORDER BY enrollment_id
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

Useful controls:

```sql
BEGIN;
SET LOCAL lock_timeout = '2s';
SET LOCAL statement_timeout = '10s';
-- Run the required statements.
COMMIT;
```

Keep transactions short. Do not wait for HTTP calls or user input inside a
transaction.

### Find Blocked Sessions

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE datname = current_database();
```

### Exercise

Open two `psql` sessions. Lock the same course row in both sessions. Inspect the
blocked session from a third session.

## Lesson 13: Indexes and Query Plans

### Goal

Add indexes that support real queries.

PostgreSQL automatically creates indexes for primary key and unique
constraints. It does not automatically index the referencing side of a foreign
key.

```sql
CREATE INDEX enrollments_student_id_idx
    ON enrollments (student_id);

CREATE INDEX enrollments_course_status_idx
    ON enrollments (course_id, status);
```

The order of columns in a multicolumn B-tree index is important. Put columns
used for equality checks before columns used for ranges when that matches the
main query.

### Partial Index

```sql
CREATE INDEX enrollments_active_course_idx
    ON enrollments (course_id, enrolled_at)
    WHERE status = 'active';
```

### Expression Index

```sql
CREATE UNIQUE INDEX students_email_lower_uidx
    ON students (lower(email));
```

### Covering Index

```sql
CREATE INDEX courses_published_idx
    ON courses (published, fee)
    INCLUDE (title);
```

### Read a Plan

```sql
EXPLAIN
SELECT * FROM enrollments WHERE student_id = 1;

EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM enrollments
WHERE course_id = 1
  AND status = 'active';
```

`ANALYZE` runs the query. Do not use it on a destructive statement unless you
put that statement in a transaction that you roll back.

Look for:

- Large differences between estimated rows and actual rows.
- Sequential scans on large tables for selective conditions.
- Large numbers of rows removed by a filter.
- Disk sorts or hash operations that use many batches.
- Expensive nested loops over many rows.
- High shared read counts and temporary file use.

A sequential scan is often correct for a small table or a query that reads most
rows.

### Safe Index Creation

On a busy production table, use:

```sql
CREATE INDEX CONCURRENTLY enrollments_status_idx
    ON enrollments (status);
```

`CREATE INDEX CONCURRENTLY` cannot run inside a transaction block. It takes
longer, but it permits normal writes during most of the operation.

### Exercise

Use `EXPLAIN (ANALYZE, BUFFERS)` on a query before and after you add an index.
Explain why the plan changed or did not change.

## Lesson 14: Views, Materialized Views, and Functions

### Goal

Reuse query logic.

### View

```sql
CREATE OR REPLACE VIEW reporting.course_summary AS
SELECT
    c.course_id,
    c.title,
    count(e.enrollment_id) AS enrollment_count,
    round(avg(e.score), 2) AS average_score
FROM learning.courses AS c
LEFT JOIN learning.enrollments AS e USING (course_id)
GROUP BY c.course_id, c.title;
```

A normal view stores a query, not its result.

### Materialized View

```sql
CREATE MATERIALIZED VIEW reporting.course_summary_cache AS
SELECT * FROM reporting.course_summary;

CREATE UNIQUE INDEX course_summary_cache_pk
    ON reporting.course_summary_cache (course_id);

REFRESH MATERIALIZED VIEW CONCURRENTLY reporting.course_summary_cache;
```

A materialized view stores results. Refresh it when source data changes.
Concurrent refresh needs a suitable unique index.

### SQL Function

```sql
CREATE OR REPLACE FUNCTION learning.course_student_count(p_course_id bigint)
RETURNS bigint
LANGUAGE sql
STABLE
AS $$
    SELECT count(*)
    FROM learning.enrollments
    WHERE course_id = p_course_id
      AND status = 'active';
$$;

SELECT learning.course_student_count(1);
```

Use the correct volatility:

- `VOLATILE` can change data or return different values in one statement.
- `STABLE` is stable during one statement.
- `IMMUTABLE` always returns the same result for the same arguments.

Do not mark a function `IMMUTABLE` if it reads tables or depends on settings.

### Exercise

Create a view that shows active enrollments with student and course names.

## Lesson 15: JSONB, Arrays, and Full-Text Search

### Goal

Use PostgreSQL data types for flexible data and search.

### JSONB

```sql
CREATE TABLE student_profiles (
    student_id bigint PRIMARY KEY REFERENCES students(student_id),
    preferences jsonb NOT NULL DEFAULT '{}'::jsonb
);

INSERT INTO student_profiles (student_id, preferences) VALUES
    (1, '{"theme":"dark","topics":["sql","docker"]}'),
    (2, '{"theme":"light","topics":["api"]}');

SELECT
    student_id,
    preferences ->> 'theme' AS theme
FROM student_profiles
WHERE preferences @> '{"theme":"dark"}';

CREATE INDEX student_profiles_preferences_gin_idx
    ON student_profiles USING gin (preferences);
```

Use normal columns for stable, important fields. Use `jsonb` for data that is
truly flexible.

### Arrays

```sql
ALTER TABLE courses ADD COLUMN tags text[] NOT NULL DEFAULT '{}';

UPDATE courses
SET tags = ARRAY['database', 'beginner']
WHERE course_code = 'SQL-101';

SELECT * FROM courses WHERE tags @> ARRAY['database'];

CREATE INDEX courses_tags_gin_idx ON courses USING gin (tags);
```

Do not use arrays as a replacement for relations when array elements need
their own attributes or foreign keys.

### Full-Text Search

```sql
ALTER TABLE courses
ADD COLUMN search_document tsvector
GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(course_code, '')), 'B')
) STORED;

CREATE INDEX courses_search_document_gin_idx
    ON courses USING gin (search_document);

SELECT
    course_code,
    title,
    ts_rank(search_document, websearch_to_tsquery('english', 'postgresql operations'))
        AS rank
FROM courses
WHERE search_document @@ websearch_to_tsquery('english', 'postgresql operations')
ORDER BY rank DESC;
```

Select a text search configuration that matches the content language.

### Exercise

Add a JSONB metadata column to a practice table. Query a nested value and add a
GIN index.

---

## Advanced Lessons

## Lesson 16: MVCC, Vacuum, and Statistics

### Goal

Understand how PostgreSQL handles concurrent versions of rows.

PostgreSQL uses Multi-Version Concurrency Control (MVCC). An update normally
creates a new row version. Old row versions remain until PostgreSQL can remove
them.

Autovacuum:

- Removes dead row versions for reuse.
- Freezes old transaction identifiers.
- Updates planner statistics through auto-analyze.
- Helps prevent transaction ID wraparound.

Do not disable autovacuum for the whole server.

### Inspect Table Health

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

Run maintenance manually when necessary:

```sql
VACUUM (ANALYZE, VERBOSE) learning.enrollments;
ANALYZE learning.enrollments;
```

Normal `VACUUM` usually does not return disk space to the operating system.
`VACUUM FULL` rewrites and locks the table. Use it only after careful planning.

Long transactions stop PostgreSQL from removing old row versions. Find them:

```sql
SELECT
    pid,
    usename,
    xact_start,
    state,
    query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

### Statistics

If estimates are poor, increase statistics for a difficult column:

```sql
ALTER TABLE enrollments
ALTER COLUMN status SET STATISTICS 500;

ANALYZE enrollments;
```

Use extended statistics for related columns:

```sql
CREATE STATISTICS enrollments_course_status_stats
    (dependencies, mcv)
    ON course_id, status
    FROM enrollments;

ANALYZE enrollments;
```

### Exercise

Update many rows in a practice table. Inspect dead tuple estimates before and
after `VACUUM (ANALYZE)`.

## Lesson 17: Query Tuning

### Goal

Use evidence to make slow queries faster.

Use this sequence:

1. Save the exact slow query and its parameters.
2. Measure its normal run time.
3. Get `EXPLAIN (ANALYZE, BUFFERS, WAL, SETTINGS)`.
4. Check row estimates and the most expensive nodes.
5. Check table statistics and data distribution.
6. Change one item.
7. Measure again.

Do not add an index to every column. Each index uses storage and makes writes
more expensive.

### Frequent Improvements

- Return only required columns.
- Filter rows early when it reduces work.
- Remove repeated queries from application loops.
- Add an index that matches `WHERE`, `JOIN`, and `ORDER BY`.
- Rewrite correlated work when it repeats for many rows.
- Keep planner statistics current.
- Use keyset pagination for deep pages.

### Keyset Pagination

Offset pagination becomes expensive at high offsets:

```sql
SELECT enrollment_id, enrolled_at
FROM enrollments
ORDER BY enrolled_at DESC, enrollment_id DESC
OFFSET 100000 LIMIT 20;
```

Use the last value from the prior page:

```sql
SELECT enrollment_id, enrolled_at
FROM enrollments
WHERE (enrolled_at, enrollment_id) < ($1, $2)
ORDER BY enrolled_at DESC, enrollment_id DESC
LIMIT 20;
```

Support it with an index:

```sql
CREATE INDEX enrollments_page_idx
    ON enrollments (enrolled_at DESC, enrollment_id DESC);
```

### Server Memory

Important settings include:

- `shared_buffers`: PostgreSQL shared cache.
- `work_mem`: limit for one sort or hash operation, not one connection.
- `maintenance_work_mem`: memory for maintenance work.
- `effective_cache_size`: planner estimate of available cache.

Do not multiply a large `work_mem` value by only the number of connections. One
query can use it for several plan nodes and parallel workers.

### Exercise

Create 100,000 practice rows with `generate_series()`. Compare offset
pagination with keyset pagination.

## Lesson 18: Partitioning

### Goal

Split a very large table into manageable parts.

Partitioning can help with:

- Removing old data quickly.
- Loading or moving data by time range.
- Limiting scans when partition pruning applies.
- Maintaining indexes in smaller units.

It does not automatically make every query faster.

### Range Partition Example

```sql
CREATE TABLE audit_events (
    event_id bigint GENERATED ALWAYS AS IDENTITY,
    occurred_at timestamptz NOT NULL,
    actor_id bigint,
    event_type text NOT NULL,
    details jsonb NOT NULL DEFAULT '{}',
    PRIMARY KEY (occurred_at, event_id)
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_events_2026_08
PARTITION OF audit_events
FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');

CREATE TABLE audit_events_default
PARTITION OF audit_events DEFAULT;

CREATE INDEX audit_events_actor_idx
    ON audit_events (actor_id, occurred_at);
```

The partition key must be part of a unique or primary key on the partitioned
table.

Plan how to:

- Create future partitions before data arrives.
- Move rows out of the default partition.
- Remove or archive old partitions.
- Monitor partition sizes.

### Exercise

Create monthly partitions for three months. Insert rows and use `EXPLAIN` to
confirm partition pruning.

## Lesson 19: Roles, Privileges, and Row Security

### Goal

Give each user only the required access.

PostgreSQL roles can log in, own objects, or group other roles.

```sql
CREATE ROLE academy_readonly NOLOGIN;
CREATE ROLE academy_app NOLOGIN;
CREATE ROLE report_user LOGIN PASSWORD 'replace_this_password';

GRANT CONNECT ON DATABASE academy TO academy_readonly, academy_app;
GRANT USAGE ON SCHEMA learning TO academy_readonly, academy_app;

GRANT SELECT ON ALL TABLES IN SCHEMA learning TO academy_readonly;
GRANT SELECT, INSERT, UPDATE, DELETE
    ON ALL TABLES IN SCHEMA learning TO academy_app;
GRANT USAGE, SELECT
    ON ALL SEQUENCES IN SCHEMA learning TO academy_app;

GRANT academy_readonly TO report_user;
```

Set privileges for future objects. Run these commands as the role that will
create the objects:

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA learning
GRANT SELECT ON TABLES TO academy_readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA learning
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO academy_app;

ALTER DEFAULT PRIVILEGES IN SCHEMA learning
GRANT USAGE, SELECT ON SEQUENCES TO academy_app;
```

Security rules:

- Applications must not connect as a superuser or object owner.
- Use separate roles for migrations, applications, and people.
- Store secrets outside source code.
- Require TLS on untrusted networks.
- Review `pg_hba.conf`.
- Set a safe `search_path` in security-definer functions.

### Row-Level Security

```sql
CREATE TABLE tenant_notes (
    note_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tenant_id bigint NOT NULL,
    body text NOT NULL
);

ALTER TABLE tenant_notes ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_notes FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tenant_notes
USING (tenant_id = current_setting('app.tenant_id')::bigint)
WITH CHECK (tenant_id = current_setting('app.tenant_id')::bigint);
```

Set tenant context in a transaction:

```sql
BEGIN;
SET LOCAL app.tenant_id = '42';
SELECT * FROM tenant_notes;
COMMIT;
```

RLS is an additional control. The application must still validate identity and
set tenant context safely. Test policies for `SELECT`, `INSERT`, `UPDATE`, and
`DELETE`.

### Exercise

Create separate read-only and read-write roles for a practice schema. Test each
role with `SET ROLE`.

## Lesson 20: Backup, Restore, and Migration

### Goal

Recover data and change schemas safely.

A backup is useful only when a restore test succeeds.

### Logical Backup

Custom format is flexible and supports parallel restore:

```bash
pg_dump \
  --format=custom \
  --file=academy.dump \
  --dbname="postgresql://student@localhost:5432/academy"
```

Restore to an empty database:

```bash
createdb -h localhost -U student academy_restore

pg_restore \
  --clean \
  --if-exists \
  --no-owner \
  --dbname="postgresql://student@localhost:5432/academy_restore" \
  academy.dump
```

Backup all global roles and tablespaces:

```bash
pg_dumpall --globals-only > globals.sql
```

Plain SQL backup:

```bash
pg_dump --format=plain academy > academy.sql
psql -d academy_restore -f academy.sql
```

For large production systems, use physical backups and continuous WAL
archiving. These support point-in-time recovery.

### Safe Schema Migration

For an important table, use an expand-and-contract migration:

1. Add a nullable column without a heavy default rewrite.
2. Deploy code that can use both old and new forms.
3. Backfill data in small batches.
4. Add constraints with a low-lock method when possible.
5. Deploy code that uses the new form.
6. Remove the old form in a later release.

Example:

```sql
ALTER TABLE students ADD COLUMN display_name text;

UPDATE students
SET display_name = full_name
WHERE student_id > $1
  AND student_id <= $2
  AND display_name IS NULL;

ALTER TABLE students
ADD CONSTRAINT students_display_name_present
CHECK (display_name IS NOT NULL) NOT VALID;

ALTER TABLE students
VALIDATE CONSTRAINT students_display_name_present;

ALTER TABLE students
ALTER COLUMN display_name SET NOT NULL;
```

Set `lock_timeout` during risky migrations. A timeout is often safer than an
unexpected long production lock.

### Exercise

Make a custom-format backup. Restore it to a new database. Compare row counts
and important constraints.

## Lesson 21: Monitoring and Connection Management

### Goal

Find database problems before users report them.

### Current Activity

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    xact_start,
    query_start,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE datname = current_database()
ORDER BY query_start;
```

### Database Statistics

```sql
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    temp_files,
    temp_bytes,
    deadlocks
FROM pg_stat_database
WHERE datname = current_database();
```

### Index Use

```sql
SELECT
    schemaname,
    relname,
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan, pg_relation_size(indexrelid) DESC;
```

Do not remove an index only because `idx_scan` is zero. Statistics can reset,
and an index can support rare but critical work or a constraint.

### Query Statistics

`pg_stat_statements` records aggregated query statistics. The server must load
it through `shared_preload_libraries` before you create the extension.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS mean_ms,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Connection Pooling

Each PostgreSQL connection uses resources. Use a connection pool when an
application can create many connections.

Common choices are:

- An application pool for simple deployments.
- PgBouncer for many application instances or short connections.

Do not increase `max_connections` without a memory and workload plan.

Monitor:

- Availability and connection use.
- Query latency and throughput.
- Lock waits and deadlocks.
- Replication lag.
- Disk space, WAL growth, and temporary files.
- Checkpoints and I/O latency.
- Dead tuples and autovacuum progress.
- Backup age and restore test results.

### Exercise

Run a slow query in one session. Find it in `pg_stat_activity` from another
session.

---

## Expert Lessons

## Lesson 22: PostgreSQL Storage and WAL

### Goal

Understand the internal parts that affect operations.

Important terms:

- A heap is the main table storage.
- A page is a fixed-size storage block. The usual size is 8 KiB.
- A tuple is a row version.
- TOAST stores large column values outside the main row when necessary.
- WAL is the write-ahead log. PostgreSQL records changes in WAL before data
  pages reach durable storage.
- A checkpoint makes recovery start from a newer safe point.
- A relation fork stores main data, free-space information, visibility
  information, or initialization data.

WAL supports crash recovery, physical replication, and point-in-time recovery.
WAL is not a replacement for a tested backup.

### HOT Updates

A Heap-Only Tuple update can avoid new index entries when:

- No indexed column changes.
- The table page has enough free space.

For update-heavy tables, a lower fill factor can leave page space:

```sql
ALTER TABLE a_frequently_updated_table SET (fillfactor = 80);
```

This setting affects new writes and table rewrites. Measure storage and
performance before you use it.

### Size Inspection

```sql
SELECT
    pg_size_pretty(pg_database_size(current_database())) AS database_size;

SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

### Exercise

Compare table size, index size, and total size for each course table. Explain
the difference.

## Lesson 23: Planner and Execution Details

### Goal

Understand why PostgreSQL selects a plan.

The planner uses:

- Table and column statistics.
- Estimated row counts.
- Cost settings.
- Available indexes.
- Join order and join methods.
- Parallel work settings.

Common scan nodes:

- Sequential Scan
- Index Scan
- Index Only Scan
- Bitmap Index Scan and Bitmap Heap Scan

Common join nodes:

- Nested Loop: good when one input is small and the other has a useful index.
- Hash Join: good for many equality joins.
- Merge Join: good when both inputs are sorted on join keys.

An index-only scan can still read heap pages when the visibility map does not
show that all tuples on a page are visible.

### Planner Controls for Diagnosis

You can disable a plan type in one session to test a hypothesis:

```sql
BEGIN;
SET LOCAL enable_nestloop = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM learning.enrollments WHERE student_id = 1;
ROLLBACK;
```

Do not use planner switches as the normal fix. Correct statistics, indexes,
query design, and data model first.

### Prepared Statements and Plan Selection

PostgreSQL can use a custom plan for parameter values or a generic plan for all
values. Highly skewed data can make one generic plan inefficient.

```sql
PREPARE enrollment_lookup(text) AS
SELECT * FROM enrollments WHERE status = $1;

EXPLAIN (ANALYZE, BUFFERS)
EXECUTE enrollment_lookup('active');

DEALLOCATE enrollment_lookup;
```

Inspect parameter distributions before you force a plan policy.

### Exercise

Build skewed data where 99 percent of rows have one status. Compare plans for a
common value and a rare value.

## Lesson 24: Replication, High Availability, and Recovery

### Goal

Design for failures.

### Physical Streaming Replication

A primary server sends WAL to standby servers. A standby can:

- Take over after primary failure.
- Serve read-only queries.
- Provide another backup source.

Replication is not a backup. A bad `DELETE` can replicate to every standby.

### Synchronous and Asynchronous Replication

- Asynchronous replication gives lower commit latency. Recent commits can be
  lost during failover.
- Synchronous replication waits for a configured standby. It reduces data loss
  risk but increases latency and can reduce availability.

Choose the mode from business recovery requirements.

### Recovery Targets

- Recovery Point Objective (RPO): acceptable data loss.
- Recovery Time Objective (RTO): acceptable service restoration time.

These targets control architecture, backup frequency, WAL archiving, and
failover design.

### Logical Replication

Logical replication sends changes for selected tables. It is useful for:

- Moving data between major versions.
- Sending selected data to another system.
- Low-downtime migration.

It does not automatically copy all schema changes, sequences, large objects, or
every database object. Plan these items separately.

### Failover Rules

- Use a proven coordinator or managed service.
- Stop split-brain. Only one writable primary must exist.
- Fence the old primary before it can accept writes again.
- Test failover and failback.
- Monitor replication slots. An inactive slot can retain large amounts of WAL.

Inspect replication:

```sql
SELECT
    application_name,
    client_addr,
    state,
    sync_state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn
FROM pg_stat_replication;
```

### Exercise

Write RPO and RTO targets for a small store, a bank payment system, and an
analytics system. Explain how the targets change the design.

## Lesson 25: Advanced Concurrency Patterns

### Goal

Build correct work queues and distributed controls.

### Work Queue

```sql
WITH selected_jobs AS (
    SELECT job_id
    FROM jobs
    WHERE status = 'ready'
    ORDER BY priority DESC, job_id
    FOR UPDATE SKIP LOCKED
    LIMIT 10
)
UPDATE jobs AS j
SET status = 'running',
    started_at = now()
FROM selected_jobs AS s
WHERE j.job_id = s.job_id
RETURNING j.*;
```

The status update and job selection occur in one statement.

### Advisory Locks

Advisory locks coordinate application-defined resources:

```sql
SELECT pg_try_advisory_xact_lock(42);
```

Transaction advisory locks are released at transaction end. Session advisory
locks remain until release or disconnect. Prefer transaction locks when
possible.

All applications must use the same lock key rules. PostgreSQL does not connect
an advisory lock to a table row automatically.

### Idempotency

Use a unique key to stop duplicate request processing:

```sql
CREATE TABLE processed_requests (
    idempotency_key text PRIMARY KEY,
    response jsonb,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

Attempt the insert in the same transaction as the business change. Define what
must happen when the same key arrives with different request content.

### Exercise

Design a job table and a worker query. Include retry count, next attempt time,
error details, and a terminal failure state.

## Lesson 26: Production Design Review

### Goal

Review a PostgreSQL system before launch.

### Data Model

- Each table has a stable key.
- Data types match the business domain.
- Constraints enforce stable rules.
- Foreign key columns have indexes when required by query or delete patterns.
- Large tables have retention and archival rules.
- Time values use a clear time-zone policy.

### Application

- Queries use parameters, not string concatenation.
- Transactions are short and have retry rules.
- Timeouts exist for connections, locks, and statements.
- Connection pools have measured limits.
- Pagination is stable.
- Batch jobs use bounded batches.

### Security

- The application is not a superuser or owner.
- Roles follow least privilege.
- TLS protects network traffic.
- Secrets have rotation procedures.
- Public schema privileges are reviewed.
- Sensitive data is encrypted or tokenized as required.
- Logs do not expose passwords or private data.

### Operations

- Backups and WAL archives have retention rules.
- Restore tests run on a schedule.
- Monitoring has useful alerts and runbooks.
- Disk growth and WAL growth are monitored.
- Autovacuum is tuned for large or high-change tables.
- Major version upgrade procedures are tested.
- Failover and disaster recovery tests have owners.

### Performance

- Top queries are measured with production-like data.
- Indexes support important access paths.
- Unused and duplicate indexes are reviewed.
- Query plans are checked after major data growth.
- Capacity tests include expected concurrency.

---

## Bonus: PostgreSQL and pgvector on Docker

`pgvector` adds a `vector` data type and vector distance operators. It is useful
for semantic search, recommendation, classification, and retrieval-augmented
generation.

### Project Files

Create this structure:

```text
postgres-vector/
├── compose.yaml
├── .env
└── postgres-init/
    └── 001_extensions.sql
```

### `compose.yaml`

```yaml
services:
  postgres:
    image: pgvector/pgvector:pg17
    container_name: postgres-vector
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - postgres_vector_data:/var/lib/postgresql/data
      - ./postgres-init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
    shm_size: 256mb

volumes:
  postgres_vector_data:
```

Pin the image to the PostgreSQL major version that you test. Before production,
you can also pin an exact image tag or digest.

### `.env`

```dotenv
POSTGRES_USER=app
POSTGRES_PASSWORD=replace_with_a_long_random_password
POSTGRES_DB=vectors
```

Do not commit a real production secret. A Docker secret or a secret manager is
safer for production.

### `postgres-init/001_extensions.sql`

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Initialization scripts run only when Docker creates an empty data volume. If
the volume already exists, run the extension command manually.

### Start and Verify

```bash
docker compose up -d
docker compose ps
docker compose logs postgres
```

Connect:

```bash
docker compose exec postgres psql -U app -d vectors
```

Verify:

```sql
SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

### Create Vector Data

The example uses three dimensions so that you can read the values. A real
embedding model can use hundreds or thousands of dimensions. The database
column dimension must match the model output dimension.

```sql
CREATE TABLE documents (
    document_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title text NOT NULL,
    body text NOT NULL,
    metadata jsonb NOT NULL DEFAULT '{}',
    embedding vector(3) NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now()
);

INSERT INTO documents (title, body, metadata, embedding) VALUES
    (
        'PostgreSQL indexes',
        'An index can make a selective query faster.',
        '{"topic":"database"}',
        '[0.10,0.80,0.20]'
    ),
    (
        'Docker volumes',
        'A volume keeps container data.',
        '{"topic":"containers"}',
        '[0.75,0.10,0.15]'
    ),
    (
        'SQL transactions',
        'A transaction groups related changes.',
        '{"topic":"database"}',
        '[0.15,0.70,0.25]'
    );
```

### Exact Nearest-Neighbor Search

L2 or Euclidean distance uses `<->`:

```sql
SELECT
    document_id,
    title,
    embedding <-> '[0.12,0.76,0.22]'::vector AS distance
FROM documents
ORDER BY embedding <-> '[0.12,0.76,0.22]'::vector
LIMIT 5;
```

Cosine distance uses `<=>`:

```sql
SELECT
    document_id,
    title,
    1 - (embedding <=> '[0.12,0.76,0.22]'::vector) AS cosine_similarity
FROM documents
ORDER BY embedding <=> '[0.12,0.76,0.22]'::vector
LIMIT 5;
```

Inner product uses `<#>`. The operator returns the negative inner product
because PostgreSQL indexes support ascending order scans:

```sql
SELECT
    document_id,
    title,
    (embedding <#> '[0.12,0.76,0.22]'::vector) * -1 AS inner_product
FROM documents
ORDER BY embedding <#> '[0.12,0.76,0.22]'::vector
LIMIT 5;
```

Select the same distance measure that your embedding model recommends.

### HNSW Index

HNSW gives fast approximate search and usually has a good speed-recall tradeoff.
It uses more memory and takes longer to build than IVFFlat.

```sql
CREATE INDEX documents_embedding_hnsw_idx
ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

The query operator must match the index operator class:

```sql
BEGIN;
SET LOCAL hnsw.ef_search = 100;

SELECT document_id, title
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
COMMIT;
```

Higher `hnsw.ef_search` usually gives better recall and slower search.

### IVFFlat Index

IVFFlat builds clusters. Add representative data before you create the index.

```sql
CREATE INDEX documents_embedding_ivfflat_idx
ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

At query time:

```sql
BEGIN;
SET LOCAL ivfflat.probes = 10;

SELECT document_id, title
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
COMMIT;
```

More probes usually improve recall and increase query time. Rebuild the index
when the data distribution changes significantly.

Do not keep HNSW and IVFFlat indexes for the same access path unless tests show
that both are necessary.

### Filtering Vector Search

Add a normal index for common metadata filters:

```sql
CREATE INDEX documents_topic_idx
ON documents ((metadata ->> 'topic'));

SELECT document_id, title
FROM documents
WHERE metadata ->> 'topic' = 'database'
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

Approximate indexes apply the filter after the index scan. A selective filter
can return fewer rows than requested. Current `pgvector` versions provide
iterative index scans to search more of the index when necessary. Test the
behavior with your installed extension version and real data.

For important filtered searches, consider:

- A B-tree or GIN index for the filter.
- Partial vector indexes for a small fixed set of categories.
- Partitioning for a small number of large categories.
- A higher HNSW search value or more IVFFlat probes.

### Hybrid Search

Hybrid search combines semantic similarity with PostgreSQL full-text search:

```sql
ALTER TABLE documents
ADD COLUMN search_document tsvector
GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B')
) STORED;

CREATE INDEX documents_search_gin_idx
ON documents USING gin (search_document);
```

Example weighted search:

```sql
WITH candidates AS (
    SELECT
        document_id,
        title,
        ts_rank(
            search_document,
            websearch_to_tsquery('english', $1)
        ) AS text_score,
        1 - (embedding <=> $2::vector) AS vector_score
    FROM documents
    WHERE search_document @@ websearch_to_tsquery('english', $1)
       OR document_id IN (
            SELECT document_id
            FROM documents
            ORDER BY embedding <=> $2::vector
            LIMIT 50
       )
)
SELECT
    document_id,
    title,
    (0.4 * text_score) + (0.6 * vector_score) AS combined_score
FROM candidates
ORDER BY combined_score DESC
LIMIT 10;
```

Raw text and vector scores can have different distributions. In a production
system, test score normalization or rank fusion with judged search results.

### Load Embeddings Safely

Use parameterized SQL. Do not build vector strings from untrusted input.

Application flow:

1. Split a source document into useful chunks.
2. Save model name, model version, source, and chunk position.
3. Create one embedding for each chunk.
4. Insert chunks in bounded batches.
5. Query with an embedding from the same model.
6. Re-embed all rows when a model change makes vectors incompatible.

Useful schema additions:

```sql
ALTER TABLE documents
    ADD COLUMN embedding_model text NOT NULL DEFAULT 'example-model',
    ADD COLUMN source_id text,
    ADD COLUMN chunk_number integer CHECK (chunk_number >= 0);

CREATE UNIQUE INDEX documents_source_chunk_uidx
    ON documents (source_id, chunk_number)
    WHERE source_id IS NOT NULL;
```

### Measure Vector Quality

Do not measure only latency. Measure:

- Recall at K against exact search.
- Precision or relevance with judged examples.
- P50, P95, and P99 latency.
- Index build time and index size.
- Insert and update cost.
- Result quality after metadata filters.

Use exact search as a baseline on a representative sample. Then tune HNSW or
IVFFlat until quality and latency meet the requirement.

### Back Up and Upgrade pgvector

`pg_dump` includes extension declarations and vector data. The target server
must have a compatible `pgvector` extension available.

Before an upgrade:

1. Read PostgreSQL and `pgvector` release notes.
2. Back up the database.
3. Test restore and application queries.
4. Test index rebuild time.
5. Upgrade a nonproduction environment first.

After you install a newer extension version:

```sql
ALTER EXTENSION vector UPDATE;
```

Check the active version:

```sql
SELECT extversion
FROM pg_extension
WHERE extname = 'vector';
```

### Stop or Remove the Docker Environment

Stop containers and keep data:

```bash
docker compose down
```

Remove containers and the database volume:

```bash
docker compose down --volumes
```

The second command deletes the database data. Make a backup first if you need
the data.

---

## Projects

## Project 1: Course Registration System

Build:

- Students, courses, instructors, lessons, and enrollments.
- Correct primary keys, foreign keys, and constraints.
- A transaction that enrolls a student without exceeding capacity.
- Reports that use joins, aggregates, and window functions.
- Indexes supported by query plans.

## Project 2: Multi-Tenant Task Application

Build:

- Organizations, users, projects, and tasks.
- Role-based privileges.
- Row-level security by organization.
- An audit event table with monthly partitions.
- Backup, restore, and migration procedures.

Test that one tenant cannot read or change another tenant's data.

## Project 3: Semantic Document Search

Build:

- Docker Compose with PostgreSQL and `pgvector`.
- Document chunk and embedding storage.
- Exact cosine search.
- An HNSW index.
- Metadata filters.
- Hybrid full-text and vector search.
- A benchmark for recall and latency.

## Expert Final Project

Prepare one project for production:

1. Write RPO and RTO requirements.
2. Make a data model review.
3. Load production-like data.
4. Record the main query plans.
5. Test concurrent traffic.
6. Set connection and statement limits.
7. Add monitoring and alerts.
8. Make and restore a backup.
9. Test one schema migration.
10. Write failure and recovery runbooks.

---

## Command Reference

### Database and Schema

```sql
CREATE DATABASE app_db;
DROP DATABASE app_db;
CREATE SCHEMA app;
DROP SCHEMA app CASCADE;
SET search_path TO app, public;
```

### Table

```sql
CREATE TABLE app.items (
    item_id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);

ALTER TABLE app.items ADD COLUMN price numeric(10, 2);
TRUNCATE TABLE app.items RESTART IDENTITY;
DROP TABLE app.items;
```

### Data

```sql
INSERT INTO app.items (name, price) VALUES ('Book', 10.00) RETURNING *;
SELECT * FROM app.items WHERE price > 5 ORDER BY price DESC;
UPDATE app.items SET price = 12.00 WHERE item_id = 1 RETURNING *;
DELETE FROM app.items WHERE item_id = 1 RETURNING *;
```

### Transaction

```sql
BEGIN;
SAVEPOINT step_one;
ROLLBACK TO SAVEPOINT step_one;
COMMIT;
-- Or use ROLLBACK for the complete transaction.
```

### Index and Plan

```sql
CREATE INDEX items_name_idx ON app.items (name);
REINDEX INDEX app.items_name_idx;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM app.items WHERE name = 'Book';
DROP INDEX app.items_name_idx;
```

### Maintenance

```sql
VACUUM (ANALYZE) app.items;
ANALYZE app.items;
CHECKPOINT;
```

Do not run manual `CHECKPOINT` as routine application work.

---

## Troubleshooting

### Connection Refused

Check:

```bash
docker compose ps
docker compose logs postgres
pg_isready -h localhost -p 5432
```

Confirm the host, port, container port mapping, and firewall.

### Password Authentication Failed

Confirm the role and password. Docker environment variables create the initial
database only when the data directory is empty. Changing `.env` does not change
an existing PostgreSQL role password.

Change it in SQL:

```sql
ALTER ROLE app WITH PASSWORD 'new_long_random_password';
```

### Relation Does Not Exist

Check the database, schema, spelling, and `search_path`:

```sql
SELECT current_database();
SHOW search_path;
SELECT to_regclass('learning.students');
```

### Permission Denied

Inspect role membership and object privileges:

```sql
\du
\dp learning.*

SELECT
    has_schema_privilege(current_user, 'learning', 'USAGE'),
    has_table_privilege(current_user, 'learning.students', 'SELECT');
```

### A Query Does Not Use an Index

Possible reasons:

- The table is small.
- The query returns a large part of the table.
- Statistics are old.
- The index column order does not match the query.
- A function or cast prevents use of the indexed expression.
- The planner correctly estimates that a sequential scan is cheaper.

Use `EXPLAIN (ANALYZE, BUFFERS)` and `ANALYZE`, then test with realistic data.

### Transactions Are Idle

An `idle in transaction` session can keep locks and old row versions:

```sql
SELECT pid, xact_start, state, query
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

Fix the application transaction boundary. Consider
`idle_in_transaction_session_timeout` as an additional safeguard.

### The Database Volume Uses Too Much Space

Check database, table, index, WAL, temporary file, and log growth. Do not delete
files directly from the PostgreSQL data directory.

For Docker:

```bash
docker system df -v
docker volume ls
```

### The Vector Extension Is Missing

Check available and installed extensions:

```sql
SELECT * FROM pg_available_extensions WHERE name = 'vector';
SELECT * FROM pg_extension WHERE extname = 'vector';
```

If `vector` is not available, use an image that includes `pgvector` or install
the extension package that matches the PostgreSQL major version.

### Vector Index Is Not Used

Check:

- The query has `ORDER BY` with a distance operator and `LIMIT`.
- The operator matches the index operator class.
- The query does not wrap the distance in an expression in `ORDER BY`.
- The table is large enough for approximate index use.
- Planner statistics are current.

Correct:

```sql
SELECT *
FROM documents
ORDER BY embedding <=> $1::vector
LIMIT 10;
```

An order such as `ORDER BY 1 - (embedding <=> $1::vector) DESC` can prevent the
expected index path.

---

## Next Steps

Use this study order:

1. Complete Lessons 1 through 6 and Project 1.
2. Complete Lessons 7 through 15.
3. Learn one application driver and parameterized queries.
4. Complete Lessons 16 through 21 with production-like data.
5. Complete Lessons 22 through 26.
6. Build the vector search project if your application needs semantic search.
7. Read the official manual for your installed PostgreSQL major version.

Primary references:

- PostgreSQL documentation: <https://www.postgresql.org/docs/>
- PostgreSQL version policy: <https://www.postgresql.org/support/versioning/>
- `psql` documentation: <https://www.postgresql.org/docs/current/app-psql.html>
- `pgvector` project: <https://github.com/pgvector/pgvector>
- PostgreSQL Docker image: <https://hub.docker.com/_/postgres>
