# 🧪 PostgreSQL Partitioning Demo (Using Docker)

In this demo, we'll spin up a PostgreSQL instance using Docker 🐳, generate a large dataset (10 million rows 📊), and prepare for partitioning. Let’s get started!

---

## 🧰 Requirements

- ✅ Docker installed
- ❌ No need to install PostgreSQL manually
- ⚙️ All setup is done inside a Docker container

---

## 🐳 Step 1: Run PostgreSQL in Docker

Run the following command in your terminal:

```bash
docker run \
  --name pg-main \
  -d \
  -e POSTGRES_PASSWORD=postgres \
  postgres
```
- -name pg-main: Name your container 🏷️
- -d: Run in detached mode 🧵
- -e POSTGRES_PASSWORD=postgres: Set the Postgres password 🔐
- postgres: Image name 🐘

## 🖥️ Step 2: Enter the Container
- Run:
```bash
docker exec -it pg-main 
````
- Then, inside the container:
```
psql -U postgres
```
- You're now inside the PostgreSQL CLI 🐘.

## 🏗️ Step 3: Create the Table

- Create a simple table to hold student grades:

```sql
CREATE TABLE grades_original (
id SERIAL NOT NULL,
g  INTEGER NOT NULL
);
```
> 🎯 Note: Columns used for partitioning must be NOT NULL.

## 🔄 Step 4: Insert 10 Million Rows

- Insert 10M rows with randomly generated grades (0–99):

```sql
INSERT INTO grades_original (g)
SELECT FLOOR(random() * 100)
FROM generate_series(1, 10000000);
```
- 🧠 We're using random() * 100 to simulate grades between 0 and 99, and generate_series() to produce 10 million rows.
- ⏳ This will take a few moments...

## 🗂️ Step 5: Create an Index
- Add an index on the g column for faster query performance:
```sql
CREATE INDEX grades_original_g_idx
ON grades_original (g);
```
> 📈 Indexing helps speed up queries, especially when scanning/filtering rows by g.

## 🧾 Step 6: Describe the Table
- To confirm everything looks good:

```sql
\d grades_original
```
- You should see:
- id: serial, not null
- g: integer, not null
- Index: grades_original_g_idx on column g


# 🔍 Query Performance and Execution Plans in PostgreSQL

Now that we’ve inserted 10 million rows into `grades_original`, let’s run some queries and understand how PostgreSQL processes them using `EXPLAIN ANALYZE`.

---

## 🔢 Query 1: Exact Match

```sql
SELECT COUNT(*) FROM grades_original WHERE g = 30;
```

## 🧾 Result:
- Around 10,000 rows match the condition g = 30.
- Fast execution due to index usage.
- 🔬 Using EXPLAIN ANALYZE:
> EXPLAIN ANALYZE SELECT COUNT(*) FROM grades_original WHERE g = 30;

## 📊 Output Insights:
- Execution Time: ~2000 ms (2 seconds) ⏱️
- Planning Time: ~0.97 ms
- Query Plan: Bitmap Index Scan
- PostgreSQL uses the index to find matching rows quickly.
- Then it visits the actual table to retrieve the rows.

## 💡 Bitmap Index Scan is efficient for selective queries but can become slower with high data spread.

## 🔢 Query 2: Range Query
- SELECT COUNT(*) FROM grades_original WHERE g BETWEEN 30 AND 35;
- 🔬 Using EXPLAIN ANALYZE:
> EXPLAIN ANALYZE SELECT COUNT(*) FROM grades_original WHERE g BETWEEN 30 AND 35;

## 📊 Output Insights:
- Execution Time: ~3000 ms (3 seconds) ⏱️
- Query Plan: Parallel Index Scan
- PostgreSQL chose parallel execution to scan the index across multiple threads.
- Then it combined the results to get the final count.

> ⚖️ Range queries can take longer depending on the number of matching rows and how data is distributed.

## 🧠 Summary

| Query Type        | Index Used?           | Execution Time | Plan Type                  |
| ----------------- | --------------------- | -------------- | -------------------------- |
| `g = 30`          | ✅ Bitmap Index Scan   | \~2 sec        | Efficient for point lookup |
| `g BETWEEN 30–35` | ✅ Parallel Index Scan | \~3 sec        | Efficient for small ranges |

- PostgreSQL chooses different execution plans depending on the query type and data distribution.
- Even with indexes, larger data ranges or less selective queries can take longer.
- Use EXPLAIN ANALYZE to understand and optimize your queries.

# 🧪 PostgreSQL Partitioning Demo

## 🎯 Goal
Implement **range-based partitioning** in PostgreSQL for a table of grades.

---

## 📘 Step 1: Create the Partitioned Table

```sql
CREATE TABLE grades_parts (
  id SERIAL NOT NULL,
  g INT NOT NULL
) PARTITION BY RANGE (g);

```
> 💡 The table is partitioned by the column g. No actual ranges are defined yet — only the structure.

## 📘 Step 2: Create Partition Tables
- We now create 4 partitions manually using the same structure as the parent table.
```sql
CREATE TABLE g0035 (
  LIKE grades_parts including indexes
);

CREATE TABLE g3560 (
  LIKE grades_parts including indexes
);

CREATE TABLE g6080 (
  LIKE grades_parts including indexes
);

CREATE TABLE g80100 (
  LIKE grades_parts including indexes
);
```

> 📝 We're using LIKE to copy the structure from the parent table. No indexes yet — we’ll add them later.

## 📘 Step 3: Attach Partitions to the Parent Table

Now, attach each partition using the ALTER TABLE ... ATTACH PARTITION command:

```sql
ALTER TABLE grades_parts
ATTACH PARTITION g0035 FOR VALUES FROM (0) TO (35);

ALTER TABLE grades_parts
ATTACH PARTITION g3560 FOR VALUES FROM (35) TO (60);

ALTER TABLE grades_parts
ATTACH PARTITION g6080 FOR VALUES FROM (60) TO (80);

ALTER TABLE grades_parts
ATTACH PARTITION g80100 FOR VALUES FROM (80) TO (100);
```

> 🔗 Each gxy partition now contains the specified range of values for g.

## 📋 Partition Summary
| Partition Name | Range |
| -------------- | ----- |
| `g_0_35`       | 0–34  |
| `g_35_60`      | 35–59 |
| `g_60_80`      | 60–79 |
| `g_80_100`     | 80–99 |

> 🧹 Note: At this point, all partitions are empty. No data has been inserted yet.

## 🔍 Table Structure Verification
- You can verify partition attachment like this:
```sql
\d+ g_80_100
```
- Will return:
> Partition of: grades_parts FOR VALUES FROM (80) TO (100)
> 🧠 Also note: Neither the parent nor the child tables have indexes yet.

## 🧠 Final Notes
- PostgreSQL doesn't auto-create partitions — you must define and attach them manually.
- This setup helps optimize queries by limiting access to relevant partitions only.
- You can later add indexes to partitions individually based on your query patterns.

# 🧪 PostgreSQL Partitioning – Inserting & Indexing Demo

---

## 📥 Step 4: Insert Data Into the Partitioned Table

We’ll now insert the **10 million rows** from the original table into the partitioned one:

```sql
INSERT INTO grades_parts
SELECT * FROM grades_original;
```
> ✅ PostgreSQL automatically routes each row to the correct partition based on the value of g.

- How Partition Routing Works
```text
If g = 20 → goes to partition g_0_35

If g = 50 → goes to partition g_35_60

If g = 75 → goes to partition g_60_80

If g = 90 → goes to partition g_80_100
```
> 💡 PostgreSQL handles the routing internally — no need to write custom logic for it.

## 📊 Step 5: Verify Data Distribution
- ✅ Total Rows in Parent Table
```sql
SELECT COUNT(*) FROM grades_parts;
-- Output: 10,000,001 (including that extra 1 we inserted earlier)
```
> 📦 Check Rows in Each Partition
```sql
SELECT COUNT(*) FROM g_0_35;      -- ~3 million rows
SELECT COUNT(*) FROM g_35_60;     -- ~2 million rows
SELECT COUNT(*) FROM g_60_80;     -- ~2.5 million rows
SELECT COUNT(*) FROM g_80_100;    -- ~2.5 million rows
```
> 📈 These values depend on random distribution, so they’ll vary slightly every run.

> 🧾 Validate Partition Boundaries
```sql
SELECT MAX(g) FROM g_0_35;        -- Expect: 34
SELECT MAX(g) FROM g_35_60;       -- Expect: 59
SELECT MAX(g) FROM g_60_80;       -- Expect: 79
SELECT MAX(g) FROM g_80_100;      -- Expect: 99
```
> 🧠 PostgreSQL strictly respects the defined ranges for partitions — no overlapping or invalid data.

## ⚙️ Step 6: Index the Partitioned Table
- Before PostgreSQL 11, indexes had to be manually created on each partition.
- From PostgreSQL 11+, you can simply index the parent table, and PostgreSQL will create the indexes on all partitions automatically.
```sql
CREATE INDEX grades_parts_index
ON grades_parts (g);
```
> ⚡ This creates identical indexes on all child tables under the hood.

## 🔍 Verify Indexes on Partitions
- Check on g_0_35
```sql
\d g_0_35
```
- You'll see: "grades_parts_index" or similar auto-generated index

- Check on g_35_60
```sql
\d g_35_60
```
- Index is present as well
> 📦 Even though the parent table is empty, PostgreSQL manages the index distribution across all partitions.

## 🧠 Final Notes
- ✅ Automatic routing + indexing makes partitioning much easier in modern PostgreSQL.
- 🧹 You don’t need to micromanage each partition — Postgres takes care of it.
- ⚡ Always index your partition key if you’ll be filtering by it in queries

# 🚀 PostgreSQL Partitioning – Querying, Indexes & Partition Pruning

---

## 📈 Performance Check: Querying the Partitioned Table

### ✅ Run an EXPLAIN ANALYZE on the partitioned table

```sql
EXPLAIN ANALYZE
SELECT COUNT(*) FROM grades_parts WHERE g = 30;
```

- The planner uses Index Scan only on one partition (🎯 g_0_35)
- Took about 1 second
- Query is efficient – only one small partition is touched!

## 🤔 Comparison with Non-Partitioned Table
```sql
EXPLAIN ANALYZE
SELECT COUNT(*) FROM grades_original WHERE g = 30;
```
- Also uses an index scan
- Similar execution time (~1 sec)

## 🧠 Why no big difference?
- Because the system (e.g., your laptop with 16GB RAM) has enough memory to cache the entire index. There's no I/O or memory bottleneck in this small dataset.
- 🧪 What If the Dataset Was Bigger?
```text
Imagine 100M or 500M rows instead of 10M. Index sizes would grow significantly. In that case, memory pressure kicks in ➡️ larger indexes won’t fit into RAM. That’s when partitioning shines: each partition's index is smaller, faster to read from disk, and more cache-friendly.
```

## 📊 Checking Index Sizes with pg_relation_size
- Use this SQL to find the size of all tables/indexes:

```sql
SELECT relname,
pg_relation_size(oid) AS size_bytes
FROM pg_class
ORDER BY pg_relation_size(oid) DESC;
```
##🧾 Example Output (approximate):

| Table / Index          | Size   |
| ---------------------- | ------ |
| `grades_original`      | 362 MB |
| `grades_parts_index`   | 69 MB  |
| `g_0_35` (table)       | 126 MB |
| `g_0_35_g_idx` (index) | 24 MB  |
| `g_35_60_g_idx`        | 18 MB  |
| `g_60_80_g_idx`        | 15 MB  |
| `g_80_100_g_idx`       | 15 MB  |


## 🎯 Observation:
> Partitioned indexes are much smaller (e.g., 24 MB vs. 69 MB)
> ✅ Smaller indexes = faster queries under load.

## 🧠 Feature Highlight: Partition Pruning
- What is it?
- Partition pruning tells PostgreSQL to skip irrelevant partitions based on the WHERE clause.

## 🔍 Check if it's enabled:
```sql
SHOW enable_partition_pruning;
-- Output: on
```
- 🔥 Turn it OFF and see the problem:
```sql
SET enable_partition_pruning TO off;

EXPLAIN
SELECT COUNT(*) FROM grades_parts WHERE g = 30;
```
## 😱 Output:
- PostgreSQL scans all partitions:
- g_0_35
- g_35_60
- g_60_80
- g_80_100
> ⚠️ Even though the value 30 is only in g_0_35, PostgreSQL doesn’t prune unused partitions.

## ✅ Now turn it back ON:
```sql
SET enable_partition_pruning TO on;

EXPLAIN
SELECT COUNT(*) FROM grades_parts WHERE g = 30;
```
> 🧼 Now PostgreSQL only scans g_0_35, as expected.

## ✅ Final Tips
- 🔧 Always verify that enable_partition_pruning is ON (it's ON by default).
- 📏 Use partitioning when working with large datasets (hundreds of millions of rows).
- 🔍 Query plans and index sizes become critical for scaling performance.
- 🚫 If pruning is OFF, PostgreSQL defeats the purpose of partitioning by hitting all partitions.
