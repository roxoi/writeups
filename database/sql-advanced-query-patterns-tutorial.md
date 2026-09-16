---
title: "SQL Interview Handbook: From MariaDB Setup to Advanced Query Patterns"
description: "A practical, interview-focused guide to SQL — covering MariaDB setup, query execution order, joins, conditional aggregation, window functions, CTEs, set operations, and high-ROI patterns like gaps & islands and Top-N per group, with worked examples and solutions."
author: [{"name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"}]
thumbnail: "/images/sql-interview-handbook.png"
tags: [SQL, MariaDB, MySQL, SDE-Interview, DBMS, Window-Functions, Query-Optimization]
keywords: ["SQL interview questions", "SQL window functions tutorial", "SQL joins interview questions", "CTE recursive SQL examples", "gaps and islands SQL", "Top N per group SQL", "conditional aggregation SQL CASE WHEN", "MariaDB Docker setup", "SQL execution order interview"]
---

# SQL Interview Handbook: From MariaDB Setup to Advanced Query Patterns

![SQL Interview Handbook](/images/sql-interview-handbook.png)


## Installing mysql(maria db lighter image (70mb))

```bash
# 1. Pull the absolute minimum, ultra-lightweight MariaDB image built on Alpine Linux
docker pull mariadb:lts
# 2. Start the container in the background
#    --name maria-db            : Sets the container name to 'maria-db'
#    -e MYSQL_ROOT_PASSWORD=... : Sets your custom root password ('raje@usr')
#    -v ~/mariadb_data:...      : Persists your database tables to your local Ubuntu home directory
#    -p 3306:3306               : Maps port 3306 so external apps can connect
#    -d                         : Runs the container detached in the background
docker run --name maria-db \
  -e MARIADB_ROOT_PASSWORD=raje@usr \
  -v ~/mariadb_data:/var/lib/mysql \
  -p 3306:3306 \
  -d mariadb:lts

# 3. Log into the MariaDB shell interactively
docker exec -it maria-db mariadb -u root -p

# create user
CREATE USER 'rajeusr'@'%' IDENTIFIED BY 'raje@usr';
GRANT ALL PRIVILEGES ON *.* TO 'rajeusr'@'%';
FLUSH PRIVILEGES;
exit
docker exec -it maria-db mariadb -u rajeusr -p
```
1. `docker exec -it maria-db` this part tell running command inside container
2. `mariadb -u root -p`: command thats run in container


```bash
docker start maria-db # start docker container (maria-db)
docker stop maria-db # stop docker container (maria-db)

## Delete history inside container
find / -name ".*history" -exec rm -f {} \; 2>/dev/null # delete all container history (only run inside the container not the host machine)
#after this restart the container
docker exec -it maria-db bash
mariadb -u rajeusr -p

```
## Execution order
SQL Query Execution Order
### 💡 "FWG-HSD-OL" (याद रखने की धांसू लाइन)
इसको एक मजेदार इंग्लिश सेंटेंस से याद रखें:
FwG HsD OL ❌ (याद रखने में मुश्किल)
👉 **Fred Wants George Harry’s Smart Dog On Leash.**
*"फ्रेड (Fred) चाहता है कि जॉर्ज हैरी का समझदार कुत्ता पट्टे (Leash) पर रहे।"*

| Letter | SQL Clause | मतलब (शॉर्ट में) |
|---|---|---|
| 🐕 Fred | FROM / JOIN | सबसे पहले टेबल ढूंढो और जोड़ो। |
| 🐕 Wants | WHERE | कच्चा (raw) डेटा फ़िल्टर करो। |
| 🐕 George | GROUP BY | डेटा की बाल्टियां (groups) बनाओ। |
| 🐕 Harry's | HAVING | बनी हुई बाल्टियों को फ़िल्टर करो। |
| 🐕 Smart | SELECT | अब कॉलम चुनो (और Alias बनाओ)। |
| 🐕 Dog | DISTINCT | डुप्लिकेट्स को बाहर फेंको। |
| 🐕 On | ORDER BY | डेटा को सीधा/उल्टा सॉर्ट करो। |
| 🐕 Leash | LIMIT / OFFSET | जितने रो (rows) चाहिए, उतने काटो। |

### 🧠 (Story Method)
अगर रटना नहीं है, तो सोचो कि डेटाबेस एक शेफ (Chef) है जो खाना बना रहा है:
   1. FROM: सबसे पहले शेफ किचन (Table) में जाता है सामान लेने।
   2. WHERE: खराब सब्जियां पहले ही बाहर फेंक देता है।
   3. GROUP BY: बची सब्जियों को काटकर अलग-अलग कटोरी में रखता है (प्याज अलग, टमाटर अलग)।
   4. HAVING: जिस कटोरी में कम सब्जी है, उस पूरी कटोरी को हटा देता है।
   5. SELECT: अब वह डिश को प्लेट में सजाता है (कॉलम चुनता है)।
   6. DISTINCT: अगर गलती से एक ही जैसी दो प्लेट बन गईं, तो एक हटा देता है।
   7. ORDER BY: प्लेट्स को टेबल पर लाइन से (सजाकर) रखता है।
   8. LIMIT: मेहमान को सिर्फ शुरुआत की 2 प्लेट सर्व करता है।


## Core Query Building Blocks (Theory)
These are the absolute foundations. In competitive programming / company coding rounds you must write them correctly and fast under pressure.

### 1. SELECT + Filtering (`WHERE`) + Sorting (`ORDER BY`) + Limiting

**Logical execution order** (very important for interviews):

```sql
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT/OFFSET
```

- **`WHERE`**: Filters **rows** before any grouping or aggregation.
- **`ORDER BY`**: Sorts the final result set. You can use column names, aliases, or expressions.
- **`LIMIT` / `TOP` / `FETCH`**: Restricts the number of rows returned.
  - MySQL / PostgreSQL / SQLite: `LIMIT n`
  - SQL Server: `TOP n` or `OFFSET ... FETCH`
  - Oracle: `FETCH FIRST n ROWS ONLY`

**Common patterns**:
```sql
SELECT column1, column2
FROM table_name
WHERE condition1 AND condition2
ORDER BY column1 DESC, column2 ASC
LIMIT 10;
```

**Key points**:
- `WHERE` cannot use column aliases created in the same `SELECT`.
- `ORDER BY` **can** use aliases.
- Multiple columns in `ORDER BY` → primary sort, then secondary sort, etc.
- `NULLS FIRST` / `NULLS LAST` (PostgreSQL) is sometimes asked.

### 2. DISTINCT

`DISTINCT` removes duplicate **rows** from the result set.

```sql
SELECT DISTINCT column1, column2
FROM table_name;
```

**Important theory**:
- `DISTINCT` applies to the **entire row** (combination of all selected columns).
- `SELECT DISTINCT col1` is different from `SELECT DISTINCT col1, col2`.
- Performance: `DISTINCT` usually forces a sort or hash aggregate → can be expensive on large data.
- Prefer `GROUP BY` when you also need aggregations.

**Common interview trick**:
```sql
-- This removes duplicate (department, salary) pairs
SELECT DISTINCT department, salary FROM employees;
```

### 3. CASE WHEN (Extremely Important)

`CASE` is SQL’s way of writing conditional logic (if-else).

**Two forms**:

**Simple CASE**:
```sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ELSE default_result
END
```

**Searched CASE** (more powerful and commonly used):
```sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ELSE default_result
END
```

**Most important use in interviews → Conditional Aggregation**

```sql
SELECT
    department,
    COUNT(*) AS total_employees,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count,
    AVG(CASE WHEN salary > 50000 THEN salary END) AS avg_high_salary
FROM employees
GROUP BY department;
```

**Key points**:
- `CASE` returns `NULL` if no condition matches and there is no `ELSE`.
- You can use `CASE` almost anywhere: `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, aggregate functions.
- Conditional aggregation with `SUM(CASE ...)` or `COUNT(CASE ...)` is one of the highest-frequency patterns in medium/hard problems.

### 4. NULL Handling (`COALESCE`, `NULLIF`, `IFNULL`)

`NULL` is **not** a value — it means “unknown”. Almost every advanced problem involves proper NULL handling.

| Function          | Description                                      | Example |
|-------------------|--------------------------------------------------|--------|
| `COALESCE(a, b, c, ...)` | Returns the **first non-NULL** value            | `COALESCE(salary, 0)` |
| `NULLIF(a, b)`    | Returns `NULL` if `a = b`, otherwise returns `a` | `NULLIF(salary, 0)` |
| `IFNULL(a, b)`    | MySQL only. Same as `COALESCE(a, b)`             | `IFNULL(salary, 0)` |
| `IS NULL` / `IS NOT NULL` | The **only** correct way to check for NULL     | `WHERE manager_id IS NULL` |

**Critical rules**:
- Any arithmetic operation with `NULL` → result is `NULL` (`5 + NULL = NULL`).
- `NULL = NULL` is **unknown** (not true). Always use `IS NULL`.
- `NOT IN (..., NULL, ...)` almost always returns empty result (dangerous).
- Prefer `COALESCE` over `IFNULL` for portability.

**Common patterns**:
```sql
-- Replace NULL with 0
SELECT COALESCE(bonus, 0) AS bonus FROM employees;

-- Safe division
SELECT revenue / NULLIF(quantity, 0) AS avg_price;
```

### 5. String & Date Functions (Common in Real Problems)

You don’t need every function — only the ones that appear frequently in coding rounds.

#### String Functions
| Function              | Purpose                          | Example |
|-----------------------|----------------------------------|--------|
| `CONCAT(a, b, ...)` / `\|\|` | Concatenate strings             | `CONCAT(first_name, ' ', last_name)` |
| `LENGTH()` / `CHAR_LENGTH()` | Length of string                | |
| `UPPER()` / `LOWER()` | Case conversion                 | |
| `SUBSTRING()` / `SUBSTR()` | Extract part of string          | `SUBSTRING(email, 1, 5)` |
| `TRIM()`, `LTRIM()`, `RTRIM()` | Remove spaces                   | |
| `REPLACE(str, from, to)` | Replace substring               | |
| `LEFT()` / `RIGHT()`  | First/last n characters         | |
| `POSITION()` / `INSTR()` | Find position of substring      | |

#### Date / Time Functions
| Function                  | Purpose                              | Example |
|---------------------------|--------------------------------------|--------|
| `CURRENT_DATE` / `NOW()`  | Current date/time                    | |
| `EXTRACT(YEAR FROM date)` | Extract part of date                 | `EXTRACT(YEAR FROM order_date)` |
| `DATE_TRUNC()` (Postgres) | Truncate to year/month/day           | |
| `DATE_ADD()` / `DATE_SUB()` / `+ INTERVAL` | Add/subtract time                | |
| `DATEDIFF()` / `AGE()`    | Difference between dates             | |
| `TO_CHAR()` / `DATE_FORMAT()` | Format date as string             | |

**Very common interview patterns**:
```sql
-- Year and Month
SELECT EXTRACT(YEAR FROM order_date) AS year,
       EXTRACT(MONTH FROM order_date) AS month
FROM orders;

-- Difference in days
SELECT DATEDIFF(end_date, start_date) AS duration;

-- Truncate to month (PostgreSQL)
SELECT DATE_TRUNC('month', order_date) AS month_start;
```

### Quick Summary – What Interviewers Expect

1. You write clean `SELECT ... WHERE ... ORDER BY ... LIMIT` instantly.
2. You know when to use `DISTINCT` vs `GROUP BY`.
3. You can write conditional aggregation using `CASE` without thinking.
4. You never forget about `NULL` (use `COALESCE`, `IS NULL`, `NULLIF`).
5. You are comfortable with basic string and date manipulations.

## REGEX
Here is a complete setup with a Dummy Table (RawData) and live examples to master advanced wildcards and string functions just like they are tested in corporate interviews.
### 🏛️ The Dummy Data Setup
Imagine an interviewer gives you this messy RawData table containing user logs:

| user_id | profile_name | corporate_email | file_path |
|---|---|---|---|
| 1 | aman Kumar | aman.k@oracle.com | /usr/bin/docs/resume.pdf |
| 2 | ROHIT sharma | rohit123@nitp.ac.in | /downloads/image.PNG |
| 3 | priya_roy | priya@gmail.com | /home/user/project.tar.gz |
| 4 | 99rahul | rahul_99@oracle.co.in | /var/log/sys.txt |
| 5 | Amit | amit_kumar@nitp.ac.in | /root/notes |

### 🚀 1. Advanced Wildcard (REGEXP) Examples## Scenario A: Find valid corporate emails only
Goal: Filter rows where the email ends with strictly @oracle.com or @nitp.ac.in, and the username before @ starts with a letter.

```sql (mysql/mariadb)
SELECT user_id, profile_name, corporate_email 
FROM users 
WHERE corporate_email REGEXP '^[a-zA-Z].*@(nitp\.ac\.in|oracle\.com)$';
```

```sql (psql)
SELECT user_id, profile_name, corporate_email 
FROM users
WHERE corporate_email ~ '^[a-zA-Z].*@(nitp\.ac\.in|oracle\.com)$';
```

* Output Result:

| profile_name | corporate_email |
|---|---|
| aman Kumar | aman.k@oracle.com |
| rohit123@nitp.ac.in | |
| Amit | amit_kumar@nitp.ac.in |

* Why row 4 (rahul_99@oracle.co.in) failed: It ends in .co.in, not .com.
* Why row 3 failed: It is a personal gmail.com address.

### Scenario B: Catch invalid usernames
Goal: Find profiles that start with a number or contain special characters like underscores (violating normal system naming rules).

SELECT user_id, profile_name FROM RawData WHERE profile_name REGEXP '^[0-9]' OR profile_name LIKE '%\_%';


* Output Result:

| user_id | profile_name |
|---|---|
| 3 | priya_roy |
| 4 | 99rahul |


### ✂️ 2. Advanced String Functions Examples## Scenario C: Clean and Standardize Names (Capitalization)
Goal: Format names nicely by forcing the first letter to be uppercase and the remaining letters lowercase.
(Let's take the first word of profile_name before any space using SUBSTRING_INDEX or basic positioning).

SELECT profile_name,
       CONCAT(
           UPPER(LEFT(SUBSTRING_INDEX(profile_name, ' ', 1), 1)), 
           LOWER(SUBSTRING(SUBSTRING_INDEX(profile_name, ' ', 1), 2))
       ) AS cleaned_first_nameFROM RawData;


* Output Result:

| profile_name | cleaned_first_name |
|---|---|
| aman Kumar | Aman |
| ROHIT sharma | Rohit |
| priya_roy | Priya_roy |


### Scenario D: Dynamic Extension Extraction
Goal: Extract only the file extension (everything after the very last dot .) from the file_path. If there is no file extension, show 'No Extension'.

SELECT file_path, 
       CASE 
           WHEN file_path LIKE '%.%' THEN LOWER(SUBSTRING_INDEX(file_path, '.', -1))
           ELSE 'No Extension'
       END AS file_extensionFROM RawData;


* Output Result:

| file_path | file_extension |
|---|---|
| /usr/bin/docs/resume.pdf | pdf |
| /downloads/image.PNG | png (Notice the auto lowercase) |
| /home/user/project.tar.gz | gz |
| /root/notes | No Extension |

---


## Joins

Joins are one of the highest-frequency topics in coding rounds. You must understand the difference between them deeply and write them quickly.

### Example Tables (We will use these throughout)

**employees**
| emp_id | name     | dept_id | salary | manager_id |
|--------|----------|---------|--------|------------|
| 1      | Alice    | 10      | 70000  | NULL       |
| 2      | Bob      | 20      | 60000  | 1          |
| 3      | Charlie  | 10      | 55000  | 1          |
| 4      | Diana    | 30      | 80000  | 2          |
| 5      | Eve      | NULL    | 45000  | 2          |

**departments**
| dept_id | dept_name   |
|---------|-------------|
| 10      | Engineering |
| 20      | Sales       |
| 30      | HR          |
| 40      | Marketing   |

### 1. INNER JOIN

Returns only the rows that have **matching values** in both tables.

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id;
```

**Result**:
| name    | dept_name   |
|---------|-------------|
| Alice   | Engineering |
| Bob     | Sales       |
| Charlie | Engineering |
| Diana   | HR          |

**Key Points**:
- Rows without a match are discarded.
- Most common join type.
- You can write just `JOIN` instead of `INNER JOIN`.

### 2. LEFT JOIN (LEFT OUTER JOIN)

Returns **all rows from the left table** + matching rows from the right table.  
If no match → right side columns become `NULL`.

```sql
SELECT e.name, d.dept_name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id;
```

**Result**:
| name    | dept_name   |
|---------|-------------|
| Alice   | Engineering |
| Bob     | Sales       |
| Charlie | Engineering |
| Diana   | HR          |
| Eve     | NULL        |

**RIGHT JOIN** is the opposite (keeps all rows from the right table).

**FULL OUTER JOIN** keeps all rows from both tables (shows `NULL` on either side when no match).

```sql
SELECT e.name, d.dept_name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.dept_id;
```

**Result** will also include Marketing with `name = NULL`.

### 3. Self Join

Joining a table to **itself**. Very common for hierarchical data (manager-employee).

```sql
SELECT 
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

**Result**:
| employee | manager |
|----------|---------|
| Alice    | NULL    |
| Bob      | Alice   |
| Charlie  | Alice   |
| Diana    | Bob     |
| Eve      | Bob     |

**Key Points**:
- You must use different aliases (`e` and `m`).
- Usually combined with `LEFT JOIN` because the top person has no manager.

### 4. Cross Join

Returns the **Cartesian product** (every row of first table with every row of second table).

```sql
SELECT e.name, d.dept_name
FROM employees e
CROSS JOIN departments d;
```

- 5 employees × 4 departments = **20 rows**.
- Rarely used intentionally. Appears when you forget the `ON` condition (accidental cross join).

### 5. Anti-Join Patterns (Very Important)

**Goal**: Find rows in one table that have **no match** in another table.

#### Method 1: `LEFT JOIN` + `IS NULL` (Most common)

```sql
-- Employees who do not belong to any department
SELECT e.name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_id IS NULL;
```

Result → `Eve`

#### Method 2: `NOT EXISTS` (Often preferred for performance & NULL safety)

```sql
SELECT e.name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 
    FROM departments d 
    WHERE d.dept_id = e.dept_id
);
```

#### Method 3: `NOT IN` (Dangerous – avoid when NULLs are possible)

```sql
-- Can give wrong results if the subquery returns NULL
SELECT name FROM employees
WHERE dept_id NOT IN (SELECT dept_id FROM departments);
```

**Best practice**: Prefer `NOT EXISTS` or `LEFT JOIN + IS NULL`.

### 6. Multi-table Joins (3–5 tables)

Just chain the joins. Keep the logic clear.

**Example**: Get employee name, department name, and manager name.

```sql
SELECT 
    e.name AS employee,
    d.dept_name,
    m.name AS manager
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.dept_id
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

**Tips for multi-table joins**:
- Always use clear aliases.
- Decide carefully between `INNER` and `LEFT` based on business requirement.
- Write one join at a time and test intermediate results.
- Be careful of **join explosion** (when one-to-many relationships multiply rows).

### Quick Comparison Table

| Join Type       | Keeps unmatched rows from | When to use |
|-----------------|---------------------------|-----------|
| INNER JOIN      | Neither                   | Only matching records |
| LEFT JOIN       | Left table                | All records from left + matches |
| RIGHT JOIN      | Right table               | All records from right + matches |
| FULL OUTER JOIN | Both                      | All records from both |
| CROSS JOIN      | Both (Cartesian)          | Rarely (combinations) |
| Self Join       | Depends                   | Hierarchy / comparison within same table |
| Anti-Join       | Left (no match)           | "Does not exist" problems |

## Aggregation & Grouping

#### Example Tables (continuing from before)

**employees**
| emp_id | name    | dept_id | salary | gender |
|--------|---------|---------|--------|--------|
| 1      | Alice   | 10      | 70000  | F      |
| 2      | Bob     | 20      | 60000  | M      |
| 3      | Charlie | 10      | 55000  | M      |
| 4      | Diana   | 30      | 80000  | F      |
| 5      | Eve     | 10      | 45000  | F      |
| 6      | Frank   | 20      | 65000  | M      |

### GROUP BY + HAVING

`GROUP BY` divides rows into groups. Aggregate functions then calculate one value **per group**.

```sql
SELECT 
    dept_id,
    COUNT(*) AS emp_count,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY dept_id;
```

**Result**:
| dept_id | emp_count | avg_salary |
|---------|-----------|------------|
| 10      | 3         | 56666.67   |
| 20      | 2         | 62500      |
| 30      | 1         | 80000      |

### Aggregate Functions

| Function            | Description                              | Notes |
|---------------------|------------------------------------------|-------|
| `COUNT(*)`          | Counts all rows                          | Includes NULLs |
| `COUNT(column)`     | Counts non-NULL values                   | Ignores NULLs |
| `COUNT(DISTINCT col)` | Counts unique non-NULL values          | Very common |
| `SUM(column)`       | Total of values                          | Ignores NULLs |
| `AVG(column)`       | Average                                  | Ignores NULLs |
| `MIN(column)` / `MAX(column)` | Minimum / Maximum                 | |

**Important**:
```sql
COUNT(*)          -- counts every row
COUNT(salary)     -- ignores rows where salary is NULL
COUNT(DISTINCT dept_id)
```

### Conditional Aggregation (Extremely High Frequency)

This is one of the most tested patterns in coding rounds.

```sql
SELECT 
    dept_id,
    COUNT(*) AS total_employees,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count,
    AVG(CASE WHEN salary > 60000 THEN salary END) AS avg_high_salary
FROM employees
GROUP BY dept_id;
```

**Alternative styles**:
```sql
COUNT(CASE WHEN gender = 'M' THEN 1 END)          -- also works
SUM(CASE WHEN gender = 'M' THEN salary ELSE 0 END)
```

### WHERE vs HAVING (Very Commonly Tested)

| Clause   | Filters                  | When it runs          | Can use Aggregate? |
|----------|--------------------------|-----------------------|--------------------|
| `WHERE`  | Individual **rows**      | Before grouping       | No                 |
| `HAVING` | **Groups**               | After grouping        | Yes                |

**Example**:
```sql
-- Wrong: Cannot use aggregate in WHERE
SELECT dept_id, AVG(salary)
FROM employees
WHERE AVG(salary) > 60000          -- Error
GROUP BY dept_id;

-- Correct
SELECT dept_id, AVG(salary) AS avg_sal
FROM employees
WHERE salary > 40000               -- filters rows first
GROUP BY dept_id
HAVING AVG(salary) > 60000;        -- filters groups
```

**Interview Rule**:
- Use `WHERE` for row-level conditions.
- Use `HAVING` when the condition involves an aggregate function.

## Subqueries

A subquery is a query nested inside another query.

### Non-Correlated Subquery

The inner query is **independent**. It runs once.

```sql
-- Employees who earn more than the company average
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

### Correlated Subquery

The inner query **depends** on the outer query. It runs once **per row**.

```sql
-- Employees who earn more than the average salary of their own department
SELECT e1.name, e1.salary, e1.dept_id
FROM employees e1
WHERE e1.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.dept_id = e1.dept_id
);
```

**Note**: Correlated subqueries can be slower. Many times they can be rewritten with JOINs or Window Functions.

### Subqueries in Different Places

| Location     | Example Use Case                          |
|--------------|-------------------------------------------|
| `WHERE`      | Filtering (most common)                   |
| `SELECT`     | Calculating a value per row               |
| `FROM`       | Using a subquery as a temporary table     |
| `HAVING`     | Filtering groups                          |

**Subquery in SELECT**:
```sql
SELECT 
    name,
    salary,
    (SELECT AVG(salary) FROM employees) AS company_avg
FROM employees;
```

**Subquery in FROM** (Derived Table):
```sql
SELECT dept_id, avg_sal
FROM (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
) AS dept_avg
WHERE avg_sal > 60000;
```

### EXISTS / NOT EXISTS vs IN / NOT IN

#### 1. `IN` / `NOT IN`

```sql
-- Employees in departments 10 or 20
SELECT name FROM employees
WHERE dept_id IN (10, 20);

-- Using subquery
SELECT name FROM employees
WHERE dept_id IN (SELECT dept_id FROM departments WHERE dept_name = 'Engineering');
```

#### 2. `EXISTS` / `NOT EXISTS`

```sql
-- Employees who belong to at least one department
SELECT e.name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM departments d 
    WHERE d.dept_id = e.dept_id
);
```

### NULL Pitfalls (Very Important)

| Expression                      | Result when subquery has NULL | Recommendation |
|---------------------------------|-------------------------------|----------------|
| `value IN (subquery)`           | Works reasonably              | OK             |
| `value NOT IN (subquery)`       | **Becomes unknown** if NULL exists → returns no rows | **Avoid** |
| `EXISTS` / `NOT EXISTS`         | Safe with NULLs               | **Preferred**  |

**Dangerous example**:
```sql
-- If any dept_id is NULL in the subquery, this returns empty result
SELECT name FROM employees
WHERE dept_id NOT IN (SELECT dept_id FROM departments);
```

**Safe alternatives**:
```sql
-- Preferred
WHERE NOT EXISTS (SELECT 1 FROM departments d WHERE d.dept_id = e.dept_id)

-- Or
LEFT JOIN ... WHERE d.dept_id IS NULL
```

### Summary – Interview Key Points

**Aggregation**
- `WHERE` → rows | `HAVING` → groups
- Master conditional aggregation (`SUM(CASE WHEN...)`)
- Know difference between `COUNT(*)`, `COUNT(col)`, `COUNT(DISTINCT col)`

**Subqueries**
- Non-correlated → runs once
- Correlated → runs per row (can be slow)
- Prefer `EXISTS` / `NOT EXISTS` over `IN` / `NOT IN` when checking existence
- Never use `NOT IN` if the subquery can contain NULL



### Practice: Conditional Aggregation Problems

Here are carefully designed problems from easy → medium (exactly the style asked in company coding rounds).

#### Sample Tables (Use these for all problems)

**employees**
```sql
emp_id | name     | dept_id | salary | gender | join_year
-------|----------|---------|--------|--------|----------
1      | Alice    | 10      | 70000  | F      | 2021
2      | Bob      | 20      | 60000  | M      | 2020
3      | Charlie  | 10      | 55000  | M      | 2022
4      | Diana    | 30      | 80000  | F      | 2019
5      | Eve      | 10      | 45000  | F      | 2023
6      | Frank    | 20      | 65000  | M      | 2021
7      | Grace    | 30      | 72000  | F      | 2020
8      | Henry    | 10      | 90000  | M      | 2018
```

**departments**
```sql
dept_id | dept_name
--------|-------------
10      | Engineering
20      | Sales
30      | HR
```

#### Problem 1: Basic Conditional Count (Easy)

Write a query to show for each department:
- Total employees
- Number of Male employees
- Number of Female employees

**Expected Output:**
```
dept_id | total_emp | male_count | female_count
--------|-----------|------------|-------------
10      | 4         | 2          | 2
20      | 2         | 2          | 0
30      | 2         | 0          | 2
```

#### Problem 2: Conditional Sum & Average (Easy-Medium)

For each department, calculate:
- Total salary of all employees
- Total salary of Male employees
- Total salary of Female employees
- Average salary of employees who earn more than 60,000

#### Problem 3: Multiple Conditions (Medium)

For each department, find:
- Number of employees who joined in 2020 or earlier
- Number of employees who joined after 2020
- Number of High Earners (salary ≥ 70000)
- Number of Female High Earners

#### Problem 4: Percentage using Conditional Aggregation (Medium)

For each department, calculate:
- Total employees
- Male percentage
- Female percentage

(Round percentages to 2 decimal places)

#### Problem 5: Classic Interview Style (Medium)

Write a single query that returns one row with the following columns:

- `total_employees`
- `engineering_employees` (dept_id = 10)
- `sales_employees` (dept_id = 20)
- `hr_employees` (dept_id = 30)
- `male_employees`
- `female_employees`
- `high_salary_employees` (salary ≥ 70000)

#### Problem 6: Advanced Conditional Aggregation (Medium-Hard)

For each department show:

| dept_id | total_emp | male_avg_salary | female_avg_salary | max_male_salary | employees_above_dept_avg |
|---------|-----------|-----------------|-------------------|-----------------|--------------------------|

- `male_avg_salary` → average salary of males only  
- `female_avg_salary` → average salary of females only  
- `max_male_salary` → highest salary among males  
- `employees_above_dept_avg` → count of employees whose salary > department average salary

#### Solutions

**Solution 1:**
```sql
SELECT 
    dept_id,
    COUNT(*) AS total_emp,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count
FROM employees
GROUP BY dept_id
ORDER BY dept_id;
```

**Solution 2:**
```sql
SELECT 
    dept_id,
    SUM(salary) AS total_salary,
    SUM(CASE WHEN gender = 'M' THEN salary ELSE 0 END) AS male_salary,
    SUM(CASE WHEN gender = 'F' THEN salary ELSE 0 END) AS female_salary,
    AVG(CASE WHEN salary > 60000 THEN salary END) AS avg_high_salary
FROM employees
GROUP BY dept_id;
```

**Solution 3:**
```sql
SELECT 
    dept_id,
    SUM(CASE WHEN join_year <= 2020 THEN 1 ELSE 0 END) AS joined_2020_or_earlier,
    SUM(CASE WHEN join_year > 2020 THEN 1 ELSE 0 END) AS joined_after_2020,
    SUM(CASE WHEN salary >= 70000 THEN 1 ELSE 0 END) AS high_earners,
    SUM(CASE WHEN gender = 'F' AND salary >= 70000 THEN 1 ELSE 0 END) AS female_high_earners
FROM employees
GROUP BY dept_id;
```

**Solution 4:**
```sql
SELECT 
    dept_id,
    COUNT(*) AS total_emp,
    ROUND(100.0 * SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) / COUNT(*), 2) AS male_pct,
    ROUND(100.0 * SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) / COUNT(*), 2) AS female_pct
FROM employees
GROUP BY dept_id;
```

**Solution 5:**
```sql
SELECT 
    COUNT(*) AS total_employees,
    SUM(CASE WHEN dept_id = 10 THEN 1 ELSE 0 END) AS engineering_employees,
    SUM(CASE WHEN dept_id = 20 THEN 1 ELSE 0 END) AS sales_employees,
    SUM(CASE WHEN dept_id = 30 THEN 1 ELSE 0 END) AS hr_employees,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_employees,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_employees,
    SUM(CASE WHEN salary >= 70000 THEN 1 ELSE 0 END) AS high_salary_employees
FROM employees;
```

**Solution 6:**
```sql
SELECT 
    e.dept_id,
    COUNT(*) AS total_emp,
    AVG(CASE WHEN gender = 'M' THEN salary END) AS male_avg_salary,
    AVG(CASE WHEN gender = 'F' THEN salary END) AS female_avg_salary,
    MAX(CASE WHEN gender = 'M' THEN salary END) AS max_male_salary,
    SUM(CASE WHEN e.salary > dept_avg.avg_sal THEN 1 ELSE 0 END) AS employees_above_dept_avg
FROM employees e
JOIN (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
) dept_avg ON e.dept_id = dept_avg.dept_id
GROUP BY e.dept_id;
```

**Pro Tips for Conditional Aggregation:**

1. `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` → counting
2. `SUM(CASE WHEN ... THEN salary ELSE 0 END)` → conditional sum
3. `AVG(CASE WHEN ... THEN salary END)` → conditional average (NULL is ignored automatically)
4. You can combine multiple conditions with `AND` / `OR` inside `CASE`


### Harder Conditional Aggregation Problems (Multiple Tables)

Here are 4 medium-to-hard level problems that combine **multiple tables + conditional aggregation**. These are very close to real company coding round questions.

#### Tables

**employees**
```sql
emp_id | name     | dept_id | salary | gender | manager_id
-------|----------|---------|--------|--------|-----------
1      | Alice    | 10      | 90000  | F      | NULL
2      | Bob      | 10      | 75000  | M      | 1
3      | Charlie  | 10      | 62000  | M      | 1
4      | Diana    | 20      | 85000  | F      | 1
5      | Eve      | 20      | 58000  | F      | 4
6      | Frank    | 20      | 72000  | M      | 4
7      | Grace    | 30      | 95000  | F      | 1
8      | Henry    | 30      | 67000  | M      | 7
9      | Ivy      | 30      | 71000  | F      | 7
```

**departments**
```sql
dept_id | dept_name
--------|-------------
10      | Engineering
20      | Sales
30      | HR
```

**projects**
```sql
project_id | emp_id | project_name     | status      | budget
-----------|--------|------------------|-------------|--------
101        | 1      | AI Platform      | Completed   | 500000
102        | 2      | AI Platform      | In Progress | 500000
103        | 3      | Mobile App       | Completed   | 200000
104        | 4      | CRM System       | Completed   | 300000
105        | 5      | CRM System       | In Progress | 300000
106        | 6      | CRM System       | Completed   | 300000
107        | 7      | HR Portal        | Completed   | 150000
108        | 8      | HR Portal        | In Progress | 150000
109        | 2      | Data Pipeline    | Completed   | 400000
110        | 9      | HR Portal        | Completed   | 150000
```

#### Problem 1: Department Performance Summary

For each department, show:

- Department name
- Total employees
- Male employees
- Female employees
- Average salary of Male employees
- Average salary of Female employees
- Number of employees who are managers (appear in `manager_id`)

#### Problem 2: Project Contribution by Gender & Status

For each department, calculate:

- Total projects involved
- Completed projects
- In Progress projects
- Projects done by Male employees
- Projects done by Female employees
- Total budget of Completed projects

#### Problem 3: Manager vs Individual Contributor Analysis

Write a query that returns one row with these metrics:

- Total employees
- Number of Managers
- Number of Individual Contributors (not managers)
- Total salary of Managers
- Total salary of Individual Contributors
- Average salary of Female Managers
- Number of Male Individual Contributors who earn more than 70000

#### Problem 4: Advanced Department + Project Insights

For each department, show:

| dept_name | total_emp | high_earners | completed_projects | male_completed_projects | female_completed_projects | avg_budget_completed |

Where:
- `high_earners` = employees with salary ≥ 75000
- `completed_projects` = number of completed projects in that department
- `male_completed_projects` / `female_completed_projects` = completed projects by gender
- `avg_budget_completed` = average budget of completed projects

#### Solutions

**Solution 1:**
```sql
SELECT 
    d.dept_name,
    COUNT(e.emp_id) AS total_employees,
    SUM(CASE WHEN e.gender = 'M' THEN 1 ELSE 0 END) AS male_employees,
    SUM(CASE WHEN e.gender = 'F' THEN 1 ELSE 0 END) AS female_employees,
    AVG(CASE WHEN e.gender = 'M' THEN e.salary END) AS male_avg_salary,
    AVG(CASE WHEN e.gender = 'F' THEN e.salary END) AS female_avg_salary,
    COUNT(DISTINCT m.manager_id) AS managers_count
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
LEFT JOIN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL) m 
       ON e.emp_id = m.manager_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

**Solution 2:**
```sql
SELECT 
    d.dept_name,
    COUNT(p.project_id) AS total_projects,
    SUM(CASE WHEN p.status = 'Completed' THEN 1 ELSE 0 END) AS completed_projects,
    SUM(CASE WHEN p.status = 'In Progress' THEN 1 ELSE 0 END) AS in_progress_projects,
    SUM(CASE WHEN e.gender = 'M' THEN 1 ELSE 0 END) AS male_projects,
    SUM(CASE WHEN e.gender = 'F' THEN 1 ELSE 0 END) AS female_projects,
    SUM(CASE WHEN p.status = 'Completed' THEN p.budget ELSE 0 END) AS completed_budget
FROM departments d
JOIN employees e ON d.dept_id = e.dept_id
LEFT JOIN projects p ON e.emp_id = p.emp_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```
Write a query that returns one row with these metrics:


**Solution 3:**
```sql
SELECT 
    COUNT(*) AS total_employees, -- Total employees
    COUNT(DISTINCT manager_id) AS managers, -- Number of Managers
    COUNT(*) - COUNT(DISTINCT manager_id) AS individual_contributors, -- Number of Individual Contributors (not managers)
    
    SUM(CASE WHEN emp_id IN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL) -- Total salary of Managers
             THEN salary ELSE 0 END) AS managers_total_salary,
             
    SUM(CASE WHEN emp_id NOT IN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL) -- Total salary of Individual Contributors
             THEN salary ELSE 0 END) AS ic_total_salary,
             
    AVG(CASE WHEN gender = 'F' AND emp_id IN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL) -- Average salary of Female Managers
             THEN salary END) AS female_manager_avg_salary,
             
    SUM(CASE WHEN gender = 'M' 
              AND emp_id NOT IN (SELECT DISTINCT manager_id FROM employees WHERE manager_id IS NOT NULL) -- Number of Male Individual Contributors who earn more than 70000
              AND salary > 70000 THEN 1 ELSE 0 END) AS male_ic_high_earners
FROM employees;
```

**Solution 4:**
```sql
SELECT 
    d.dept_name as dept_name, 
    COUNT(DISTINCT e.emp_id) AS total_emp,
    COUNT(DISTINCT CASE WHEN e.salary >= 75000 THEN e.emp_id END) AS high_ern,
    COUNT(CASE WHEN p.status = 'Completed' THEN 1 END) AS completed_projects,
    COUNT(CASE WHEN e.gender='M' AND  p.status = 'Completed' THEN 1 END) AS male_completed_projects,
    COUNT(CASE WHEN e.gender='F' AND  p.status = 'Completed' THEN 1 END) AS female_completed_projects,
    AVG(CASE WHEN p.status = 'Completed' THEN p.budget END) AS avg_budget_completed
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id 
LEFT JOIN projects p ON e.emp_id = p.emp_id
GROUP BY d.dept_name
ORDER BY d.dept_name;

SELECT 
    d.dept_name,
    COUNT(DISTINCT e.emp_id) AS total_emp,
    COUNT(DISTINCT CASE WHEN e.salary >= 75000 THEN e.emp_id END) AS high_earners,
    SUM(CASE WHEN p.status = 'Completed' THEN 1 ELSE 0 END) AS completed_projects,
    SUM(CASE WHEN p.status = 'Completed' AND e.gender = 'M' THEN 1 ELSE 0 END) AS male_completed_projects,
    SUM(CASE WHEN p.status = 'Completed' AND e.gender = 'F' THEN 1 ELSE 0 END) AS female_completed_projects,
    AVG(CASE WHEN p.status = 'Completed' THEN p.budget END) AS avg_budget_completed
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
LEFT JOIN projects p ON e.emp_id = p.emp_id
GROUP BY d.dept_name
ORDER BY d.dept_name;

SELECT 
    d.dept_name,
    COUNT(DISTINCT e.emp_id) AS total_emp,
    COUNT(DISTINCT CASE WHEN e.salary >= 75000 THEN e.emp_id END) AS high_earners,
    SUM(CASE WHEN p.status = 'Completed' THEN 1 ELSE 0 END) AS completed_projects,
    SUM(CASE WHEN p.status = 'Completed' AND e.gender = 'M' THEN 1 ELSE 0 END) AS male_completed_projects,
    SUM(CASE WHEN p.status = 'Completed' AND e.gender = 'F' THEN 1 ELSE 0 END) AS female_completed_projects,
    -- Safe Side: COALESCE lagaya taaki NULL ke badle 0 aaye( The COALESCE() function returns the first non-null value in a list.)
    COALESCE(AVG(CASE WHEN p.status = 'Completed' THEN p.budget END), 0) AS avg_budget_completed
FROM departments d
LEFT JOIN employees e ON d.dept_id = e.dept_id
LEFT JOIN projects p ON e.emp_id = p.emp_id
GROUP BY d.dept_name
ORDER BY d.dept_name;


```

**Key Patterns Used:**
- Multiple `CASE WHEN` inside `SUM` / `AVG` / `COUNT`
- Combining `JOIN` + conditional aggregation
- Using subqueries inside `CASE` for manager logic
- Handling `LEFT JOIN` carefully so departments with no projects still appear

---

## Window Functions
(Most Important Advanced Topic for Tier-1 / Tier-2 Coding Rounds)

Window functions are the biggest differentiator between average and strong candidates.  
They allow you to perform calculations across a set of rows **related to the current row**, without collapsing the result like `GROUP BY`.

### Basic Syntax

[FUNCTION_NAME()]    +    [OVER]    +    [(WINDOW PARAMETERS)]

```sql
function_name() OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3 ASC/DESC]
    [frame_clause]
)
```
- `OVER` → it is the mandatory keyword that activates a Window Function. It explicitly tells the database engine to create a custom window (subset of rows) for the function to operate on, rather than collapsing the rows using a standard GROUP BY.
- `PARTITION BY` → divides data into groups (similar to `GROUP BY`, but rows are **not** collapsed)
- `ORDER BY` → orders rows **inside** each partition
- Frame clause → defines which rows in the partition are used for the calculation

#### Rows are NOT collapsed
Jab aap standard GROUP BY use karte hain, toh database pure groups ko sametkar (collapse karke) sirf ek summary row bana deta hai. Har individual row ka wajood khatam ho jata hai.
Lekin jab aap PARTITION BY use karte hain, toh database piche background me groups toh banata hai, lekin woh har ek row ko as-is screen par barkarar rakhta hai. Woh har employee ki row ke aage uske pure department ki summary calculate karke chipka deta hai.

Real Example Se Samjhein (Standard Table Data):
Maan lijiye aapke paas Engineering department me 3 log hain:

* Alice (Salary: 90,000)
* Bob (Salary: 75,000)
* Charlie (Salary: 62,000)

##### Case A: GROUP BY dept_id (Rows get Collapsed)
Agar aap pure department ki total salary nikalenge standard GROUP BY se:
```sql
SELECT dept_id, SUM(salary) FROM employees GROUP BY dept_id;
```

* Output: Sirf 1 single row aayegi:
Engineering | 227,000
* Nuksan: Alice, Bob, aur Charlie ke naam aur unki individual salaries output se gayab (collapse) ho gayin.

##### Case B: PARTITION BY dept_id (Rows are NOT Collapsed)
Agar aap wahi total nikalenge Window Function (OVER PARTITION BY) use karke:
```sql
SELECT name, salary, SUM(salary) OVER(PARTITION BY dept_id) AS dept_total FROM employees;
```

* Output: Pure 3 rows aayengi! Ek bhi row collapse nahi hogi:
1. Alice | 90,000 | 227,000
2. Bob | 75,000 | 227,000
3. Charlie | 62,000 | 227,000
* Fayda: Aap har employee ki details bhi dekh pa rahe hain, aur uske side me uske pure department ka total bhi chal raha hai. (Isse aap easily compare kar sakte hain ki kiski salary total se kitni kam ya zyada hai).

#### OVER
SQL me OVER ek keyword hai jo database ko batata hai: "Suno! Agla function koi ordinary aggregation function nahi hai, balki ek Window Function hai."

* Ordinary Function: Agar aap sirf `SUM(salary)` likhenge, toh SQL use normal aggregate samjhega aur aapse `GROUP BY` mangega.
* Window Function: Jab aap `SUM(salary) OVER (...)` likhte hain, toh `OVER` keyword database ke liye ek khidki (window) kholta hai. Yeh khidki database ko batati hai ki is calculation ko karne ke liye tumhe table ke kis hisse (partition) par nazar rakhni hai.


### 1. Ranking Functions

| Function       | Behavior                                                                 | Gaps in ranking? |
|----------------|--------------------------------------------------------------------------|------------------|
| `ROW_NUMBER()` | Unique sequential number (1, 2, 3, 4...) even if values are equal        | No               |
| `RANK()`       | Same rank for ties, then **skips** the next ranks                       | Yes              |
| `DENSE_RANK()` | Same rank for ties, **does not skip** ranks                             | No               |
| `NTILE(n)`     | Divides rows into `n` roughly equal buckets                             | -                |

**Example:**

```sql
SELECT 
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS rank_num,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_num
FROM employees;
```

### 2. Value Functions

| Function            | Description                                      |
|---------------------|--------------------------------------------------|
| `LAG(column, n)`    | Value from **n rows before** the current row     |
| `LEAD(column, n)`   | Value from **n rows after** the current row      |
| `FIRST_VALUE(col)`  | First value in the window frame                  |
| `LAST_VALUE(col)`   | Last value in the window frame                   |

```sql
SELECT 
    name,
    salary,
    LAG(salary, 1) OVER (ORDER BY salary) AS prev_salary,
    LEAD(salary, 1) OVER (ORDER BY salary) AS next_salary,
    FIRST_VALUE(salary) OVER (ORDER BY salary) AS lowest_salary
FROM employees;
```

**Common use cases**:
- Difference from previous day/month
- Compare with previous record
- Finding gaps

### 3. Aggregate Window Functions

You can use normal aggregate functions as window functions:

```sql
SUM() OVER()
AVG() OVER()
COUNT() OVER()
MIN() OVER()
MAX() OVER()
```

**Example – Running Total:**

```sql
SELECT 
    name,
    salary,
    SUM(salary) OVER (ORDER BY salary) AS running_total,
    AVG(salary) OVER (ORDER BY salary) AS running_avg,
    COUNT(*) OVER () AS total_employees
FROM employees;
```

### 4. PARTITION BY + ORDER BY (Most Used Combination)

```sql
-- Rank employees by salary within each department
SELECT 
    name,
    dept_id,
    salary,
    RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dept_rank,
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dept_row_num
FROM employees;
```

**Key Difference from GROUP BY**:
- `GROUP BY` → collapses rows
```sql
SELECT dept_id, SUM(salary) FROM employees GROUP BY dept_id;
```
**Output:**
```
+---------+-------------+
| dept_id | SUM(salary) |
+---------+-------------+
|      10 |   227000.00 |
|      20 |   215000.00 |
|      30 |   233000.00 |
+---------+-------------+
3 rows in set (0.001 sec)
```

- `PARTITION BY` → keeps all rows and adds extra calculated columns
```sql
SELECT name, salary, SUM(salary) OVER(PARTITION BY dept_id) AS dept_total FROM employees;
```
**Output:**
```
+---------+----------+------------+
| name    | salary   | dept_total |
+---------+----------+------------+
| Bob     | 75000.00 |  227000.00 |
| Alice   | 90000.00 |  227000.00 |
| Charlie | 62000.00 |  227000.00 |
| Eve     | 58000.00 |  215000.00 |
| Diana   | 85000.00 |  215000.00 |
| Frank   | 72000.00 |  215000.00 |
| Ivy     | 71000.00 |  233000.00 |
| Henry   | 67000.00 |  233000.00 |
| Grace   | 95000.00 |  233000.00 |
+---------+----------+------------+
```

### 5. Frame Clauses (`ROWS BETWEEN`)

Controls **which rows** are included in the calculation.

Common frames:

```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW   -- default for running total
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING           -- previous + current + next
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- entire partition
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
```

**Example – 3-month moving average style:**

```sql
SELECT 
    name,
    salary,
    AVG(salary) OVER (
        ORDER BY salary
        ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
    ) AS moving_avg
FROM employees;
```

### 6. Most Important Patterns (Must Master)

#### Pattern 1: Top-N per Group
```sql
SELECT *
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn <= 2;          -- Top 2 earners per department
```

#### Pattern 2: Latest / Most Recent Record per Group
```sql
SELECT *
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY join_date DESC) AS rn
    FROM employee_history
) t
WHERE rn = 1;
```

#### Pattern 3: Running Total / Cumulative Sum
```sql
SELECT 
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

#### Pattern 4: Rank + Filter in Outer Query
```sql
-- Get only the highest paid employee in each department
SELECT name, dept_id, salary
FROM (
    SELECT 
        name, dept_id, salary,
        RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
) t
WHERE rnk = 1;
```

#### Quick Comparison: RANK vs DENSE_RANK vs ROW_NUMBER

| Salary | ROW_NUMBER | RANK | DENSE_RANK |
|--------|------------|------|------------|
| 90000  | 1          | 1    | 1          |
| 85000  | 2          | 2    | 2          |
| 85000  | 3          | 2    | 2          |
| 80000  | 4          | 4    | 3          |

- Use `ROW_NUMBER()` when you need unique numbers (Top-N, pagination, latest record)
- Use `DENSE_RANK()` when ties should not create gaps
- Use `RANK()` when you want Olympic-style ranking (with gaps)

#### Interview Tips

1. Prefer `ROW_NUMBER()` for "Top N" and “latest record” problems.
2. Always ask yourself: Do I need `PARTITION BY`?
3. When filtering on a window function → must use subquery / CTE (you cannot use window function directly in `WHERE`).
4. Frame clause is rarely asked in depth, but knowing `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is useful.
5. Window functions are usually faster and cleaner than correlated subqueries.


### Window Functions – Practice Problems

Here is a progressive set of problems (Easy → Medium → Hard) focused on the most important patterns asked in Tier-1 & Tier-2 coding rounds.

#### Sample Table: `employees`

```sql
emp_id | name     | dept_id | salary | join_date  | gender
-------|----------|---------|--------|------------|--------
1      | Alice    | 10      | 90000  | 2020-01-15 | F
2      | Bob      | 10      | 75000  | 2021-03-20 | M
3      | Charlie  | 10      | 75000  | 2019-07-10 | M
4      | Diana    | 20      | 85000  | 2020-11-05 | F
5      | Eve      | 20      | 58000  | 2022-02-14 | F
6      | Frank    | 20      | 72000  | 2021-09-30 | M
7      | Grace    | 30      | 95000  | 2018-05-22 | F
8      | Henry    | 30      | 67000  | 2023-01-08 | M
9      | Ivy      | 30      | 71000  | 2020-08-19 | F
10     | Jack     | 10      | 82000  | 2022-06-12 | M
```

#### Easy Problems

**Problem 1: Basic Ranking**  
Show each employee’s name, salary, and their overall rank based on salary (highest first). Use both `RANK()` and `DENSE_RANK()`.

**Problem 2: Ranking within Department**  
For each employee, show:
- name
- dept_id
- salary
- Rank within their department (highest salary = rank 1)

**Problem 3: Running Total**  
Show employee name, salary, and the running total of salary when ordered by salary ascending.

#### Medium Problems

**Problem 4: Top-N per Group**  
Find the **Top 2 highest paid employees in each department**.

**Problem 5: Latest Record per Group**  
Assuming we want the employee who joined most recently in each department, write a query to return only those employees.

**Problem 6: LAG & LEAD**  
For each employee (ordered by salary), show:
- Current salary
- Previous employee’s salary (`LAG`)
- Next employee’s salary (`LEAD`)
- Difference between current and previous salary

**Problem 7: First & Last Value**  
For each department, show every employee along with:
- The highest salary in their department (`FIRST_VALUE`)
- The lowest salary in their department (`LAST_VALUE`)

#### Hard Problems

**Problem 8: Compare with Department Average**  
Show each employee’s name, salary, department average salary, and the difference (salary – dept_avg).

**Problem 9: Rank + Filter (Classic Pattern)**  
Find the employee(s) who have the **2nd highest salary in each department**.  
(Handle ties properly — decide whether to use `RANK` or `DENSE_RANK`)

**Problem 10: Complex Combination**  
For each department, show:

| dept_id | name | salary | dept_rank | salary_diff_from_top | running_total_in_dept |

Where:
- `dept_rank` → rank by salary within department
- `salary_diff_from_top` → difference from the highest salary in that department
- `running_total_in_dept` → running total of salary within department (ordered by salary)

#### Solutions

**Solution 1:**
```sql
SELECT 
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank_num,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_num
FROM employees;
```

**Solution 2:**
```sql
SELECT 
    name,
    dept_id,
    salary,
    RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dept_rank
FROM employees;
```

**Solution 3:**
```sql
SELECT 
    name,
    salary,
    SUM(salary) OVER (ORDER BY salary) AS running_total
FROM employees;
```

**Solution 4: Top 2 per Department**
```sql
SELECT *
FROM (
    SELECT 
        name, dept_id, salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn <= 2;
```

**Solution 5: Most Recently Joined per Department**
```sql
SELECT *
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY join_date DESC) AS rn
    FROM employees
) t
WHERE rn = 1;
```

**Solution 6: LAG & LEAD**
```sql
SELECT 
    name,
    salary,
    LAG(salary) OVER (ORDER BY salary) AS prev_salary,
    LEAD(salary) OVER (ORDER BY salary) AS next_salary,
    salary - LAG(salary) OVER (ORDER BY salary) AS diff_from_prev
FROM employees;
```

**Solution 7: First & Last Value**
```sql
SELECT 
    name,
    dept_id,
    salary,
    FIRST_VALUE(salary) OVER (PARTITION BY dept_id ORDER BY salary DESC) AS highest_in_dept,
    LAST_VALUE(salary) OVER (
        PARTITION BY dept_id 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS lowest_in_dept
FROM employees;


```

**Solution 8: Salary vs Department Average**
```sql
SELECT 
    name,
    dept_id,
    salary,
    AVG(salary) OVER (PARTITION BY dept_id) AS dept_avg,
    salary - AVG(salary) OVER (PARTITION BY dept_id) AS diff_from_avg
FROM employees;
```

**Solution 9: 2nd Highest Salary per Department**
```sql
SELECT name, dept_id, salary
FROM (
    SELECT 
        name, dept_id, salary,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS drk
    FROM employees
) t
WHERE drk = 2;
```

**Solution 10: Complex Combination**
```sql
SELECT 
    dept_id,
    name,
    salary,
    RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dept_rank,
    FIRST_VALUE(salary) OVER (PARTITION BY dept_id ORDER BY salary DESC) - salary AS salary_diff_from_top,
    SUM(salary) OVER (PARTITION BY dept_id ORDER BY salary) AS running_total_in_dept
FROM employees
ORDER BY dept_id, salary DESC;
```

**Key Patterns You Should Master**

1. `ROW_NUMBER()` + filter in outer query → Top-N / Latest record
2. `RANK()` / `DENSE_RANK()` for competitive ranking
3. `LAG()` / `LEAD()` for previous/next comparisons
4. `SUM() OVER(ORDER BY ...)` → Running total
5. `AVG() OVER(PARTITION BY ...)` → Compare with group average
6. Always wrap window functions in a subquery/CTE when you need to filter on them

---

## CTEs (Common Table Expressions)

CTEs make complex queries **readable, modular, and easier to debug**.  
In interviews, using CTEs instead of deeply nested subqueries is considered a sign of clean thinking.

### 1. Simple CTE (`WITH`)

A CTE is a temporary named result set that exists only for the duration of the query.

**Syntax:**
```sql
WITH cte_name AS (
    -- your query here
)
SELECT *
FROM cte_name;
```

**Example:**
```sql
WITH high_salary_employees AS (
    SELECT *
    FROM employees
    WHERE salary > 70000
)
SELECT name, salary, dept_id
FROM high_salary_employees
ORDER BY salary DESC;
```

**Key Points:**
- CTE improves readability.
- You can refer to the CTE multiple times in the main query.
- CTE is **not** stored permanently (unlike a temporary table).

### 2. Multiple CTEs

You can define several CTEs by separating them with commas.

```sql
WITH 
dept_avg AS (
    SELECT 
        dept_id,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
),
high_earners AS (
    SELECT *
    FROM employees
    WHERE salary > 75000
)
SELECT 
    h.name,
    h.salary,
    d.avg_salary,
    h.salary - d.avg_salary AS diff_from_avg
FROM high_earners h
JOIN dept_avg d ON h.dept_id = d.dept_id;
```

**When to use multiple CTEs:**
- Breaking a complex problem into logical steps
- Reusing intermediate results
- Making window function + filtering cleaner

### 3. CTEs + Window Functions (Very Common Pattern)

This is one of the cleanest ways to write Top-N or ranking problems.

```sql
WITH ranked_employees AS (
    SELECT 
        name,
        dept_id,
        salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT name, dept_id, salary
FROM ranked_employees
WHERE rn <= 2;
```

**Why this is preferred:**
- Much more readable than nested subqueries
- Easy to modify (change `rn <= 2` to `rn <= 3`)

### 4. Recursive CTEs

Used for **hierarchical / tree-structured data** (manager → employee, category → subcategory, graph traversal, etc.).

**Basic Structure:**
```sql
WITH RECURSIVE cte_name AS (
    -- Anchor member (starting point)
    SELECT ...
    FROM ...
    WHERE condition          -- base case

    UNION ALL

    -- Recursive member
    SELECT ...
    FROM cte_name
    JOIN original_table ON ...
)
SELECT * FROM cte_name;
```

### Classic Example: Employee Hierarchy (Manager Chain)

```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Anchor: Start with top-level managers (no manager)
    SELECT 
        emp_id,
        name,
        manager_id,
        1 AS level,
        CAST(name AS CHAR(200)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive part: Find employees reporting to current level
    SELECT 
        e.emp_id,
        e.name,
        e.manager_id,
        eh.level + 1,
        CONCAT(eh.path, ' → ', e.name)
    FROM employees e
    JOIN employee_hierarchy eh ON e.manager_id = eh.emp_id
)
SELECT *
FROM employee_hierarchy
ORDER BY path;
```

**What this does:**
- Starts from employees who have no manager
- Recursively finds all people below them
- Builds the full reporting path and level

### Another Common Recursive Use Case: Numbers / Sequence Generation

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1
    FROM numbers
    WHERE n < 10
)
SELECT * FROM numbers;
```

### Important Differences: CTE vs Subquery vs Temporary Table

| Feature              | CTE                  | Subquery             | Temp Table          |
|----------------------|----------------------|----------------------|---------------------|
| Readability          | Excellent            | Poor (when nested)   | Good                |
| Reusability          | Can be referenced multiple times | Limited         | Yes                 |
| Performance          | Usually same         | Usually same         | Can be better for very large data |
| Scope                | Only current query   | Only current query   | Session-level       |
| Recursive support    | Yes                  | No                   | No                  |

### Interview Tips for CTEs

1. Prefer CTEs over deeply nested subqueries — interviewers love clean code.
2. Name CTEs meaningfully (`ranked_employees`, `dept_stats`, `high_earners`).
3. You **cannot** use a CTE in the same `WITH` clause before it is defined (order matters).
4. Recursive CTEs are asked less frequently, but when asked, they usually involve:
   - Organization hierarchy
   - Bill of materials
   - Finding all subordinates
   - Graph traversal (friends of friends, etc.)
5. In most databases, recursive CTEs have a default recursion limit (usually 100). You can control it in some systems.

### Common Patterns You Should Practice

- Using CTE for Top-N per group
- Using CTE to calculate aggregates and then join back
- Multiple CTEs for step-by-step logic
- Recursive CTE for hierarchy

---

## Set Operations

Set operations combine the results of two or more queries into a single result set.  
They are less frequent than joins or window functions, but still appear in coding rounds (especially when comparing two result sets).

### 1. UNION vs UNION ALL

| Operator     | Description                              | Removes Duplicates? | Performance |
|--------------|------------------------------------------|---------------------|-------------|
| `UNION`      | Combines results and **removes duplicates** | Yes                | Slower (needs sort/distinct) |
| `UNION ALL`  | Combines results and **keeps duplicates**  | No                 | Faster      |

**Syntax:**
```sql
SELECT column1, column2 FROM table1
UNION          -- or UNION ALL
SELECT column1, column2 FROM table2;
```

**Rules:**
- Both queries must have the **same number of columns**.
- Corresponding columns must have **compatible data types**.
- Column names come from the **first** query.
- `ORDER BY` can only be used at the very end.

**Example:**
```sql
-- Employees from Engineering + Sales
SELECT name, dept_id FROM employees WHERE dept_id = 10
UNION
SELECT name, dept_id FROM employees WHERE dept_id = 20;
```

**When to use which?**
- Use `UNION ALL` by default (faster) unless you specifically need unique rows.
- `UNION` is useful when you want distinct combined results.

### 2. INTERSECT

Returns only the rows that appear in **both** result sets.

```sql
SELECT name FROM employees WHERE dept_id = 10
INTERSECT
SELECT name FROM employees WHERE salary > 70000;
```

**Behavior:**
- Removes duplicates automatically (like `UNION`).
- Useful for finding common records between two sets.

**Note:** Not supported in older MySQL versions (available in MySQL 8.0.31+).

### 3. EXCEPT / MINUS

Returns rows that are in the **first** query but **not** in the second query.

| Database       | Keyword   |
|----------------|-----------|
| PostgreSQL, SQL Server, SQLite | `EXCEPT` |
| Oracle         | `MINUS`   |

```sql
-- Employees in Engineering but not high earners
SELECT name FROM employees WHERE dept_id = 10
EXCEPT
SELECT name FROM employees WHERE salary > 80000;
```

**Key Points:**
- Order matters: `A EXCEPT B` is different from `B EXCEPT A`.
- Duplicates are removed.
- Equivalent to an anti-join in many cases.

### Important Rules for All Set Operations

1. **Same number of columns** in all queries.
2. **Compatible data types** in corresponding positions.
3. Column names / aliases are taken from the **first** SELECT.
4. `ORDER BY` is allowed only once — at the end of the entire statement.
5. `NULL` values are considered equal for matching purposes in set operations.

### Practical Examples

**Example 1: Combine two departments**
```sql
SELECT name, 'Engineering' AS dept
FROM employees WHERE dept_id = 10
UNION ALL
SELECT name, 'Sales' AS dept
FROM employees WHERE dept_id = 20;
```

**Example 2: Find common high performers**
```sql
SELECT emp_id FROM employees WHERE salary > 80000
INTERSECT
SELECT emp_id FROM projects WHERE status = 'Completed';
```

**Example 3: Find employees who never worked on any project**
```sql
SELECT emp_id FROM employees
EXCEPT
SELECT emp_id FROM projects;
```

### Set Operations vs Joins

| Use Case                        | Prefer Set Operation      | Prefer Join                  |
|---------------------------------|---------------------------|------------------------------|
| Combine similar result sets     | `UNION` / `UNION ALL`     | -                            |
| Find common rows                | `INTERSECT`               | `INNER JOIN`                 |
| Find rows in A but not in B     | `EXCEPT`                  | `LEFT JOIN + IS NULL` or `NOT EXISTS` |
| Need columns from both tables   | -                         | Joins                        |
| Performance on large data       | Joins usually better      | -                            |

**Interview Tip:**  
Many problems that can be solved with `EXCEPT` can also be solved with `NOT EXISTS` or `LEFT JOIN ... IS NULL`. Interviewers sometimes prefer the join/anti-join version for performance reasons.

### Performance Notes

- `UNION ALL` is significantly faster than `UNION` (no deduplication).
- `INTERSECT` and `EXCEPT` usually require sorting or hashing → can be expensive on large datasets.
- On very large tables, rewriting with joins + `EXISTS`/`NOT EXISTS` is often faster.
- Always check the execution plan if performance matters.

### Summary – Quick Reference

| Operator       | Keeps Duplicates? | Meaning                          | Common Alternative          |
|----------------|-------------------|----------------------------------|-----------------------------|
| `UNION`        | No                | A ∪ B (unique)                   | -                           |
| `UNION ALL`    | Yes               | A ∪ B (with duplicates)          | Preferred for performance   |
| `INTERSECT`    | No                | A ∩ B                            | `INNER JOIN`                |
| `EXCEPT`/`MINUS` | No              | A − B                            | `NOT EXISTS` / Anti-join    |

**Set Operations – Practice Problems** (Short & Focused)

### Sample Data

**employees**
```sql
emp_id | name    | dept_id | salary
-------|---------|---------|--------
1      | Alice   | 10      | 90000
2      | Bob     | 10      | 75000
3      | Charlie | 20      | 82000
4      | Diana   | 20      | 68000
5      | Eve     | 30      | 95000
```

**projects**
```sql
emp_id | project_name
-------|---------------
1      | AI Platform
2      | AI Platform
3      | CRM System
6      | Data Pipeline   -- emp_id 6 does not exist in employees
```

**Problem 1**  
Get a combined list of all employee names and all project names (with duplicates allowed).

**Problem 2**  
Find employees who work in either department 10 or department 20 (remove duplicates).

**Problem 3**  
Find emp_ids that exist in **both** the `employees` table and the `projects` table.

**Problem 4**  
Find emp_ids that exist in `employees` but **not** in `projects`.

#### Solutions

```sql
-- Problem 1
SELECT name AS item FROM employees
UNION ALL
SELECT project_name FROM projects;

-- Problem 2
SELECT name FROM employees WHERE dept_id = 10
UNION
SELECT name FROM employees WHERE dept_id = 20;

-- Problem 3
SELECT emp_id FROM employees
INTERSECT
SELECT emp_id FROM projects;

-- Problem 4
SELECT emp_id FROM employees
EXCEPT
SELECT emp_id FROM projects;
```

### 8. Advanced Patterns (High ROI for Tier-1 / Tier-2)

#### 1. Gaps & Islands

**Problem type**: Find consecutive sequences (consecutive days, consecutive numbers, consecutive logins, etc.).

**Classic Example** – Find consecutive salary ranks or consecutive dates.

**Common Technique**:
```sql
WITH numbered AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (ORDER BY some_column) AS rn,
        some_column - ROW_NUMBER() OVER (ORDER BY some_column) AS group_id
    FROM table
)
SELECT 
    MIN(some_column) AS start_value,
    MAX(some_column) AS end_value,
    COUNT(*) AS island_length
FROM numbered
GROUP BY group_id;
```

#### 2. Pivot (Rows to Columns)

Turning row values into columns using conditional aggregation.

```sql
SELECT 
    dept_id,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count
FROM employees
GROUP BY dept_id;
```

#### 3. Unpivot (Columns to Rows)

Less common, but sometimes asked. Usually done with `UNION ALL` or lateral joins.

#### 4. Deduplication (Keep Latest Record)

```sql
SELECT *
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY updated_at DESC) AS rn
    FROM employee_history
) t
WHERE rn = 1;
```

#### 5. Running Totals / Cumulative Metrics

Already covered under Window Functions (`SUM() OVER (ORDER BY ...)`).

#### 6. Hierarchical / Tree Problems

Solved with **Recursive CTEs** (already covered).

#### 7. Date Spine / Sequence Generation

Generate a continuous series of dates (very useful for reporting).

```sql
WITH RECURSIVE date_spine AS (
    SELECT DATE '2025-01-01' AS dt
    UNION ALL
    SELECT dt + 1
    FROM date_spine
    WHERE dt < DATE '2025-01-31'
)
SELECT * FROM date_spine;
```

#### Most Important Advanced Patterns to Master

| Pattern                      | Frequency | Main Technique                  |
|-----------------------------|-----------|---------------------------------|
| Top-N per group             | Very High | `ROW_NUMBER()` + filter         |
| Latest record per group     | Very High | `ROW_NUMBER()` + filter         |
| Gaps & Islands              | Medium    | `ROW_NUMBER()` difference trick |
| Pivot                       | Medium    | Conditional Aggregation         |
| Running Total / Cumulative  | High      | Window `SUM()`                  |
| Hierarchy                   | Medium    | Recursive CTE                   |
| Deduplication               | High      | `ROW_NUMBER()`                  |

---

## Advanced / Pattern-Based Topics (High ROI)

These are the patterns that appear most often in medium and hard SQL coding rounds. Mastering them gives you a big advantage.

### 1. Deduplication (Keep Latest Record)

**Goal**: Keep only the most recent row for each entity.

```sql
SELECT *
FROM (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY emp_id ORDER BY updated_at DESC) AS rn
    FROM employee_history
) t
WHERE rn = 1;
```

**Variations**:
- Use `RANK()` or `DENSE_RANK()` if you want to keep ties.
- Prefer `ROW_NUMBER()` when you need exactly one row.

### 2. Top-N / Nth Highest Problems

**Top-N per group** (most common):
```sql
SELECT *
FROM (
    SELECT 
        name, dept_id, salary,
        ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn <= 3;          -- Top 3 per department
```

**Nth Highest (Overall)**:
```sql
SELECT DISTINCT salary
FROM employees e1
WHERE 2 = (
    SELECT COUNT(DISTINCT salary)
    FROM employees e2
    WHERE e2.salary >= e1.salary
);   -- 2nd highest salary
```

**Better modern way** (using window function):
```sql
SELECT *
FROM (
    SELECT 
        *,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS drk
    FROM employees
) t
WHERE drk = 2;
```

### 3. Gaps & Islands

**Goal**: Find consecutive sequences (consecutive dates, consecutive numbers, consecutive logins, etc.).

**Classic Technique**:
```sql
WITH cte AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (ORDER BY some_date) AS rn,
        some_date - INTERVAL '1 day' * ROW_NUMBER() OVER (ORDER BY some_date) AS island_id
    FROM events
)
SELECT 
    MIN(some_date) AS start_date,
    MAX(some_date) AS end_date,
    COUNT(*) AS consecutive_days
FROM cte
GROUP BY island_id
ORDER BY start_date;
```

**Key Idea**:  
`value - ROW_NUMBER()` stays constant within a consecutive group.

### 4. Pivot / Unpivot Style

**Pivot (Rows → Columns)** using Conditional Aggregation:

```sql
SELECT 
    dept_id,
    SUM(CASE WHEN gender = 'M' THEN 1 ELSE 0 END) AS male_count,
    SUM(CASE WHEN gender = 'F' THEN 1 ELSE 0 END) AS female_count,
    SUM(CASE WHEN gender = 'M' THEN salary ELSE 0 END) AS male_salary,
    SUM(CASE WHEN gender = 'F' THEN salary ELSE 0 END) AS female_salary
FROM employees
GROUP BY dept_id;
```

**Unpivot (Columns → Rows)** – usually done with `UNION ALL`:

```sql
SELECT emp_id, 'Q1' AS quarter, q1_sales AS sales FROM sales
UNION ALL
SELECT emp_id, 'Q2', q2_sales FROM sales
UNION ALL
SELECT emp_id, 'Q3', q3_sales FROM sales;
```

### 5. Date/Time Calculations & Consecutive Days/Events

Common requirements:
- Consecutive login days
- Users active for N consecutive days
- Gaps between events
- Month-over-month / Year-over-year

**Example – Consecutive Days**:
```sql
WITH ordered AS (
    SELECT 
        user_id,
        login_date,
        login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date)::INT AS grp
    FROM user_logins
)
SELECT 
    user_id,
    MIN(login_date) AS start_date,
    MAX(login_date) AS end_date,
    COUNT(*) AS consecutive_days
FROM ordered
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;      -- at least 3 consecutive days
```

### 6. Percentage / Ratio Calculations

```sql
SELECT 
    dept_id,
    COUNT(*) AS dept_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) AS percentage_of_total
FROM employees
GROUP BY dept_id;
```

**Within group percentage**:
```sql
SELECT 
    name,
    dept_id,
    salary,
    ROUND(100.0 * salary / SUM(salary) OVER (PARTITION BY dept_id), 2) AS pct_of_dept_salary
FROM employees;
```

### 7. Finding Duplicates

**Find duplicate emails / values**:
```sql
SELECT email, COUNT(*)
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

**Show all rows that have duplicates**:
```sql
SELECT *
FROM users
WHERE email IN (
    SELECT email
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
);
```

**Using Window Function**:
```sql
SELECT *
FROM (
    SELECT 
        *,
        COUNT(*) OVER (PARTITION BY email) AS cnt
    FROM users
) t
WHERE cnt > 1;
```

### 8. Employees Earning More Than Their Manager (Self-Join Classic)

```sql
SELECT 
    e.name AS employee,
    e.salary AS emp_salary,
    m.name AS manager,
    m.salary AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

**Using Window Function alternative** (less common but possible):
```sql
SELECT *
FROM (
    SELECT 
        name,
        salary,
        manager_id,
        FIRST_VALUE(salary) OVER (PARTITION BY manager_id ORDER BY salary DESC) AS highest_under_manager
    FROM employees
) t
WHERE salary > (SELECT salary FROM employees WHERE emp_id = t.manager_id);
```

### Priority Order for Practice

| Priority | Pattern                        | Why |
|---------|--------------------------------|-----|
| Highest | Top-N per group + Deduplication | Extremely common |
| High    | Self-join (Employee > Manager) | Classic interview question |
| High    | Percentage / Ratio             | Very frequent in analytics rounds |
| Medium  | Gaps & Islands                 | Differentiator in harder rounds |
| Medium  | Pivot using CASE               | Useful for reporting style questions |
| Medium  | Consecutive days/events        | Common in product analytics |

### Gaps & Islands – Practice Problems

Gaps & Islands problems ask you to find **consecutive sequences** (islands) and the breaks between them (gaps).

#### Sample Table: `login_events`

```sql
user_id | login_date
--------|------------
1       | 2025-01-01
1       | 2025-01-02
1       | 2025-01-03
1       | 2025-01-05
1       | 2025-01-06
1       | 2025-01-09
2       | 2025-01-01
2       | 2025-01-02
2       | 2025-01-04
2       | 2025-01-05
2       | 2025-01-06
2       | 2025-01-07
3       | 2025-01-03
3       | 2025-01-04
3       | 2025-01-08
```

#### Problem 1: Basic Islands (Easy)

Find all consecutive login streaks for each user.  
For every island, show:

- `user_id`
- `start_date`
- `end_date`
- `streak_length` (number of consecutive days)

**Expected Output (example):**
```
user_id | start_date | end_date   | streak_length
--------|------------|------------|---------------
1       | 2025-01-01 | 2025-01-03 | 3
1       | 2025-01-05 | 2025-01-06 | 2
1       | 2025-01-09 | 2025-01-09 | 1
2       | 2025-01-01 | 2025-01-02 | 2
2       | 2025-01-04 | 2025-01-07 | 4
3       | 2025-01-03 | 2025-01-04 | 2
3       | 2025-01-08 | 2025-01-08 | 1
```

#### Problem 2: Longest Streak per User (Medium)

For each user, find their **longest consecutive login streak**.

Output:
- `user_id`
- `longest_streak`

#### Problem 3: Users with Streak ≥ 3 Days (Medium)

Find all users who have at least one consecutive login streak of **3 or more days**.  
Return only the `user_id`s.

#### Problem 4: Gaps Between Logins (Medium)

For each user, find the gaps (missing days) between their logins.

Output columns:
- `user_id`
- `gap_start` (day after previous login)
- `gap_end` (day before next login)
- `gap_days` (number of missing days)

#### Problem 5: Advanced – Islands with Additional Condition (Hard)

Using the same data, find consecutive streaks where the user logged in on **weekdays only** (ignore weekends if present).  
(For simplicity, assume the given dates are the only ones — treat any break as a gap).

More realistic version:  
Find streaks of consecutive days, but only count the island if the average of some metric is above a threshold (you can simulate this with a `score` column if needed).

#### Solutions

**Solution 1: Basic Islands**
```sql
WITH numbered AS (
    SELECT 
        user_id,
        login_date,
        login_date - INTERVAL '1 day' * (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) - 1) AS island_id
    FROM login_events
)
SELECT 
    user_id,
    MIN(login_date) AS start_date,
    MAX(login_date) AS end_date,
    COUNT(*) AS streak_length
FROM numbered
GROUP BY user_id, island_id
ORDER BY user_id, start_date;
```

**Alternative (cleaner for dates):**
```sql
WITH cte AS (
    SELECT 
        user_id,
        login_date,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn,
        login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) * INTERVAL '1 day' AS grp
    FROM login_events
)
SELECT 
    user_id,
    MIN(login_date) AS start_date,
    MAX(login_date) AS end_date,
    COUNT(*) AS streak_length
FROM cte
GROUP BY user_id, grp
ORDER BY user_id, start_date;
```

**Solution 2: Longest Streak per User**
```sql
WITH islands AS (
    SELECT 
        user_id,
        login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) * INTERVAL '1 day' AS grp,
        COUNT(*) AS streak_length
    FROM login_events
    GROUP BY user_id, login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) * INTERVAL '1 day'
)
SELECT 
    user_id,
    MAX(streak_length) AS longest_streak
FROM islands
GROUP BY user_id;
```

**Solution 3: Users with Streak ≥ 3**
```sql
WITH islands AS (
    SELECT 
        user_id,
        COUNT(*) AS streak_length
    FROM (
        SELECT 
            user_id,
            login_date - ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) * INTERVAL '1 day' AS grp
        FROM login_events
    ) t
    GROUP BY user_id, grp
)
SELECT DISTINCT user_id
FROM islands
WHERE streak_length >= 3;
```

**Solution 4: Gaps Between Logins**
```sql
WITH ordered AS (
    SELECT 
        user_id,
        login_date,
        LEAD(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS next_login
    FROM login_events
)
SELECT 
    user_id,
    login_date + INTERVAL '1 day' AS gap_start,
    next_login - INTERVAL '1 day' AS gap_end,
    (next_login - login_date - 1) AS gap_days
FROM ordered
WHERE next_login - login_date > 1
ORDER BY user_id, gap_start;
```

#### Key Pattern to Remember

```sql
date_column - ROW_NUMBER() OVER (PARTITION BY ... ORDER BY date_column) * INTERVAL '1 day'
```

This expression stays **constant** within a consecutive island and changes when a gap appears.

---

## Query Writing Best Practices (Interviewers notice these)
- Writing clean, readable multi-step queries (prefer CTEs over deeply nested subqueries)
- Correct handling of NULLs
- Understanding logical query execution order
- Avoiding common mistakes (join inflation, wrong grain, etc.)

## Priority Order for Preparation

| Priority | Topics                                      | Why |
|---------|---------------------------------------------|-----|
| Highest | Joins + Aggregation + Window Functions     | Appear in 70-80% of problems |
| High    | CTEs + Subqueries + Conditional Aggregation| Needed for medium/hard problems |
| Medium  | Ranking patterns, Top-N, Dedup             | Very common in company rounds |
| Lower   | Recursive CTEs, Set operations, Optimization| Asked less frequently |

---