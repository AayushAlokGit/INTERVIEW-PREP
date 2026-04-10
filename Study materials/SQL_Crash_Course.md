# SQL Crash Course — Zero to Interview-Ready

> Tailored for Delve tech screen. All examples use a compliance platform domain.

---

## Level 0: Mental Model (MongoDB -> SQL)

| MongoDB | SQL/Postgres |
|---------|-------------|
| Collection | Table |
| Document | Row |
| Field | Column |
| Embedded document | Separate table + JOIN |
| `_id` (auto) | `id SERIAL PRIMARY KEY` |
| No schema | Strict schema (columns defined upfront) |
| `find({})` | `SELECT * FROM table` |
| `find({ name: "x" })` | `SELECT * FROM table WHERE name = 'x'` |
| `insertOne({...})` | `INSERT INTO table (...) VALUES (...)` |
| `updateOne(...)` | `UPDATE table SET ... WHERE ...` |
| `deleteOne(...)` | `DELETE FROM table WHERE ...` |

**The big shift:** In MongoDB you nest related data inside a document. In SQL you put related data in **separate tables** and connect them with **foreign keys** + **JOINs**.

---

## Level 1: Tables and Basic CRUD

### Creating tables
```sql
CREATE TABLE companies (
  id SERIAL PRIMARY KEY,        -- auto-incrementing integer
  name VARCHAR(255) NOT NULL,   -- string, max 255 chars
  industry VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  company_id INTEGER REFERENCES companies(id),  -- foreign key
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  role VARCHAR(50) DEFAULT 'member'
);
```

**Key data types:**
- `SERIAL` — auto-incrementing integer (for primary keys)
- `INTEGER`, `BIGINT` — numbers
- `VARCHAR(n)` — variable-length string up to n chars
- `TEXT` — unlimited string
- `BOOLEAN` — true/false
- `TIMESTAMP` / `TIMESTAMPTZ` — date + time (with/without timezone)
- `JSONB` — JSON data (Postgres-specific, very powerful)
- `UUID` — universally unique identifier

### INSERT — adding data
```sql
-- Single row
INSERT INTO companies (name, industry) VALUES ('Acme Corp', 'fintech');

-- Multiple rows
INSERT INTO companies (name, industry) VALUES
  ('Acme Corp', 'fintech'),
  ('Beta Inc', 'healthcare'),
  ('Gamma LLC', 'saas');

-- Insert and get the new row back (Postgres-specific)
INSERT INTO companies (name, industry)
VALUES ('Delta Co', 'compliance')
RETURNING id, name;
```

### SELECT — reading data
```sql
-- Everything
SELECT * FROM companies;

-- Specific columns
SELECT name, industry FROM companies;

-- With conditions
SELECT * FROM companies WHERE industry = 'fintech';

-- Multiple conditions
SELECT * FROM users WHERE role = 'admin' AND company_id = 1;
SELECT * FROM users WHERE role = 'admin' OR role = 'owner';

-- Pattern matching
SELECT * FROM companies WHERE name LIKE 'A%';    -- starts with A
SELECT * FROM companies WHERE name ILIKE '%corp%'; -- case-insensitive contains

-- IN clause (instead of multiple ORs)
SELECT * FROM users WHERE role IN ('admin', 'owner');

-- NULL checks (NOT = or !=, use IS NULL / IS NOT NULL)
SELECT * FROM users WHERE email IS NOT NULL;

-- Ordering
SELECT * FROM companies ORDER BY created_at DESC;

-- Limit and offset (pagination)
SELECT * FROM companies ORDER BY id LIMIT 10 OFFSET 20;  -- page 3

-- Aliasing
SELECT name AS company_name, industry AS sector FROM companies;
```

### UPDATE — modifying data
```sql
UPDATE users SET role = 'admin' WHERE email = 'aayush@acme.com';

-- Update multiple columns
UPDATE users SET role = 'admin', name = 'Aayush Alok' WHERE id = 1;

-- IMPORTANT: always use WHERE — without it you update ALL rows
UPDATE users SET role = 'member';  -- THIS UPDATES EVERY USER!
```

### DELETE — removing data
```sql
DELETE FROM users WHERE id = 5;

-- Same warning: always use WHERE
DELETE FROM users;  -- DELETES ALL USERS!
```

---

## Level 2: JOINs (the #1 thing they will test)

### Setup: Delve-like schema
```sql
-- Companies being monitored for compliance
CREATE TABLE companies (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL
);

-- Compliance frameworks (SOC 2, HIPAA, etc.)
CREATE TABLE frameworks (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL  -- 'SOC 2', 'HIPAA', 'GDPR'
);

-- Controls within a framework
CREATE TABLE controls (
  id SERIAL PRIMARY KEY,
  framework_id INTEGER REFERENCES frameworks(id),
  code VARCHAR(50) NOT NULL,     -- 'CC6.1', 'CC7.2'
  description TEXT
);

-- Evidence collected for each control
CREATE TABLE evidence (
  id SERIAL PRIMARY KEY,
  control_id INTEGER REFERENCES controls(id),
  company_id INTEGER REFERENCES companies(id),
  status VARCHAR(20) NOT NULL,  -- 'passing', 'failing', 'pending'
  collected_at TIMESTAMP DEFAULT NOW(),
  data JSONB
);
```

### INNER JOIN — only rows that match in BOTH tables
```sql
-- Get evidence with control names
SELECT e.id, c.code, c.description, e.status
FROM evidence e
INNER JOIN controls c ON e.control_id = c.id;
```
**Think of it as:** "Give me rows where there's a match on both sides."

If a control has no evidence, it won't appear. If evidence somehow has no matching control, it won't appear.

### LEFT JOIN — all rows from left table, matched or not
```sql
-- Get ALL controls, even ones with no evidence yet
SELECT c.code, c.description, e.status
FROM controls c
LEFT JOIN evidence e ON c.id = e.control_id;
```
**Think of it as:** "Give me every control. If there's evidence, include it. If not, fill with NULL."

Result:
| code | description | status |
|------|------------|--------|
| CC6.1 | Access control | passing |
| CC6.2 | Encryption | failing |
| CC7.1 | Monitoring | NULL | <-- no evidence yet

### RIGHT JOIN — all rows from right table (rarely used)
```sql
-- Same as LEFT JOIN but reversed — just swap table order instead
-- These two are equivalent:
SELECT * FROM controls c LEFT JOIN evidence e ON c.id = e.control_id;
SELECT * FROM evidence e RIGHT JOIN controls c ON c.id = e.control_id;
```

### FULL OUTER JOIN — all rows from both tables
```sql
SELECT c.code, e.status
FROM controls c
FULL OUTER JOIN evidence e ON c.id = e.control_id;
```
Rarely used in practice. Shows everything, NULLs where no match on either side.

### Multi-table JOINs (very common in interviews)
```sql
-- "Show me all evidence for SOC 2 controls for Acme Corp"
SELECT
  co.name AS company,
  f.name AS framework,
  c.code AS control,
  e.status,
  e.collected_at
FROM evidence e
INNER JOIN controls c ON e.control_id = c.id
INNER JOIN frameworks f ON c.framework_id = f.id
INNER JOIN companies co ON e.company_id = co.id
WHERE f.name = 'SOC 2'
  AND co.name = 'Acme Corp'
ORDER BY c.code;
```

### JOIN mental model
```
Table A          Table B
┌─────┐         ┌─────┐
│  1  │─────────│  1  │  <- matched: appears in INNER, LEFT, RIGHT, FULL
│  2  │─────────│  2  │  <- matched
│  3  │         │  4  │
└─────┘         └─────┘
  3 has no match on right -> appears in LEFT JOIN and FULL JOIN (B cols = NULL)
  4 has no match on left  -> appears in RIGHT JOIN and FULL JOIN (A cols = NULL)
```

---

## Level 3: Aggregations + GROUP BY

### COUNT, SUM, AVG, MIN, MAX
```sql
-- How many evidence records total?
SELECT COUNT(*) FROM evidence;

-- How many per status?
SELECT status, COUNT(*) AS count
FROM evidence
GROUP BY status;
-- Result:
-- passing  | 42
-- failing  | 7
-- pending  | 15

-- How many passing controls per company?
SELECT co.name, COUNT(*) AS passing_count
FROM evidence e
INNER JOIN companies co ON e.company_id = co.id
WHERE e.status = 'passing'
GROUP BY co.name
ORDER BY passing_count DESC;
```

### FILTER — conditional aggregation (Postgres-specific, very powerful)

`FILTER (WHERE ...)` lets you apply a condition **inside** an aggregate function. This lets you compute multiple aggregations in a single query without multiple JOINs or subqueries.

It works with **all aggregate functions** — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

```sql
-- Count passing and failing in one query (without FILTER you'd need two subqueries)
SELECT
  company_id,
  COUNT(*)                              AS total,
  COUNT(*) FILTER (WHERE status = 'passing') AS passing,
  COUNT(*) FILTER (WHERE status = 'failing') AS failing,
  COUNT(*) FILTER (WHERE status = 'pending') AS pending
FROM evidence
GROUP BY company_id;

-- SUM with FILTER — total amount for completed vs refunded orders
SELECT
  SUM(amount)                                  AS total_revenue,
  SUM(amount) FILTER (WHERE status = 'paid')   AS paid_revenue,
  SUM(amount) FILTER (WHERE status = 'refund') AS refunded
FROM orders
GROUP BY company_id;

-- AVG with FILTER — average score only for passing controls
SELECT
  company_id,
  AVG(score)                                    AS avg_all,
  AVG(score) FILTER (WHERE status = 'passing')  AS avg_passing
FROM evidence
GROUP BY company_id;

-- MIN/MAX with FILTER — earliest failing evidence per company
SELECT
  company_id,
  MIN(collected_at) FILTER (WHERE status = 'failing') AS first_failure,
  MAX(collected_at) FILTER (WHERE status = 'passing') AS last_pass
FROM evidence
GROUP BY company_id;
```

**FILTER vs WHERE:**
- `WHERE status = 'passing'` — filters rows before grouping, so you only have passing rows to work with
- `FILTER (WHERE status = 'passing')` inside COUNT — keeps all rows for other aggregates, but counts only passing for that specific column

```sql
-- These are NOT the same:

-- Only counts passing (WHERE removes all other rows first)
SELECT company_id, COUNT(*) AS count
FROM evidence
WHERE status = 'passing'
GROUP BY company_id;

-- Counts both total and passing in one pass (FILTER is per-aggregate)
SELECT company_id,
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE status = 'passing') AS passing
FROM evidence
GROUP BY company_id;
```

### GROUP BY rules
1. Every column in SELECT must either be in GROUP BY or wrapped in an aggregate function
2. WHERE filters rows BEFORE grouping
3. HAVING filters groups AFTER grouping

```sql
-- WRONG: framework_id is not in GROUP BY and not aggregated
SELECT framework_id, status, COUNT(*)
FROM evidence
GROUP BY status;

-- RIGHT:
SELECT framework_id, status, COUNT(*)
FROM evidence
GROUP BY framework_id, status;
```

### WHERE vs HAVING
```sql
-- WHERE: filter individual rows before grouping
-- "Count failing evidence, but only for SOC 2 controls"
SELECT c.code, COUNT(*) AS fail_count
FROM evidence e
JOIN controls c ON e.control_id = c.id
JOIN frameworks f ON c.framework_id = f.id
WHERE e.status = 'failing'
  AND f.name = 'SOC 2'
GROUP BY c.code;

-- HAVING: filter groups after aggregation
-- "Show controls that have more than 3 failing evidence records"
SELECT c.code, COUNT(*) AS fail_count
FROM evidence e
JOIN controls c ON e.control_id = c.id
WHERE e.status = 'failing'
GROUP BY c.code
HAVING COUNT(*) > 3;
```

### Execution order (critical to understand)
```
FROM      -> Which tables?
JOIN      -> Combine tables
WHERE     -> Filter individual rows (removes rows before grouping)
GROUP BY  -> Form groups from remaining rows
            └── FILTER (WHERE ...) runs here — per aggregate, on rows within each group
            └── Aggregates (COUNT, SUM, AVG...) compute on those filtered subsets
            └── Group collapses into one output row
HAVING    -> Filter groups based on aggregate results
SELECT    -> Pick columns / compute values
ORDER BY  -> Sort results
LIMIT     -> Cap number of results
```

Key implications:
- `WHERE` removes rows before grouping — affects ALL aggregates
- `FILTER` removes rows per aggregate, after grouping — other aggregates still see the full group
- `HAVING` filters after aggregates are computed — this is why `HAVING COUNT(*) > 1` works but `WHERE COUNT(*) > 1` doesn't
- You can't use a SELECT alias in WHERE or HAVING — both run before SELECT

---

## Level 4: Subqueries

### Subquery in WHERE
```sql
-- Companies that have at least one failing control
SELECT name FROM companies
WHERE id IN (
  SELECT company_id FROM evidence WHERE status = 'failing'
);

-- Controls with more evidence than average
SELECT code FROM controls
WHERE id IN (
  SELECT control_id FROM evidence
  GROUP BY control_id
  HAVING COUNT(*) > (SELECT AVG(cnt) FROM (
    SELECT COUNT(*) AS cnt FROM evidence GROUP BY control_id
  ) sub)
);
```

### Subquery in FROM (derived table)
```sql
-- Get compliance score per company (% passing)
SELECT
  sub.company_name,
  sub.total_checks,
  sub.passing_checks,
  ROUND(sub.passing_checks * 100.0 / sub.total_checks, 1) AS compliance_pct
FROM (
  SELECT
    co.name AS company_name,
    COUNT(*) AS total_checks,
    COUNT(*) FILTER (WHERE e.status = 'passing') AS passing_checks
  FROM evidence e
  JOIN companies co ON e.company_id = co.id
  GROUP BY co.name
) sub
ORDER BY compliance_pct DESC;
```

### EXISTS (more efficient than IN for large datasets)
```sql
-- Companies that have at least one failing control
SELECT co.name
FROM companies co
WHERE EXISTS (
  SELECT 1 FROM evidence e
  WHERE e.company_id = co.id AND e.status = 'failing'
);
```

---

## Level 5: Advanced Postgres Features

### CTEs (Common Table Expressions) — readable subqueries
```sql
-- Same compliance score query, but readable
WITH company_stats AS (
  SELECT
    co.name AS company_name,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE e.status = 'passing') AS passing
  FROM evidence e
  JOIN companies co ON e.company_id = co.id
  GROUP BY co.name
)
SELECT
  company_name,
  total,
  passing,
  ROUND(passing * 100.0 / total, 1) AS compliance_pct
FROM company_stats
ORDER BY compliance_pct DESC;
```
**CTE vs Subquery:** Same result, but CTEs are named and can be referenced multiple times. Use CTEs in interviews — they're cleaner and show maturity.

### Window Functions (differentiator in interviews)

Window functions compute a value **across a set of rows related to the current row** — without collapsing them like GROUP BY does. They're written with `OVER (...)`.

#### GROUP BY vs Window Functions — the key difference

```sql
-- GROUP BY: collapses rows — one row per group
SELECT company_id, COUNT(*) AS total
FROM evidence
GROUP BY company_id;
-- Result: 3 rows (one per company), original rows are gone

-- Window function: keeps all rows, attaches the group calculation to each
SELECT id, company_id, status,
  COUNT(*) OVER (PARTITION BY company_id) AS company_total
FROM evidence;
-- Result: all rows, each knows its company's total
-- id | company_id | status  | company_total
-- 1  | 1          | passing | 3
-- 2  | 1          | failing | 3   <- same company, same total
-- 3  | 1          | pending | 3
-- 4  | 2          | passing | 2
-- 5  | 2          | failing | 2
```

#### PARTITION BY — how to define the group

`PARTITION BY` splits the rows into groups. The window function runs independently within each group.

```sql
-- Without PARTITION BY: one big group (entire table)
SELECT id, COUNT(*) OVER () AS total_records FROM evidence;

-- With PARTITION BY: grouped by company
SELECT id, company_id,
  COUNT(*) OVER (PARTITION BY company_id) AS per_company_total
FROM evidence;

-- With PARTITION BY + ORDER BY: running total within each company
SELECT id, company_id, collected_at,
  COUNT(*) OVER (PARTITION BY company_id ORDER BY collected_at) AS running_count
FROM evidence;
```

#### ROW_NUMBER() — the most interview-critical window function

Assigns a sequential number (1, 2, 3...) to each row **within its partition**, ordered by whatever you specify.

```sql
SELECT
  id,
  control_id,
  status,
  collected_at,
  ROW_NUMBER() OVER (PARTITION BY control_id ORDER BY collected_at DESC) AS rn
FROM evidence;

-- Result:
-- id | control_id | status  | collected_at | rn
-- 5  | 10         | passing | 2025-03-01   | 1   <- newest for control 10
-- 2  | 10         | failing | 2025-01-15   | 2
-- 1  | 10         | passing | 2024-12-01   | 3
-- 4  | 11         | failing | 2025-02-20   | 1   <- newest for control 11
-- 3  | 11         | passing | 2025-01-10   | 2
```

**The "latest per group" pattern — memorise this:**
```sql
-- Get the most recent evidence for each control
SELECT * FROM (
  SELECT
    e.*,
    ROW_NUMBER() OVER (PARTITION BY control_id ORDER BY collected_at DESC) AS rn
  FROM evidence e
  WHERE company_id = 1
) sub
WHERE rn = 1;
```

Why the subquery? You can't filter on `rn` directly — window functions are computed after WHERE. So you wrap the whole thing and filter on the outside.

#### ROW_NUMBER vs RANK vs DENSE_RANK

These differ only when there are **ties** (two rows with the same ORDER BY value):

```
Scores: 100, 90, 90, 80

ROW_NUMBER:  1, 2, 3, 4   -- always unique, ties broken arbitrarily
RANK:        1, 2, 2, 4   -- ties get same rank, next rank skips (gap)
DENSE_RANK:  1, 2, 2, 3   -- ties get same rank, no gap
```

```sql
-- Use ROW_NUMBER when you need exactly one row per group (e.g. latest per control)
-- Use RANK/DENSE_RANK when you want to expose ties (e.g. top 3 companies by score)
```

**RANK example — rank companies by number of failing controls:**
```sql
SELECT
  co.name,
  COUNT(*) AS fail_count,
  RANK() OVER (ORDER BY COUNT(*) DESC) AS rank
FROM evidence e
JOIN companies co ON e.company_id = co.id
WHERE e.status = 'failing'
GROUP BY co.name;

-- Result:
-- name       | fail_count | rank
-- Acme Corp  | 5          | 1
-- Beta Inc   | 3          | 2
-- Gamma LLC  | 3          | 2    <- tie, both get rank 2
-- Delta Co   | 1          | 4    <- rank 3 is skipped (gap)
```

**DENSE_RANK example — same query, no gaps:**
```sql
SELECT
  co.name,
  COUNT(*) AS fail_count,
  DENSE_RANK() OVER (ORDER BY COUNT(*) DESC) AS rank
FROM evidence e
JOIN companies co ON e.company_id = co.id
WHERE e.status = 'failing'
GROUP BY co.name;

-- Result:
-- name       | fail_count | rank
-- Acme Corp  | 5          | 1
-- Beta Inc   | 3          | 2
-- Gamma LLC  | 3          | 2    <- tie, both get rank 2
-- Delta Co   | 1          | 3    <- rank 3, no gap
```

#### LAG and LEAD — look at adjacent rows

```sql
-- Compare each evidence record to the previous one for the same control
SELECT
  id,
  control_id,
  status,
  collected_at,
  LAG(status) OVER (PARTITION BY control_id ORDER BY collected_at) AS prev_status
FROM evidence;

-- Result:
-- id | control_id | status  | prev_status
-- 1  | 10         | passing | NULL        <- no previous row
-- 2  | 10         | failing | passing     <- previous was passing
-- 5  | 10         | passing | failing     <- previous was failing (regression then recovery)

-- Detect regressions: controls that went from passing to failing
SELECT * FROM (
  SELECT
    control_id,
    status,
    LAG(status) OVER (PARTITION BY control_id ORDER BY collected_at) AS prev_status
  FROM evidence
) sub
WHERE status = 'failing' AND prev_status = 'passing';
```

**Key window functions summary:**
- `ROW_NUMBER()` — unique sequential number per partition (use for "latest per group")
- `RANK()` — rank with gaps (1, 2, 2, 4)
- `DENSE_RANK()` — rank without gaps (1, 2, 2, 3)
- `LAG(col)` — value from the previous row in the partition
- `LEAD(col)` — value from the next row in the partition
- `SUM/COUNT/AVG OVER (...)` — running totals or group stats without collapsing rows

### UPSERT (INSERT ... ON CONFLICT)
```sql
-- Insert new evidence, or update status if it already exists for that control+company
INSERT INTO evidence (control_id, company_id, status, collected_at)
VALUES (1, 1, 'passing', NOW())
ON CONFLICT (control_id, company_id)
DO UPDATE SET
  status = EXCLUDED.status,
  collected_at = EXCLUDED.collected_at;
```

### JSONB (Postgres's superpower)
```sql
-- Store flexible metadata
INSERT INTO evidence (control_id, company_id, status, data)
VALUES (1, 1, 'passing', '{"source": "aws", "screenshot_url": "/s3/img1.png", "auto_remediated": true}');

-- Query JSON fields
SELECT * FROM evidence WHERE data->>'source' = 'aws';
SELECT * FROM evidence WHERE (data->>'auto_remediated')::boolean = true;

-- Get a nested value
SELECT data->'details'->>'message' FROM evidence;

-- Check if key exists
SELECT * FROM evidence WHERE data ? 'screenshot_url';
```

`->` returns JSON, `->>` returns text. Use `->>` for comparisons.

### RETURNING
```sql
-- Get the row you just inserted/updated/deleted
INSERT INTO companies (name) VALUES ('NewCo') RETURNING *;
UPDATE evidence SET status = 'passing' WHERE id = 5 RETURNING id, status;
DELETE FROM evidence WHERE collected_at < '2025-01-01' RETURNING id;
```

---

## Level 6: Schema Design Principles

### Normalization (what interviewers want to hear)
- **1NF:** No repeating groups. Each cell has one value.
- **2NF:** No partial dependencies. Every non-key column depends on the WHOLE primary key.
- **3NF:** No transitive dependencies. Non-key columns don't depend on other non-key columns.

**In plain English:** Don't duplicate data. If a company name changes, you should update it in ONE place, not 500 evidence rows.

### Foreign keys and relationships
```
One-to-Many:  One company has many users
              companies.id <- users.company_id

Many-to-Many: A company can have many frameworks, a framework can have many companies
              companies <-> company_frameworks <-> frameworks
              (needs a junction/join table)

One-to-One:   One company has one billing_config
              companies.id <- billing_configs.company_id (UNIQUE)
```

**Junction table example:**
```sql
CREATE TABLE company_frameworks (
  company_id INTEGER REFERENCES companies(id),
  framework_id INTEGER REFERENCES frameworks(id),
  started_at TIMESTAMP DEFAULT NOW(),
  PRIMARY KEY (company_id, framework_id)  -- composite primary key
);
```

### Indexes — when and why
```sql
-- Create index on columns you frequently filter/join on
CREATE INDEX idx_evidence_company ON evidence(company_id);
CREATE INDEX idx_evidence_status ON evidence(status);

-- Composite index (order matters — leftmost column first)
CREATE INDEX idx_evidence_company_status ON evidence(company_id, status);
-- This helps: WHERE company_id = 1 AND status = 'failing'
-- This helps: WHERE company_id = 1
-- This does NOT help: WHERE status = 'failing' (need separate index)

-- Unique index (enforces uniqueness)
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

**Rules of thumb:**
- Always index foreign keys
- Index columns used in WHERE and JOIN ON
- Don't over-index — each index slows down INSERT/UPDATE
- Composite index: put the more selective column first

### Transactions
```sql
BEGIN;
  -- Deactivate old evidence
  UPDATE evidence SET status = 'superseded' WHERE control_id = 1 AND company_id = 1;
  -- Insert new evidence
  INSERT INTO evidence (control_id, company_id, status) VALUES (1, 1, 'passing');
COMMIT;
-- If anything fails between BEGIN and COMMIT, nothing is applied
-- Use ROLLBACK to manually abort
```

---

## Quick Reference: SQL Patterns You'll Use in Interviews

### Pattern 1: "Find items that DON'T have a match"
```sql
-- Controls with NO evidence
SELECT c.* FROM controls c
LEFT JOIN evidence e ON c.id = e.control_id
WHERE e.id IS NULL;
```

### Pattern 2: "Find the latest/first of each group"
```sql
-- Latest evidence per control
SELECT * FROM (
  SELECT e.*, ROW_NUMBER() OVER (PARTITION BY control_id ORDER BY collected_at DESC) AS rn
  FROM evidence e
) sub WHERE rn = 1;
```

### Pattern 3: "Percentage / ratio"
```sql
SELECT
  company_id,
  COUNT(*) FILTER (WHERE status = 'passing') * 100.0 / COUNT(*) AS pass_rate
FROM evidence
GROUP BY company_id;
```

### Pattern 4: "Top N per group"
```sql
-- Top 3 most failing controls per framework
SELECT * FROM (
  SELECT
    f.name AS framework,
    c.code,
    COUNT(*) AS fail_count,
    ROW_NUMBER() OVER (PARTITION BY f.id ORDER BY COUNT(*) DESC) AS rn
  FROM evidence e
  JOIN controls c ON e.control_id = c.id
  JOIN frameworks f ON c.framework_id = f.id
  WHERE e.status = 'failing'
  GROUP BY f.id, f.name, c.code
) sub WHERE rn <= 3;
```

### Pattern 5: "Running total / cumulative"
```sql
SELECT
  collected_at::date AS day,
  COUNT(*) AS daily_count,
  SUM(COUNT(*)) OVER (ORDER BY collected_at::date) AS running_total
FROM evidence
GROUP BY collected_at::date;
```

---

## Practice Problems (do these before your screen)

**Easy:**
1. List all controls for the 'SOC 2' framework
2. Count how many evidence records exist per status
3. Find all companies that have no evidence records at all

**Medium:**
4. For each company, show the number of passing and failing controls
5. Find the framework with the highest overall pass rate
6. List controls that have changed status (had both 'passing' and 'failing' evidence)

**Hard:**
7. For each company + framework combination, calculate the compliance percentage
8. Find the most recent evidence for every control for a given company
9. Rank companies by their overall compliance score across all frameworks
10. Find controls that went from 'passing' to 'failing' (regression) — compare the two most recent evidence records

---

## Interview Tips

1. **Talk while you write.** Say "I'll start with a JOIN between evidence and controls..." before writing it.
2. **Start simple, then refine.** Write a basic SELECT first, then add JOINs, then GROUP BY.
3. **Use aliases.** `FROM evidence e` not `FROM evidence`. It's faster and shows fluency.
4. **Use CTEs over nested subqueries.** Cleaner, easier to debug, interviewers love them.
5. **Name your aggregates.** `COUNT(*) AS fail_count` not just `COUNT(*)`.
6. **Ask about indexes AFTER writing the query.** "I'd add an index on evidence(company_id, status) for this query" — shows you think about performance.
7. **Mention trade-offs.** "I could denormalize this for read speed, but it'd make updates harder."