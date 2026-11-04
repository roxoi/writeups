---
title: "Module 5: Databases & the Data Layer (PostgreSQL, SQLite, Redis)"
description: "Connect Node.js and Bun to PostgreSQL with safe parameterized queries, connection pools and transactions; write migrations; compare raw SQL, query builders and ORMs; avoid N+1 queries and race conditions; and add Redis caching."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/nodejs-module-5.png"
tags: [Node.js, Bun, PostgreSQL, SQL, Migrations, Redis, Interview-Prep]
keywords: ["Node.js PostgreSQL tutorial", "pg Pool transactions", "SQL injection parameterized queries", "N+1 query problem", "Node.js database migrations", "Bun.sql", "node:sqlite", "Drizzle vs Prisma", "Redis cache aside"]
---

# Module 5: Databases & the Data Layer (PostgreSQL, SQLite, Redis)

![Node.js Module 5](/images/nodejs-module-5.png)

## Words you will meet in this module

| Word                          | Simple meaning                                                                                      |
| ----------------------------- | --------------------------------------------------------------------------------------------------- |
| **Relational database**       | Data in tables with rows and columns, linked by keys. Queried with SQL. (PostgreSQL, MySQL, SQLite) |
| **Primary key / foreign key** | A column that uniquely identifies a row / a column pointing at another table's primary key.         |
| **Parameterized query**       | A query whose values are sent **separately** from the SQL text. The defense against SQL injection.  |
| **Connection pool**           | A small set of reusable database connections shared by all requests.                                |
| **Transaction**               | A group of statements that **all succeed or all fail**. (ACID)                                      |
| **Migration**                 | A versioned script that changes the database schema in a controlled, repeatable way.                |
| **ORM**                       | Object-Relational Mapper: a library that maps tables to objects/types and generates SQL for you.    |
| **N+1 problem**               | Running one query for a list, then one more query **per item**. Slow.                               |
| **Index**                     | A data structure that makes lookups on a column fast, at the cost of slower writes.                 |
| **Race condition**            | Two operations interleave and produce a wrong result (a **lost update**).                           |
| **Optimistic locking**        | Detecting concurrent edits with a `version` column. The second writer fails and retries.            |
| **Cache-aside**               | Read the cache first, on a miss read the database and fill the cache.                               |

## 5.1 Choosing Where Data Lives

| Store                   | Type                | Best for                                                         | Notes                                            |
| ----------------------- | ------------------- | ---------------------------------------------------------------- | ------------------------------------------------ |
| **PostgreSQL**          | Relational (SQL)    | Almost everything: users, orders, tasks, reporting, JSON columns | **The default choice for new apps**              |
| **MySQL / MariaDB**     | Relational          | Web apps, existing ecosystems                                    | Similar concepts                                 |
| **SQLite**              | Embedded relational | Local dev, tests, small apps, mobile, edge, CLIs                 | A single file. One writer at a time              |
| **MongoDB**             | Document            | Flexible, nested, rapidly changing documents                     | Joins and cross-document transactions are harder |
| **Redis**               | In-memory key-value | Cache, sessions, rate limits, queues, pub/sub                    | Fast but not your system of record               |
| **S3 / object storage** | Blobs               | Files, images, exports                                           | Never store big files in the database            |

**Rule of thumb:** start with **PostgreSQL** as the source of truth, add **Redis** when you have a measured need (cache, rate limits, queues). See [System Design: Databases](system-design/databases) for SQL vs NoSQL trade-offs, and the [DBMS interview handbook](database/database-management-systems-interview-handbook) for ACID, normalization and indexing theory.

## 5.2 SQL Essentials for an API

This module assumes you know the basics. If you want depth, work through the [SQL query patterns handbook](database/sql-advanced-query-patterns-tutorial). The schema for TaskFlow:

```sql
-- migrations/001_init.sql
CREATE TABLE projects (
  id          INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name        TEXT        NOT NULL CHECK (length(name) BETWEEN 1 AND 100),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tasks (
  id          INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  project_id  INTEGER     NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  title       TEXT        NOT NULL CHECK (length(title) BETWEEN 1 AND 200),
  status      TEXT        NOT NULL DEFAULT 'todo' CHECK (status IN ('todo', 'doing', 'done')),
  version     INTEGER     NOT NULL DEFAULT 1,            -- for optimistic locking
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX tasks_project_status_idx ON tasks (project_id, status);

INSERT INTO projects (name) VALUES ('Website redesign'), ('Mobile app');
```

### The four statements you will write daily

```sql
SELECT id, title, status FROM tasks WHERE project_id = 1 AND status = 'todo' ORDER BY id LIMIT 20;

INSERT INTO tasks (project_id, title) VALUES (1, 'Buy milk') RETURNING id, title, status;   -- RETURNING gives the new row back

UPDATE tasks SET status = 'done', updated_at = now() WHERE id = 7 RETURNING *;

DELETE FROM tasks WHERE id = 7;

-- A join and an aggregate
SELECT p.id, p.name, count(t.id) AS open_tasks
  FROM projects p LEFT JOIN tasks t ON t.project_id = p.id AND t.status <> 'done'
 GROUP BY p.id, p.name;
```

### Design rules that save you later

- Let the database enforce rules: `NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`. Application code has bugs. Constraints do not.
- Store time as **`timestamptz`** (UTC). Store money as **integer cents** or `NUMERIC`, never floats.
- Use an integer identity or a UUID as the primary key. Do not use email or names as keys.
- Index columns you **filter, join and sort by**, especially foreign keys.

## 5.3 PostgreSQL from Node and Bun

### Run Postgres locally with Docker

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: taskflow
    ports: ['5432:5432']
    volumes: [pgdata:/var/lib/postgresql/data]
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U app -d taskflow']
      interval: 5s
      timeout: 3s
      retries: 10
  redis:
    image: redis:7-alpine
    ports: ['6379:6379']
volumes:
  pgdata: {}
```

```bash
docker compose up -d
# DATABASE_URL=postgres://app:secret@localhost:5432/taskflow
```

(Any currently supported Postgres version works. Docker itself is covered in the [Docker tutorial](devops/docker/docker-full-tutorial).)

### The `pg` driver and a connection pool

```bash
npm install pg
npm install -D @types/pg
```

```ts
// src/db/pool.ts
import pg from 'pg';
import { env } from '../env.ts';

export const pool = new pg.Pool({
  connectionString: env.DATABASE_URL,
  max: 10, // maximum connections THIS process may hold
  idleTimeoutMillis: 30_000, // close idle connections after 30 s
  connectionTimeoutMillis: 5_000, // fail instead of waiting forever for a free connection
});

pool.on('error', (err) => console.error('Unexpected idle client error', err));
```

**Why a pool?** Opening a database connection is slow (a TCP + TLS + auth handshake) and the database can only handle a limited number. A pool opens a few connections once and **lends** them to queries. `pool.query()` borrows one, runs the query and returns it immediately.

### Parameterized queries: the #1 rule

```ts
// SAFE: values travel separately from the SQL text. $1, $2 are placeholders
const { rows } = await pool.query(
  'SELECT id, title, status FROM tasks WHERE project_id = $1 AND status = $2',
  [projectId, status],
);

// VULNERABLE: string concatenation or template literals into SQL
await pool.query(`SELECT * FROM tasks WHERE title = '${userInput}'`);
// userInput = "x'; DROP TABLE tasks; --"   → SQL INJECTION
```

With placeholders, the database receives the **query structure** and the **values** on separate channels, so a value can never be interpreted as SQL. **Placeholders work only for values**, not for table or column names. If you must pick a column dynamically (sorting), map user input through a **whitelist**:

```ts
const SORTABLE = { id: 'id', title: 'title', created: 'created_at' } as const;
const column = SORTABLE[req.query.sort as keyof typeof SORTABLE] ?? 'id'; // never use the raw input
await pool.query(`SELECT * FROM tasks ORDER BY ${column} LIMIT $1`, [limit]);
```

### Transactions

When several statements must succeed or fail **together**, use one dedicated connection and `BEGIN` / `COMMIT` / `ROLLBACK`. This is a helper you will reuse everywhere:

```ts
// src/db/transaction.ts
import type pg from 'pg';
import { pool } from './pool.ts';

export async function withTransaction<T>(
  fn: (client: pg.PoolClient) => Promise<T>,
): Promise<T> {
  const client = await pool.connect(); // borrow ONE connection for the whole transaction
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release(); // ALWAYS give it back, or the pool runs dry
  }
}
```

**You cannot run a transaction through `pool.query`.** Each `pool.query` call may use a different connection, and a transaction lives on one connection.

### Bun: the built-in SQL client

Bun ships a Postgres client with a tagged-template API. Interpolated values are sent as **parameters automatically**, so this is safe:

```ts
import { sql } from 'bun'; // reads DATABASE_URL by default

const tasks = await sql`
  SELECT id, title, status FROM tasks
  WHERE project_id = ${projectId} AND status = ${status}
`;

await sql.begin(async (tx) => {
  // transaction
  const [project] =
    await tx`INSERT INTO projects (name) VALUES (${name}) RETURNING id`;
  await tx`INSERT INTO tasks (project_id, title) VALUES (${project.id}, 'Kick-off')`;
});
```

(`postgres` (postgres.js) is a popular Node package with the same tagged-template style. Bun's SQL client has also grown support for other databases in recent releases. Check the Bun docs for your version.)

### Mapping errors from the database

Postgres reports violations as error codes. Translate them into meaningful API errors:

```ts
function mapDbError(err: any): never {
  if (err.code === '23503')
    throw new AppError('Referenced resource not found', 404, 'NOT_FOUND'); // foreign_key_violation
  if (err.code === '23505')
    throw new AppError('Already exists', 409, 'CONFLICT'); // unique_violation
  if (err.code === '23514')
    throw new AppError('Value not allowed', 422, 'VALIDATION_ERROR'); // check_violation
  throw err;
}
```

#### Interview traps

- "How do you prevent SQL injection?" **Parameterized queries** (placeholders), with whitelists for identifiers. Escaping by hand is fragile. ORMs and query builders parameterize for you, but raw fragments reintroduce the risk.
- "Why does my transaction not roll back?" You ran the statements through `pool.query` (different connections) instead of one `client`. Or you forgot `ROLLBACK` on error.
- "Why does my app hang after some traffic?" Pool exhaustion: a path that never calls `client.release()` (no `finally`), or long-running queries holding all connections.
- "Why is `COUNT(*)` a string in JavaScript?" `pg` returns 64-bit integers (`BIGINT`, and `count()`) as **strings** to avoid precision loss. Cast with `::int` in SQL when safe, or use `BigInt` / `Number` deliberately.

## 5.4 SQLite: The Embedded Database

SQLite is a full SQL database in **one file**, with no server process. It is excellent for tests, local development, small tools and edge deployments. It allows **many readers but one writer at a time**, so it is not the first choice for a write-heavy multi-instance API.

### `node:sqlite` (built into Node.js)

```ts
import { DatabaseSync } from 'node:sqlite';

const db = new DatabaseSync(':memory:'); // or "app.db" for a file
db.exec(
  `CREATE TABLE tasks (id INTEGER PRIMARY KEY, title TEXT NOT NULL, done INTEGER DEFAULT 0)`,
);

const insert = db.prepare('INSERT INTO tasks (title) VALUES (?)'); // prepared + parameterized
insert.run('Buy milk');
insert.run('Walk the dog');

const rows = db.prepare('SELECT * FROM tasks WHERE id > ?').all(0);
const one = db.prepare('SELECT * FROM tasks WHERE id = ?').get(1);
```

The module is only available under the `node:` prefix. In Node 24 its stability level is "release candidate", so check the current docs for your Node version before relying on it in production. Its API is synchronous.

### `bun:sqlite`

```ts
import { Database } from 'bun:sqlite';

const db = new Database('app.db');
db.run('PRAGMA journal_mode = WAL'); // better concurrency: readers do not block the writer
const rows = db.query('SELECT * FROM tasks WHERE id > ?').all(0);
```

### Where SQLite shines in this series

Use an **in-memory SQLite** database for fast unit tests of repositories, but run **integration tests against real PostgreSQL** (Module 7), because SQL dialects and behaviors differ (types, `RETURNING`, locking, JSON operators).

## 5.5 Migrations

### The idea in plain words

A **migration** is a numbered script that changes the schema. The database remembers which ones have run. Every environment (your laptop, CI, staging, production) applies the same scripts in the same order, so the schema is **reproducible** and reviewable in Git, like code.

```
 migrations/
   001_init.sql
   002_users_and_sessions.sql     (you will add this in Module 6)
   003_add_task_due_date.sql
```

### A tiny migration runner (about 30 lines)

You can use a tool (see below), but writing this once shows exactly what they do:

```ts
// scripts/migrate.ts
import { readdir, readFile } from 'node:fs/promises';
import path from 'node:path';
import { pool } from '../src/db/pool.ts';

const dir = path.join(import.meta.dirname, '..', 'migrations');
const client = await pool.connect();

try {
  await client.query(`CREATE TABLE IF NOT EXISTS schema_migrations (
    name TEXT PRIMARY KEY, applied_at TIMESTAMPTZ NOT NULL DEFAULT now())`);
  await client.query('SELECT pg_advisory_lock(727274)'); // only one migrator at a time

  const applied = new Set(
    (await client.query('SELECT name FROM schema_migrations')).rows.map(
      (r) => r.name,
    ),
  );
  const files = (await readdir(dir)).filter((f) => f.endsWith('.sql')).sort();

  for (const file of files) {
    if (applied.has(file)) continue;
    console.log('applying', file);
    const sql = await readFile(path.join(dir, file), 'utf8');
    await client.query('BEGIN');
    try {
      await client.query(sql); // the whole file in one transaction
      await client.query('INSERT INTO schema_migrations (name) VALUES ($1)', [
        file,
      ]);
      await client.query('COMMIT');
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    }
  }
  console.log('migrations up to date');
} finally {
  await client.query('SELECT pg_advisory_unlock(727274)').catch(() => {});
  client.release();
  await pool.end();
}
```

Run it with `node --env-file=.env scripts/migrate.ts` (add `"migrate"` to `package.json`). Because Postgres supports **transactional DDL**, a failed migration leaves nothing half-applied.

### Rules of migrations

| Rule                                                       | Why                                                               |
| ---------------------------------------------------------- | ----------------------------------------------------------------- |
| **Never edit a migration that has been applied** elsewhere | Other environments would diverge. Add a **new** migration instead |
| Migrations are **forward-only** in production              | Rollbacks of data changes are rarely safe. Fix forward            |
| Keep them **small** and **deterministic**                  | Easier to review and to re-run                                    |
| Run them as a **deploy step** before the new code starts   | With the **expand / contract** pattern (below)                    |
| Test them on a **copy of production-sized data**           | A one-second change on 100 rows can lock a 100 million-row table  |

**Expand / contract** (zero-downtime renames and changes): to rename a column, (1) _expand_: add the new column and write to both, (2) backfill old rows, (3) deploy code that reads the new column, (4) _contract_: drop the old column in a later migration. Never rename in one step while the old code is still running.

### Tools

| Tool                        | Style                                            |
| --------------------------- | ------------------------------------------------ |
| **node-pg-migrate**         | JS/SQL migrations for Postgres                   |
| **Drizzle Kit**             | Generates migrations from your TypeScript schema |
| **Prisma Migrate**          | Generates migrations from `schema.prisma`        |
| **Flyway / Atlas / dbmate** | Language-agnostic SQL migration tools            |

## 5.6 Raw SQL, Query Builders and ORMs

| Approach               | Example tools                      | You write                           | Strengths                                             | Weaknesses                                                              |
| ---------------------- | ---------------------------------- | ----------------------------------- | ----------------------------------------------------- | ----------------------------------------------------------------------- |
| **Raw SQL**            | `pg`, `postgres`, `Bun.sql`        | SQL strings with parameters         | Full power, no surprises, exactly the query you wrote | No type safety unless you add it. More boilerplate                      |
| **Query builder**      | Kysely, Knex                       | A typed fluent API that mirrors SQL | Type-safe, composable, close to SQL                   | You still think in SQL terms                                            |
| **ORM / schema-first** | Prisma, Drizzle, TypeORM, MikroORM | Models / schema in code             | Typed models, migrations, relations, productivity     | Abstraction leaks. Generated queries can be inefficient. Learning curve |

A short Drizzle example, to show the flavor (the API evolves, so check its docs):

```ts
import { pgTable, serial, text, integer } from 'drizzle-orm/pg-core';
import { drizzle } from 'drizzle-orm/node-postgres';
import { and, eq } from 'drizzle-orm';
import { pool } from './pool.ts';

export const tasks = pgTable('tasks', {
  id: serial('id').primaryKey(),
  projectId: integer('project_id').notNull(),
  title: text('title').notNull(),
  status: text('status').notNull().default('todo'),
});

const db = drizzle(pool);
const todo = await db
  .select()
  .from(tasks)
  .where(and(eq(tasks.projectId, 1), eq(tasks.status, 'todo')));
```

**This series uses `pg` with plain SQL inside small repository functions.** You see exactly what runs, which is the best way to learn, and the skills carry to any tool. In real projects, pick what your team will maintain well. A type-safe layer (Drizzle, Kysely, Prisma) is a very reasonable choice. Whatever you choose, learn to read **`EXPLAIN`** output, because every abstraction eventually needs it.

## 5.7 The Repository Layer with Transactions

A repository takes a **`db`** argument that can be either the **pool** or a **transaction client**. The same function then works both inside and outside a transaction.

```ts
// src/repositories/taskRepo.ts
import type pg from 'pg';
import type { Task } from '../schemas/tasks.ts';

type Db = Pick<pg.Pool | pg.PoolClient, 'query'>;

const COLUMNS = `id, project_id AS "projectId", title, status, version`;

export const taskRepo = {
  async list(
    db: Db,
    projectId: number,
    f: { status?: string; q?: string; limit: number; offset: number },
  ) {
    const params = [projectId, f.status ?? null, f.q ?? null];
    const where = `project_id = $1
                   AND ($2::text IS NULL OR status = $2)
                   AND ($3::text IS NULL OR title ILIKE '%' || $3 || '%')`;

    const items = await db.query<Task>(
      `SELECT ${COLUMNS} FROM tasks WHERE ${where} ORDER BY id LIMIT $4 OFFSET $5`,
      [...params, f.limit, f.offset],
    );
    const total = await db.query<{ total: number }>(
      `SELECT count(*)::int AS total FROM tasks WHERE ${where}`,
      params,
    );
    return { items: items.rows, total: total.rows[0].total };
  },

  async create(db: Db, projectId: number, title: string) {
    const { rows } = await db.query<Task>(
      `INSERT INTO tasks (project_id, title) VALUES ($1, $2) RETURNING ${COLUMNS}`,
      [projectId, title],
    );
    return rows[0];
  },

  async update(db: Db, id: number, patch: { title?: string; status?: string }) {
    const { rows } = await db.query<Task>(
      `UPDATE tasks
          SET title = COALESCE($2, title), status = COALESCE($3, status),
              version = version + 1, updated_at = now()
        WHERE id = $1
        RETURNING ${COLUMNS}`,
      [id, patch.title ?? null, patch.status ?? null],
    );
    return rows[0]; // undefined if no such task
  },

  async remove(db: Db, id: number) {
    const { rowCount } = await db.query('DELETE FROM tasks WHERE id = $1', [
      id,
    ]);
    return (rowCount ?? 0) > 0;
  },
};
```

The service decides **when a transaction is needed**:

```ts
// src/services/projectService.ts: create a project AND its starter tasks atomically
import { withTransaction } from '../db/transaction.ts';

export async function createProjectWithStarterTasks(name: string) {
  return withTransaction(async (client) => {
    const { rows } = await client.query(
      'INSERT INTO projects (name) VALUES ($1) RETURNING id, name',
      [name],
    );
    const project = rows[0];
    for (const title of ['Plan', 'Build', 'Ship']) {
      await taskRepo.create(client, project.id, title); // same connection, same transaction
    }
    return project; // if anything threw, NOTHING was saved
  });
}
```

Use a transaction whenever **two or more writes must succeed together** (create an order and its items, transfer money, move a task and update a counter).

## 5.8 N+1, Indexes, Pool Sizing and Concurrency

### The N+1 query problem

```ts
// 1 query for the projects + N queries for their tasks = N+1 round trips
const projects = (await pool.query('SELECT * FROM projects')).rows;
for (const p of projects) {
  p.tasks = (
    await pool.query('SELECT * FROM tasks WHERE project_id = $1', [p.id])
  ).rows; // runs once per project!
}
```

With 100 projects that is 101 queries, each with network latency. Fix it by fetching related rows **in one query**:

```ts
// 2 queries total
const projects = (await pool.query('SELECT id, name FROM projects')).rows;
const ids = projects.map((p) => p.id);
const { rows: tasks } = await pool.query(
  'SELECT * FROM tasks WHERE project_id = ANY($1::int[])',
  [ids],
);
// group tasks by project_id in JS

// or 1 query: let Postgres assemble the JSON
const { rows } = await pool.query(`
  SELECT p.id, p.name,
         COALESCE(json_agg(t.*) FILTER (WHERE t.id IS NOT NULL), '[]') AS tasks
    FROM projects p LEFT JOIN tasks t ON t.project_id = p.id
   GROUP BY p.id`);
```

ORMs hide this problem well, so many N+1 bugs live in ORM code (lazy-loaded relations in a loop). Log query counts per request in development to catch it.

### Indexes: find out, do not guess

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM tasks WHERE project_id = 1 AND status = 'todo' ORDER BY id LIMIT 20;
```

Look for **`Seq Scan`** (reading the whole table) on big tables versus **`Index Scan`**. Remember:

- Index **foreign keys** and the columns in `WHERE` / `ORDER BY`. Order columns in a composite index by how you filter (equality first).
- Every index slows writes and uses disk. Do not index everything.
- `ILIKE '%text%'` cannot use a normal B-tree index. Use the `pg_trgm` extension with a GIN index, or full-text search.

### Pool sizing

```
 total connections used  =  pool.max  ×  number of API instances      (+ workers, migrations, admin tools)
 PostgreSQL default max_connections ≈ 100
```

If you run 20 instances with `max: 10`, that is 200 connections, which exceeds the limit. Keep the total under the database limit with headroom, and put **PgBouncer** (a connection pooler) in front when you scale out. A **bigger pool is not faster**: Postgres performs best with a modest number of active connections (roughly a small multiple of its CPU cores). Set safeguards on the database side too:

```sql
ALTER ROLE app SET statement_timeout = '5s';                          -- kill runaway queries
ALTER ROLE app SET idle_in_transaction_session_timeout = '15s';       -- kill forgotten transactions
```

### Race conditions: the lost update

```ts
// BROKEN under concurrency: read, modify in JS, write
const task = await taskRepo.get(pool, id); // both requests read version 1
await pool.query('UPDATE tasks SET title = $2 WHERE id = $1', [
  id,
  task.title + '!',
]);
// request A and B both write based on the same old data: one change is silently lost
```

Three fixes, from simplest to heaviest:

**1. Make the change atomic in SQL** (best when possible):

```sql
UPDATE tasks SET status = 'done' WHERE id = $1;             -- no read first
UPDATE accounts SET balance = balance - $2 WHERE id = $1 AND balance >= $2;   -- check and change in one statement
```

**2. Optimistic locking** with the `version` column: the client sends the version it saw, and the update only applies if it is unchanged:

```ts
const { rowCount } = await pool.query(
  `UPDATE tasks SET title = $2, version = version + 1
    WHERE id = $1 AND version = $3`,
  [id, newTitle, versionFromClient],
);
if (rowCount === 0)
  throw new AppError(
    'Task was changed by someone else',
    409,
    'VERSION_CONFLICT',
  );
```

**3. Pessimistic locking**: lock the row while you work on it.

```ts
await withTransaction(async (c) => {
  const { rows } = await c.query(
    'SELECT * FROM tasks WHERE id = $1 FOR UPDATE',
    [id],
  ); // others wait here
  // ...decide and update...
});
```

And a cardinal rule: **never "check then insert"** (`if (!exists) insert`). Two requests pass the check together. Put a **`UNIQUE` constraint** in the database and handle the `23505` error. The database is the only place that can enforce uniqueness safely.

### Isolation in one paragraph

PostgreSQL's default isolation level is **READ COMMITTED**: each statement sees data committed before that statement started. For stricter guarantees there are **REPEATABLE READ** and **SERIALIZABLE** (which can abort transactions with a serialization error that you must retry). Most apps need only the default plus atomic updates, locks and constraints.

#### Interview traps

- "What is the N+1 problem and how do you fix it?" One query for a list plus one per row. Fix with a join, a batched `IN`/`ANY` query, or an ORM's eager loading.
- "Why not just increase the pool size when requests queue?" Postgres handles a limited number of concurrent queries well, and the total across all instances is capped by `max_connections`. Fix slow queries first, then consider a pooler.
- "How do you prevent two users from claiming the last seat?" An atomic conditional `UPDATE ... WHERE seats > 0`, a row lock (`FOR UPDATE`), or a unique constraint, never read-then-write in application code.
- "Where do uniqueness checks belong?" In a `UNIQUE` constraint. Application-level checks race.

#### Tricky questions and answers

###### Q1 [SDE-2]: After a deploy, the API intermittently returns "timeout acquiring connection", yet database CPU is low. What do you check?

**Answer:** Pool exhaustion in the application. Check for code paths that borrow a client (`pool.connect()`) and do not release it on error (missing `finally`), transactions held open across slow external calls (an HTTP request inside a transaction), or a sudden rise in concurrent requests with a small `max`. Look at `pool.totalCount`, `idleCount` and `waitingCount`, set `idle_in_transaction_session_timeout` to surface forgotten transactions, and never perform network I/O while holding a transaction.

###### Q2 [SDE-3]: You must rename `tasks.title` to `tasks.name` with zero downtime and several API instances running different versions during the rollout. How?

**Answer:** Expand/contract. (1) Migration A: add `name` (nullable) and a trigger or dual-write so both stay in sync. (2) Deploy code that writes both columns and still reads `title`. (3) Backfill old rows in batches. (4) Deploy code that reads `name`. (5) Later, a final migration drops `title` and the sync logic. At no step does a running version of the code reference a column that does not exist, which is what a one-step rename would break.

###### Q3 [SDE-2]: Why is `SELECT * FROM tasks WHERE title ILIKE '%milk%'` slow on 10 million rows even with an index on `title`?

**Answer:** A leading wildcard means the database cannot use the sorted order of a B-tree index, so it scans every row. Options: a trigram index (`pg_trgm` with GIN) which supports `ILIKE '%x%'`, Postgres full-text search (`tsvector`) for word-based search, or a dedicated search engine for heavy search workloads.

## 5.9 Redis and Caching

### Why a cache?

Some data is **read far more often than it changes** (a project's name, a user's profile, a product page). Recomputing or re-querying it on every request wastes database capacity and time. A cache keeps recent results in memory. **Redis** is an in-memory key-value server that all your API instances can share.

### The cache-aside pattern

```
 request ──► check Redis ──hit──► return it
                 │
                miss
                 ▼
            query PostgreSQL ──► store in Redis with a TTL ──► return it
```

```bash
npm install redis
```

```ts
// src/cache.ts
import { createClient } from 'redis';
import { env } from './env.ts';

export const redis = createClient({ url: env.REDIS_URL });
redis.on('error', (err) => console.error('Redis error', err));
await redis.connect();

export async function cached<T>(
  key: string,
  ttlSeconds: number,
  load: () => Promise<T>,
): Promise<T> {
  const hit = await redis.get(key);
  if (hit !== null) return JSON.parse(hit) as T;

  const value = await load(); // miss: go to the source of truth
  if (value !== undefined && value !== null) {
    await redis.set(key, JSON.stringify(value), { EX: ttlSeconds }); // ALWAYS set a TTL
  }
  return value;
}

// usage
const project = await cached(`project:${id}`, 60, () =>
  projectRepo.get(pool, id),
);

// invalidate on write
await projectRepo.update(pool, id, patch);
await redis.del(`project:${id}`);
```

> **Bun note:** recent Bun releases include a built-in Redis client (`Bun.redis`). The `redis` and `ioredis` packages also work in Bun.

### Cache problems to know

| Problem                            | What happens                                                            | Mitigation                                                                         |
| ---------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| **Stale data**                     | The database changed, the cache still serves the old value              | Short TTLs, delete or update the key on writes                                     |
| **Cache stampede**                 | A popular key expires and hundreds of requests hit the database at once | Locking or request coalescing, randomized TTLs ("jitter")                          |
| **Caching errors / empty results** | A temporary failure gets cached for a minute                            | Only cache successful results, use short TTLs for negatives                        |
| **Unbounded memory**               | Keys without TTLs fill Redis                                            | Always set TTLs and a `maxmemory` eviction policy                                  |
| **Per-user data in a shared key**  | One user sees another's data                                            | Include the user or tenant in the key. Never cache private data under a public key |

"There are only two hard things in computer science: cache invalidation and naming things." Cache only what you have measured to be a bottleneck. See [Caching and CDN](system-design/caching-and-cdn) for eviction policies, write-through vs. write-back and CDNs. Redis also powers **sessions** (Module 6), **rate limiting** and **queues** (Module 7).

#### Interview traps

- "What is cache-aside?" The application checks the cache, falls back to the database on a miss, and fills the cache. The cache is never the source of truth.
- "How do you avoid stale cache?" TTLs plus explicit invalidation on writes, and accept that caching trades freshness for speed.
- "Why must every cached key have a TTL?" So stale or orphaned entries expire and memory stays bounded.

---

## Hands-On Lab: Move TaskFlow to PostgreSQL

```
1. Start Postgres and Redis with the docker-compose.yml above. Check with
   docker compose exec db psql -U app -d taskflow -c "\dt"
2. Add DATABASE_URL and REDIS_URL to src/env.ts and .env.
3. Create migrations/001_init.sql and scripts/migrate.ts. Run it twice.
   The second run must apply nothing. Look at the schema_migrations table.
4. Create src/db/pool.ts and src/db/transaction.ts. Rewrite taskRepo to use
   SQL with the Db parameter. Keep taskService and the routes unchanged except
   for awaiting the now-async calls. (This is why we separated layers.)
5. Map database errors with mapDbError (foreign key → 404, unique → 409).
   POST a task to project 999. Do you get a clean 404, not a 500?
6. Implement createProjectWithStarterTasks with withTransaction. Force a failure
   on the 3rd insert (throw) and prove that no project or tasks were saved.
7. Add the version column check to PATCH /api/tasks/:id (client sends `version`).
   Write a script that fires two PATCH requests with the same version at the same
   time with Promise.all. Exactly one must succeed and one must get 409.
8. Create the N+1 version of "projects with their tasks" and log the number of
   queries. Replace it with the json_agg query. Compare latency with 200 projects.
9. Run EXPLAIN ANALYZE on the task list query with 1,000,000 rows
   (use generate_series to insert them). Drop and re-create the index and compare.
10. Add the cached() helper around GET /api/projects. Check hits and misses with
    redis-cli MONITOR. Invalidate on project writes.
11. Repeat step 4 with Bun.sql for one repository function. Which style do you prefer?
```

Questions to answer for yourself:

- Which rules are enforced by the database and which only by your code? Which would you want to move?
- If you ran 5 API instances, what must `max` be so the total stays under 100 connections?
- What would happen to your cache if two instances updated the same project at the same time?

---

## Module 5 Cheat Sheet

| Concept                   | One-line interview answer                                                                                      |
| ------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Default database          | PostgreSQL as the system of record. Add Redis for cache, rate limits and queues when measured.                 |
| Parameterized query       | Values sent separately from SQL (`$1`). Prevents SQL injection. Whitelist identifiers.                         |
| Connection pool           | A few reusable connections shared by requests. Total across instances must fit `max_connections`.              |
| `pool.query` vs. `client` | `pool.query` may use any connection. A transaction needs one dedicated `client`.                               |
| Transaction               | `BEGIN` / `COMMIT` / `ROLLBACK` on one connection. Always `release()` in `finally`.                            |
| BIGINT / count in `pg`    | Returned as strings. Cast to `int` or convert deliberately.                                                    |
| Constraint codes          | `23505` unique, `23503` foreign key, `23514` check. Map them to 409 / 404 / 422.                               |
| SQLite                    | One-file embedded DB (`node:sqlite`, `bun:sqlite`). Many readers, one writer. Great for tests and small tools. |
| Migration                 | Versioned, forward-only schema scripts tracked in a table. Never edit an applied one.                          |
| Expand / contract         | Add new, dual-write, backfill, switch reads, drop old. Zero-downtime schema changes.                           |
| Raw SQL vs. ORM           | Raw: control, more boilerplate. Query builder: typed SQL. ORM: productivity, hidden queries.                   |
| Repository layer          | Takes `db` (pool or client), so the same function works with or without a transaction.                         |
| N+1                       | One list query plus one query per row. Fix with a join, `ANY($1)` batching or eager loading.                   |
| Index                     | Speeds reads, slows writes. Index FKs and filter/sort columns. Verify with `EXPLAIN ANALYZE`.                  |
| `ILIKE '%x%'`             | Cannot use a B-tree. Use `pg_trgm`/GIN or full-text search.                                                    |
| Lost update               | Read-modify-write races. Fix: atomic SQL, optimistic `version` check, or `FOR UPDATE`.                         |
| Check-then-insert         | Racy. Use a `UNIQUE` constraint and handle the violation.                                                      |
| Cache-aside               | Read cache → on miss read DB → set with TTL. Invalidate on write.                                              |
| Cache hazards             | Stale data, stampede, cached errors, unbounded memory, leaking per-user data.                                  |
