# 🤖 Automating Table Partitioning in PostgreSQL (with Node.js)

In response to a great question from the *Intro to Database Engineering* course:
[automate_partitions](automate_partitions)
> 🧠 **"How do I dynamically create multiple partitions for a large table like `customers`?"**

Here's how to automate horizontal partitioning using **JavaScript + PostgreSQL**, ideal for scenarios like supporting **1 billion customers**, each partitioned by ID ranges of **10 million**.

---

## 🏗️ What We're Building

- A main table: `customers`
- Partitioned **by range on the `id` field**
- Automatically create **100 partitions** (1B ÷ 10M)
- Insert 10M dummy customers across all partitions

---

## ⚙️ Tools & Environment

- PostgreSQL (via Docker)
- Node.js + `pg` npm package
- SQL scripting for partitioning logic

---

## 🚀 Steps Breakdown

### 1. Start PostgreSQL via Docker

```bash
docker run --name pgk \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -d postgres



> 🔐 Credentials:
>
> * user: `postgres`
> * password: `password`

---

### 2. JavaScript Scripts

#### 🛠️ `create_partitions.js`

* Connects to PostgreSQL
* (Optionally) drops and recreates `customers` DB
* Creates a partitioned table:

```sql
CREATE TABLE customers (
  id INT NOT NULL,
  name TEXT NOT NULL
) PARTITION BY RANGE (id);
```

* Creates 100 partitions dynamically:

    * Names like: `customers_id_from_0_to_9999999`, etc.
    * Range steps of 10,000,000 per partition
    * Attaches each partition to the main table

#### 🧪 Example Partition Attach Query:

```sql
ALTER TABLE customers
ATTACH PARTITION customers_id_from_0_to_9999999
FOR VALUES FROM (0) TO (10000000);
```

---

### 3. Populate Dummy Data

#### 📝 `populate_customers.js`

* Loops to insert 10M customer rows into the `customers` table
* Uses batch inserts for speed

> ✅ Postgres will automatically route inserts to the correct partition based on `id`.

---

### 4. Connect & Query

To connect to the database via CLI:

```bash
docker exec -it pgk psql -U postgres -d customers
```

Useful queries:

```sql
-- See partition details
\d+ customers

-- Check number of rows in a partition
SELECT COUNT(*) FROM customers_id_from_0_to_9999999;

-- Max inserted ID
SELECT MAX(id) FROM customers;
```

---

## 📁 Directory Structure

```
.
├── create_partitions.js
├── populate_customers.js
├── package.json
└── README.md
```

> Make sure to run `npm install pg` before executing scripts.

---

## 🧠 Tips & Warnings

* ⚠️ Don’t run this on production without modification — especially the DB drop logic!
* Each partition is a **standalone table** under the hood.
* You **only insert into the parent** (`customers`) — Postgres routes rows for you.
* Easily scale this to **billions of rows** with partitioning.

---

## 📦 Summary

| What You Gained     | Description                                        |
| ------------------- | -------------------------------------------------- |
| ✅ Automation        | Created 100 partitions programmatically            |
| 🔧 Reusability      | JS script works for other tables & ranges          |
| 🗃️ Scalable Design | Ready for billions of rows                         |
| ⚡ Performance       | Partitions reduce index size and boost performance |

---

> 🙌 Partitioning doesn't have to be painful — automating it saves time, avoids mistakes, and sets you up for scalable systems.

```
