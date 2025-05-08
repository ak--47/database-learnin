# Database Learnin w/AK

An interactive playground for exploring transactional vs. analytical databases using a good 'ol fashion **Web App** along with **SQLite** (Transactional) and **DuckDB** (Analytical)

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

## 🛢 Example CRUD queries

Here are some simple SQL operations you can run in any SQL shell; they are used to modify the state of the database:

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

## 💾 Example Transactional Queries

Here are some transactional queries which modify the state of the database, but do so in a more verbose way allowing for more complex operations:


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

Here are some example analytical queries you can run in DuckDB, which are all about aggregating data and getting insights from the entire dataset:

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

