# Database Partitioning – Summary Notes

## Definition
**Database partitioning** is a technique used to split a large table into smaller, more manageable pieces (partitions). The database automatically determines which partition to query based on the `WHERE` clause.

---

## Key Topics Covered

### 1. What is Partitioning?
- A method to improve performance and manageability of large datasets.
- Database internally routes queries to the correct partition.

### 2. Why Partitioning Exists
- To enhance query performance, scalability, and maintainability of large tables.

### 3. Types of Partitioning

#### Horizontal Partitioning (main focus)
- Splits data rows across partitions based on values (e.g., by range, list, hash).

#### Vertical Partitioning
- Splits columns across tables.
- Rarely used today; similar in concept to normalization.

### 4. Partitioning Strategies
- **Range-based**
- **List-based**
- **Hash-based**
- **Composite** (combination of the above)

### 5. Partitioning vs. Sharding
- **Sharding**: Distributes data across multiple machines.
- **Partitioning**: Splits data within a single database instance.
- Sharding is generally more complex than partitioning.

### 6. Demo Overview (Using PostgreSQL)
- Demonstrates partitioning a table with 110 million rows.
- Shows practical implementation and performance benefits.

### 7. Pros and Cons

#### Pros
- Improved query performance
- Easier maintenance (e.g., archiving old partitions)
- Better data management

#### Cons
- Added complexity in design
- Overhead in certain query patterns
- Not always beneficial for small or simple datasets

---

## Final Notes
From this point forward, “partitioning” refers to **horizontal partitioning** unless specified otherwise.  
Vertical partitioning is largely outdated in modern systems.

# What is Partitioning? – Summary Notes
[Partitioning.pdf](Partitioning.pdf)

## Problem: Querying Large Tables
- Suppose we have a `customers` table with **1 million rows**.
- Query example:
```sql
  SELECT name FROM customers WHERE id = 700001;
```
- If there's an index, the database uses it to directly locate the row.
- Without an index, a sequential scan is performed—scanning all rows.
- Even with indexes, querying massive tables (millions to billions of rows) can be slow.

### Solution: Partitioning
- Partitioning splits a large table into smaller, manageable pieces called partitions.
- This reduces the amount of data the database needs to scan for a given query.
> Example: Range-Based Partitioning
- Original customers table (1M rows) is divided into 5 partitions:
```text
customers_200k: IDs 1–200,000
customers_400k: IDs 200,001–400,000
customers_600k: IDs 400,001–600,000
customers_800k: IDs 600,001–800,000
customers_1m: IDs 800,001–1,000,000
```
- The main customers table becomes a parent table, containing no data.
- Actual data resides in the partitioned child tables.

```How It Works
When a query is run:
SELECT name FROM customers WHERE id = 700001;
The database checks partition metadata to identify the correct child table.

Example: ID 700001 falls in customers_800k (600001–800000).

Query is routed directly to the correct partition.

This reduces scanned data from 1M rows to only 200K rows.
```

> Bonus: Indexing Partitions
- Indexes can still be applied on individual partitions.
- With an index on customers_800k, the query becomes even faster.

# Horizontal vs. Vertical Partitioning 

## Horizontal Partitioning

### What is it?
- **Splits a table by rows** based on a defined condition (e.g., ID ranges, zip codes).
- Each partition contains **different rows** but **all columns**.
- The parent table is logically unified, but data is physically divided.

### Example:
- `customers` table with 1M rows is split by ID range:
    - `customers_200k`: IDs 1–200,000
    - `customers_400k`: IDs 200,001–400,000
    - ...and so on.

### Partitioning Strategies:
- **Range**: e.g., IDs 1–200K in one partition, 200K–400K in another.
- **List**: e.g., specific values like zip codes or regions.

### Visual Analogy:
> Like slicing a loaf of bread **horizontally**—you divide it into thinner horizontal pieces (partitions) by row.

### Use Case:
- Improves performance by limiting data scans to relevant partitions based on row values (like `WHERE id = 700001`).

---

## Vertical Partitioning

### What is it?
- **Splits a table by columns**, moving certain columns into separate tables.
- Each vertical partition contains **all rows**, but **only a subset of columns**.
- Typically used to isolate infrequently accessed or large columns.

### Example:
- Move a large `document_blob` column into a separate table:
    - `customers_main`: `id`, `name`, `email`
    - `customers_documents`: `id`, `document_blob`

### Benefits:
- Keeps frequently queried data (small columns) on **fast storage** (e.g., SSD).
- Large columns (e.g., blobs, JSON, text) are moved to **slower or cheaper storage**.
- **Improves I/O efficiency**, reduces memory usage, and speeds up common queries.

### Visual Analogy:
> Like slicing a cake **vertically**—you divide the table by cutting out columns, not rows.

### Use Case:
- Ideal when large columns are rarely accessed but consume significant space or slow down queries.

---

## Summary

| Type                  | Splits By      | Contains       | Use Case                                      |
|-----------------------|----------------|----------------|-----------------------------------------------|
| **Horizontal**        | Rows           | All columns    | Improve query performance on large datasets   |
| **Vertical**          | Columns        | All rows       | Optimize storage and access for large columns |


# Partitioning Strategies

Partitioning is not one-size-fits-all. Different strategies exist to split large datasets depending on the use case. Here are the most common partitioning methods:

---

## 1. Range Partitioning

### Description:
- Splits data based on **ranges of values** (e.g., date or ID ranges).
- Commonly used for **time-series data** or **sequential IDs**.

### Examples:
- Partition by `id`:
    - `1–200K`, `200K–400K`, etc.
- Partition by `log_date`:
    - Each **year**, **quarter**, or **month** gets its own partition:
        - `2012_data`, `2013_data`, `2020_Q1`, etc.

### Benefits:
- Great for aging data.  
  Example: Move old partitions (e.g., 1995 data) to slower or cheaper storage like tape or archive drives.
- Only relevant time ranges are queried—**increases performance**.

---

## 2. List Partitioning

### Description:
- Splits data based on **discrete values** rather than ranges.

### Examples:
- Partition by **state**, **country**, or **zip code**:
    - California data → `customers_CA`
    - Alabama data → `customers_AL`
    - Zip 90205 → `zip_90205`

### Benefits:
- Ideal for **categorical data** (e.g., regions, departments, product types).
- Allows for **targeted partitioning** based on known values.

---

## 3. Hash Partitioning

### Description:
- Uses a **hash function** to evenly distribute rows across partitions.
- Hash is calculated based on one or more column values.

### Examples:
- Hash customer ID, order ID, etc.:
    - `hash(id) % 4 → partition_0 to partition_3`

### Common Usage:
- Used heavily in distributed systems like **Cassandra**.
- Similar to **IP hashing** in load balancers: distributes load evenly.

### Benefits:
- Avoids uneven data distribution (aka *data skew*).
- Good for **load balancing** and parallel querying.

---

## Summary Table

| Type               | Based On          | Best For                            | Example Use Cases                        |
|--------------------|-------------------|--------------------------------------|------------------------------------------|
| **Range**          | Ranges of values  | Time-series or ordered data          | IoT logs, IDs, timestamps                |
| **List**           | Discrete values   | Categorical grouping                 | States, countries, zip codes             |
| **Hash**           | Hash of column(s) | Even data distribution               | High-scale systems, Cassandra, proxies   |

# 🧩 Partitioning vs. Sharding – Summary Notes

Understanding the distinction between **partitioning** and **sharding** is critical in designing scalable databases. Though both involve splitting large datasets into smaller pieces, they differ in **where** and **how** the data is stored and accessed.

---

## 1. 🧱 Partitioning (Horizontal Partitioning)

### 🔍 What is it?
- Divides a large table into **multiple partitions** (smaller tables).
- All partitions reside on the **same database server**.
- Managed **transparently by the database engine**.

### ✅ Characteristics:
- 👤 Client is **agnostic** of the partitions.
- 🧾 Query syntax does **not change**.
- 📊 The database decides which partition to query based on the `WHERE` clause and internal metadata.
- 🛠️ Examples: PostgreSQL, MySQL, SQL Server native partitioning.

### 🚀 Benefits:
- ✅ Simplifies client logic.
- ⚡ Efficient query routing handled internally.
- 🧹 Easier to maintain than sharding.

---

## 2. 🌍 Sharding

### 🔍 What is it?
- Splits data across **multiple database servers** (shards).
- Each shard holds a **subset** of the overall data.
- Typically done to **scale horizontally** across infrastructure.

### ✅ Characteristics:
- 📍 The same table logically exists on multiple servers.
- 🌐 Shards often located in **different geographical regions** or datacenters.
- 🔀 Query routing can be done via:
    - 🧭 Proxy (e.g., ProxySQL, Citus, Vitess)
    - 🧠 Application logic
- 👁️ Clients are often **aware of the shard** they're communicating with.

### 🌟 Example Use Case:
- 🏙️ California customers' data stored on a **California-based DB server**.
- 🗾 Asian customers' data stored on an **Asia-based DB server**.
- 🌐 Request routing depends on user location/IP.

### ⚠️ Downsides:
- 🧩 Increased complexity in query routing, failover, and consistency.
- 🛠️ Harder to implement and maintain.
- 💰 Requires significant investment and architectural decisions.

---

## ⚖️ Key Differences

| Feature                    | 🧱 Partitioning                             | 🌍 Sharding                                 |
|----------------------------|---------------------------------------------|----------------------------------------------|
| 📍 Data Location            | Same database server                        | Different database servers                   |
| 👁️ Client Awareness         | **Client is unaware** of partitions         | **Client or proxy is aware** of shards       |
| ⚙️ Management               | Handled by the **database engine**          | Handled by **application or proxy layer**    |
| 🎯 Use Case                 | Performance optimization                    | Horizontal scaling, global distribution      |
| 🧾 Query Syntax             | Stays the same                              | Stays the same, but target server differs    |
| 🏷️ Table Names              | Partition tables may differ                 | Table name typically **stays the same**      |
| 🧠 Complexity               | Relatively low                              | High                                         |

---

## 📝 Final Thoughts

- Use **🧱 partitioning** when scaling within a single database instance for performance and manageability.
- Use **🌍 sharding** when scaling **beyond** a single server, especially in global or high-traffic applications.
- Tools like **Vitess** and **Citus** aim to abstract away sharding complexity—but it's still a **major architectural decision**.

# 📦 Pros and Cons of Table Partitioning in Databases

> ⚖️ Partitioning is powerful — but not perfect. As backend engineers, it's critical to weigh the tradeoffs.

---

## ✅ Pros of Partitioning

### 🔥 1. **Improved Query Performance**
- Queries targeting a **specific partition** are significantly faster than scanning a large monolithic table.
- Reduces the **amount of data read**, **I/O overhead**, and **memory usage**.
- Especially useful when:
    - Tables have **billions of rows**.
    - You're **memory bound** or **I/O bound**.

> 🧠 Example:  
> If your machine has limited RAM and the index is too big to fit in memory, scanning a smaller partitioned index is much faster than scanning one massive index.

---

### 🧠 2. **Smarter Query Planning**
- The **PostgreSQL planner** can:
    - Avoid scanning irrelevant partitions (via **partition pruning**).
    - Choose between **sequential scan** or **index scan** based on data size and filter selectivity.
- For smaller partitions, even **sequential scans can outperform** index scans (because no index jumping overhead).

---

### 🚀 3. **Faster & Easier Bulk Loading**
- You can:
    - Load data into a staging table.
    - Then **attach it as a partition** to a partitioned table.
- ✅ No need to reindex or copy data if the constraints are already valid.

---

### 💾 4. **Storage Optimization & Archival**
- Archive old partitions (e.g., year 2001 data) to **cheap, slow storage** (e.g., HDD).
- Keep only recent, frequently accessed data on **fast SSDs**.
- Done via **tablespaces** or moving partitions manually.

---

### 🧰 5. **Maintainability**
- Smaller partitions are easier to:
    - Vacuum
    - Analyze
    - Backup or drop independently
- Can **truncate or delete** old partitions without affecting current data.

---

## ⚠️ Cons of Partitioning

### 🤯 1. **Complexity**
- Adds significant overhead to your schema design and query planning.
- You need to:
    - Manually create and manage partitions.
    - Be aware of how queries interact with partition pruning.

---

### 📉 2. **Performance Not Always Better**
- If your data is small or fits in memory:
    - Partitioning may **not improve performance**.
    - Might even add slight overhead due to partition lookup logic.

---

### 🧪 3. **Query Planner May Still Scan All Partitions**
- If **partition pruning is off** (e.g., `enable_partition_pruning = off`):
    - PostgreSQL may **scan every partition**, even if only one is needed.
- 💡 Always verify pruning behavior using `EXPLAIN`.

---

### 🧱 4. **Limitations in Some Features**
- Not all PostgreSQL features work seamlessly with partitioned tables:
    - Foreign keys to/from partitioned tables are **limited**.
    - Certain types of constraints, triggers, or indexes may behave differently.

---

### 🧹 5. **Maintenance Overhead**
- While partitioning simplifies data management in large tables, it also:
    - Requires **monitoring partition boundaries**.
    - Needs **scripts or logic** to add/remove partitions regularly.

---

## 🧾 Summary

| Pros 👍 | Cons 👎 |
|--------|---------|
| Better query performance (when optimized) | Added schema complexity |
| Efficient memory and I/O usage | Not always faster on small data |
| Easier data archival and deletion | Pruning must be configured correctly |
| Smarter planning, smaller index scans | Feature limitations in PostgreSQL |
| Scalable schema design | Needs ongoing maintenance |

---

## 🧠 Final Thoughts

Partitioning is a **powerful tool**, especially when scaling large systems. But it's not a silver bullet. Use it when:

- You're handling **large datasets**
- You need **optimized access patterns**
- You want **manageable archival strategies**

> 🎯 "Everything is a double-edged sword in backend engineering."  
> Understand your use case deeply before committing to partitioning.

