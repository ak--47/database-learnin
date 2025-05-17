# Database Learnin w/AK

An interactive playground for exploring **transactional vs. analytical databases** using a good 'ol fashion **Web App** along with **SQLite** (Transactional) and **DuckDB** (Analytical). 

Recall that *transactional queries* (with SQLite) are all about dealing with individual single rows. `INSERT`, `UPDATE`, `DELETE`, you know... CRUD!

By contrast *analytical queries* (with DuckDB) are all about dealing with the entire dataset, doing things like aggregations, window functions, fast scans... you know... analytics!


---

## 🔧 Prerequisites

- **macOS** (or any Unix-like OS)  
- **Node.js** (v14+) & **npm**  ([download](https://nodejs.org/en/download/))  
- **SQLite CLI** (installed by default on macOS)  
- **DuckDB CLI**  
  ```bash
  # Homebrew
  brew install duckdb

  # —or— direct install script
  curl https://install.duckdb.org | sh
  ```
---

## ▶️ Run the UI

1. **Clone**  
   ```bash
   git clone https://github.com/ak--47/database-learnin.git && cd database-learnin
   ```
2. **Start the dev server**  
   ```bash
   npm run start
   ```
3. **Open** your browser to  
   ```
   http://localhost:3000
   ```
4. In the app, click **Open Database…** and choose (or create) a folder to store your files.  
   > **Tip:** Pick the project root so `database.sqlite` lives alongside your code.

This will create (or open) `database.sqlite` and seed it with data!

---

## 🗄️ Connect with SQLite (Transactional)

Use the built-in SQLite shell to poke at your todo table:

```bash
cd <your-chosen-folder>
sqlite3 database.sqlite
```

At the `sqlite>` prompt:

```sql
.tables             -- list all tables
.schema todos       -- show the 'todos' table schema
SELECT * FROM todos;-- view all todo items
```

Every UI action (add, delete, reorder) is executed as a real SQL `INSERT` / `DELETE` / `UPDATE` on this file.

---

## 📊 Connect with DuckDB (Analytical)

DuckDB shines for analytics. You can query your SQLite file in the DuckDB REPL:

```bash
cd <your-chosen-folder>
duckdb database.sqlite
```

Inside the DuckDB shell:

```sql
-- 1. Install & load the SQLite scanner (only needed once per machine)
INSTALL sqlite;  
LOAD sqlite;     

-- 2. Attach the SQLite file as a virtual schema
ATTACH 'database.sqlite' AS sqldb (FORMAT 'sqlite');

-- 3. List tables (DuckDB + attached schemas)
.tables         

-- 4. Query todos analytically
SELECT content, LENGTH(content) AS len
FROM sqldb.todos
ORDER BY len DESC
LIMIT 5;
```

---

## 🛢 Example CRUD queries from the command line

Here are some simple SQL operations you can run in any SQL shell (duckDB or sqlite); they are used to modify the state of the database:

```sql
-- Insert a new todo
INSERT INTO todos (content, position)
VALUES ('Write documentation', (SELECT MAX(position) + 1 FROM todos));

-- Update an existing todo
UPDATE todos
SET content = 'Review pull request'
WHERE id = 1;

-- Delete a todo
DELETE FROM todos
WHERE id = 5;
```
---

## 🦆 DuckDB vs SQLite 

now you might be asking... why do we need duckdb? how is it different from sqlite?

it's a good question. at a high level, duckdb is a columnar database that is optimized for analytical queries, while sqlite is a row-based database that is optimized for transactional queries. this brings up an important distinction in the wide world of databases:

## 🔄 OLTP vs OLAP

there are two main types of databases: OLTP (Online Transaction Processing) and OLAP (Online Analytical Processing). they are purpose built for different types of queries and workloads.

- **OLTP** (Online Transaction Processing)  
  - Optimized for transactional queries (CRUD)
  - Fast, low-latency operations
  - Handles concurrent writes and row-level locks
  - Examples: SQLite, MySQL, PostgreSQL

- **OLAP** (Online Analytical Processing)
  - Optimized for analytical queries (aggregations, scans)
  - Fast, large dataset processing
  - Handles complex queries and rich analytical expressions
  - Examples: DuckDB, BigQuery, Snowflake

### ✅ Queries You Can **Only Do in OLTP** (SQLite, MySQL)

These rely on row-level operations, low latency, and high write consistency — typical of a transactional system.

#### 1. Insert a Purchase Order with a Foreign Key

```sql
BEGIN TRANSACTION;
INSERT INTO orders (user_id, total) VALUES (123, 49.99);
INSERT INTO order_items (order_id, product_id, quantity)
VALUES (last_insert_rowid(), 456, 2);
COMMIT;
```

> 🧠 *Why OLTP?* You’re inserting dependent rows, in a precise order, and need rollback support if one insert fails.

---

#### 2. Concurrent Record Locking / Isolation

```sql
BEGIN IMMEDIATE; -- locks the database
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 999;
COMMIT;
```

> 🧠 *Why OLTP?* OLTP engines handle concurrency and row-level locks; DuckDB doesn't handle concurrent writes.

---

#### 3. Real-time User Authentication

```sql
SELECT * FROM users WHERE email = 'ak@mixpanel.com' AND password_hash = '...';
```

> 🧠 *Why OLTP?* Low-latency, point lookup. Meant to serve a web request or mobile app instantly.

---

### 📊 Queries You Can **Only Do in OLAP** (DuckDB, BigQuery, etc.)

These leverage fast scanning, columnar reads, large dataset processing, and rich analytical expressions.

#### 1. Window Functions Over the Full Table

```sql
SELECT
  id,
  content,
  ROW_NUMBER() OVER (ORDER BY created_at DESC) AS recent_rank
FROM sqldb.todos;
```

> 🧠 *Why OLAP?* OLTP databases support limited window functions and don't optimize full-table scans well. this means that concepts like "sessions" are slower and more expensive on OLTP

---

#### 2. Joining Large Datasets with On-Demand Ad-Hoc Analysis

```sql
SELECT
  t.user_id,
  AVG(LENGTH(t.content)) AS avg_length
FROM sqldb.todos t
JOIN parquet_scan('logs/*.parquet') logs
  ON t.user_id = logs.user_id
GROUP BY t.user_id
ORDER BY avg_length DESC;
```

> 🧠 *Why OLAP?* Ad-hoc joins across large files (e.g., Parquet) aren't possible in OLTP engines. this means concepts like "user profiles" aren't possible in OLTP

---

### 3. Time-Series Aggregations

```sql
SELECT
  strftime(created_at, '%Y-%m') AS month,
  COUNT(*) AS todos_created
FROM sqldb.todos
GROUP BY month
ORDER BY month;
```

> 🧠 *Why OLAP?* DuckDB handles large time-based aggregation with ease; OLTP databases are slow and expensive on double and triple GROUP BYs

---

#### 4. Complex Aggregates Over Billions of Rows

```sql
SELECT
  content,
  COUNT(*) AS frequency
FROM sqldb.todos
GROUP BY content
ORDER BY frequency DESC
LIMIT 10;
```

> 🧠 *Why OLAP?* OLTP can’t scan large datasets this efficiently or support columnar optimizations.

---

### 🧠 Summary Comparison

| Task                                | OLTP (SQLite/MySQL) | OLAP (DuckDB/BigQuery)       |
|-------------------------------------|----------------------|-------------------------------|
| Insert a new row (form submission)  | ✅ Yes               | 🚫 No (not designed for writes) |
| Transactional safety (rollback)     | ✅ Yes               | 🚫 Not supported              |
| Real-time query (by ID/email)       | ✅ Yes               | ⚠️ Inefficient                |
| Full-table scan w/ window functions | 🚫 Inefficient       | ✅ Fast                       |
| Time-series reporting (e.g. trends) | 🚫 Painful           | ✅ Designed for it            |
| Joining large datasets              | 🚫 Not viable        | ✅ Easy & fast                |

---

that is all the theory; now let's get into some practical examples of how to use duckdb and sqlite for both transactional and analytical queries in our own database.

## 💾 Example Transactional Queries

Run these transactional queries which modify the state of the database in SQLite. They will not work in DuckDB!


```sql
-- Batch insert by placing multiple rows in a single transaction
BEGIN TRANSACTION;
INSERT INTO todos (content, position) VALUES
  ('Batch task 1', (SELECT MAX(position)+1 FROM todos)),
  ('Batch task 2', (SELECT MAX(position)+2 FROM todos)),
  ('Batch task 3', (SELECT MAX(position)+3 FROM todos));
COMMIT;
```

```sql
-- Start a transaction but allow rollbacks
BEGIN TRANSACTION;

-- Change A
UPDATE todos SET position = position + 1 WHERE id = 1;

SAVEPOINT midway;   -- mark this spot

-- Change B
UPDATE todos SET position = position - 1 WHERE id = 2;

-- Oops—don’t like Change B?
ROLLBACK TO midway;   -- undoes Change B but keeps Change A

-- Finalize
COMMIT;
```

---

## 📈 Example Analytical Queries

Here are some example analytical queries you can run in DuckDB, which are all about aggregating data and getting insights from the entire dataset. They _might_ work in SQLite, but they are not optimized for it:

```sql
-- Count total todos
SELECT COUNT(*) AS total_todos
FROM sqldb.todos;

-- Bucket by length
SELECT len_bucket, COUNT(*) AS cnt
FROM (
  SELECT CASE
           WHEN LENGTH(content) <= 20 THEN '<=20'
           WHEN LENGTH(content) <= 40 THEN '21–40'
           ELSE '>40'
         END AS len_bucket
  FROM sqldb.todos
)
GROUP BY len_bucket;
```

---

## .

i believe understanding the different use cases of OLAP and OLTP are essential learning. i hope it is clear that the first databases were OLTP. OLAP emerged as new analytical use cases (like product analytics) become more important to organizations. (mixpanel's ARB is OLAP database)

the last thing i would like you to reflect on is that you have now had the experience of dealing with raw data FILES and also  dealing with TABLES (an abstraction on that file). 

data engineers usually deal with FILES and created them into TABLES, whereas data analysts work on the TABLES and turn them into DASHBOARDS. and that's how BI works at like 95% of companies.

but importantly, metrics on dashboards are really just SQL queries on top of tables which are really just files (which are really just bytes on disk ... ones and zeros). but it's sufficiently complicated enough that you need a whole team of technical people to do it! most of the time, this team is underappreciated and underpaid and under water. if we can make it easier for them, we can add value to the whole company.

i deeply appreciate and value your time. i hope you enjoyed this adventure with me! i am happy to hear what you thought about this activity! ak@mixpanel.com


## 🛠 Troubleshooting

- **`IO Error: extension ".../sqlite_scanner.duckdb_extension" not found`**  
  Run inside DuckDB:
  ```sql
  INSTALL sqlite;
  LOAD sqlite;
  ```
- **Cannot open database**  
  Make sure you’re in the folder containing `database.sqlite`, and that you passed the correct filename to `duckdb`.
---

