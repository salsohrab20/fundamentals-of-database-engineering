# Database Sharding Overview
[Sharding.pdf](Sharding.pdf)
**Sharding** is the process of dividing a large dataset into smaller, more manageable **partitions**, distributed across multiple database instances. This improves **performance**, **scalability**, and **query efficiency**.

## Topics Covered

1. **What is Sharding?**
    - Understanding the core concept and why it's used.

2. **Consistent Hashing (Brief Overview)**
    - A key technique often used in sharding to ensure even data distribution.

3. **Sharding vs. Horizontal Partitioning**
    - These concepts are closely related, but not identical. We'll clarify the subtle differences.

4. **Hands-on Example**
    - We'll build a simple **URL shortener** using **Node.js** and multiple **PostgreSQL shards**.
    - Demonstrate writing to and reading from these shards.

5. **Pros and Cons of Sharding**
    - No system is perfect. We'll discuss the trade-offs involved.

Here's a **refined and formatted version** of your content in Markdown that captures the ideas clearly and concisely, while maintaining a natural, conversational tone. You can copy this into a `.md` file:

# Introduction to Sharding

Before we dive into **sharding**, let’s first understand the traditional approach to databases.

## Traditional Approach

Typically, applications rely on **one large centralized database** with huge tables. You query this single database, often supported by indexes, to retrieve data.

### The Problem

As your data grows:

- Tables become massive (e.g., millions of rows).
- Indexes grow in size, increasing lookup times.
- Queries get slower.
- You require more memory, CPU, and disk I/O.
- Eventually, your **single server becomes a bottleneck**.

For example, in a URL shortener service, you might have a table like this:

| Code | URL             |
|------|-----------------|
| ab12 | https://...     |
| cd34 | https://...     |
| ...  | ...             |

A query like:

```sql
SELECT url FROM urls WHERE code = 'ab12';
````

…will slow down as the number of entries grows—even with indexing—because large indexes are still costly to maintain.

## Enter Sharding

To handle this scalability issue, we introduce **sharding**.

### What Is Sharding?

Sharding is the practice of **splitting one large table** into smaller **partitioned tables**, and distributing them across **multiple databases or servers**.

Each partition (or *shard*) holds a subset of the data. For instance:

* Shard 1: Rows 1–200,000
* Shard 2: Rows 200,001–400,000
* Shard 3: Rows 400,001–600,000

This reduces the load on each server and makes queries faster by narrowing the search space.

### The Key Challenge: How Do You Shard?

Sharding relies on a **partition key** (also called *shard key*), which determines how rows are distributed.

#### Examples:

* **User Data:** Shard based on user ID ranges (e.g., users 1–500K go to shard A, 500K–1M to shard B).
* **Geographic Data:** Shard based on region or ZIP code (e.g., East vs. West Coast).
* **URLs (String Keys):** This is trickier—how do you know which shard contains a code like `5fJxe`?

This is where **consistent hashing** comes in.

## Why Consistent Hashing?

When your shard key is non-numeric or unpredictable (e.g., strings), consistent hashing offers a solution.

* It maps keys (like `"5fJxe"`) to a specific shard using a hash function.
* The hash function ensures that the same input always maps to the same shard.
* This allows you to query **only the right shard**, avoiding broadcasting queries to all shards.

For example:

```js
const shard = getShard("5fJxe"); // e.g., returns shard #3
connectTo(shard).query("SELECT url FROM urls WHERE code = '5fJxe'");
```

## Summary

* Traditional databases can’t scale indefinitely on one server.
* Sharding splits large tables across multiple databases to improve performance.
* Choosing a shard key is critical.
* **Consistent hashing** helps route requests to the correct shard, especially for string-based keys.

Here's a **refined and well-structured Markdown version** of your explanation on **consistent hashing**, tailored for technical clarity and easier reading. This fits well as a follow-up section in your `.md` presentation or notes.

# Consistent Hashing

Let’s assume we have **three shards**—three separate databases running on different ports:

- Shard 1 → `localhost:5432`
- Shard 2 → `localhost:5433`
- Shard 3 → `localhost:5434`

This is the setup we’ll use in our hands-on example later.

## The Problem

Given an **input string** (e.g., `input1`, `input2`, etc.), how do we decide **which shard** to send the query to?

We want to ensure that **every time the same input is used**, it consistently maps to the **same database shard**. This way:

- `input1` always maps to shard 1
- `input2` always maps to shard 2
- `input3` always maps to shard 3
- and so on...

That’s where **consistent hashing** comes in.

## What Is Consistent Hashing?

Consistent hashing is a technique that maps inputs (like strings) to a **fixed set of nodes** (or shards) in a way that:

- Is deterministic (same input → same shard)
- Minimizes rebalancing if a node is added/removed
- Supports even data distribution

## How It Works (Simplified)

One way to implement consistent hashing is:

1. **Hash the input string**  
   Convert the string (e.g., `"input1"`) into a number using a hash function.

2. **Modulo Operation**  
   Take the **modulus** of that number with the total number of shards.
 ```js
   const num = hash("input1"); // e.g., result = 10
   const shardIndex = num % 3; // e.g., 10 % 3 = 1
```

3. **Map to a Shard**
   Based on the index, connect to a specific database:

    * Index 0 → Port `5432`
    * Index 1 → Port `5433`
    * Index 2 → Port `5434`

4. **Always Consistent**
   The same input (`"input1"`) will **always** hash to the same shard (`5433` in this case).

## Example Mapping

| Input    | Hash Result | Modulo 3 | Port |               |
| -------- | ----------- | -------- | ---- | ------------- |
| `input1` | 10          | 1        | 5433 |               |
| `input2` | 11          | 2        | 5434 |               |
| `input3` | 12          | 0        | 5432 |               |
| `input2` | 11          | 2        | 5434 | ✅ Consistent! |

## Why a Ring?

This concept is often visualized as a **ring** (called a *hash ring*), where:

* Nodes (shards) are placed on the ring.
* Inputs are hashed to a position on the ring.
* You move clockwise on the ring to find the first node responsible for the input.

This model allows for **better balancing** and **minimal reshuffling** when nodes are added or removed.

> Technologies like **Cassandra** and **DynamoDB** use consistent hashing to distribute data across nodes efficiently.

---

# Horizontal Partitioning vs Sharding

While **horizontal partitioning** and **sharding** seem similar—they both involve splitting data—there are key differences in how and where the data is stored.

---

## 🔹 Horizontal Partitioning

- **What is it?**  
  Splits a large table into smaller **tables** (or partitions) **within the same database instance**.

- **How it's done:**  
  Data is divided based on a key (e.g., region, user ID range), and stored in different tables or schemas.  
  Example:
    - `customers_west`
    - `customers_east`

- **Where is the data?**  
  All partitions reside on the **same server/database instance**.

- **Client responsibility:**  
  The client (or app logic) must be aware of which partition (table/schema) to query.

---

## 🔹 Sharding

- **What is it?**  
  Splits a large table into parts that are stored across **multiple database instances or servers**.

- **How it's done:**  
  Each shard contains a subset of the original table’s rows—same structure, **same table name**, but stored in different databases.  
  Example:
    - All databases have a table called `customers`, but each stores different rows.

- **Where is the data?**  
  Spread across **multiple servers** (e.g., `db1`, `db2`, `db3`), each with its own independent database instance.

- **Client responsibility:**  
  The application must determine **which database server** to connect to (usually via a sharding strategy like consistent hashing).

---

## 🔁 Summary Table

| Feature                 | Horizontal Partitioning            | Sharding                           |
|-------------------------|------------------------------------|------------------------------------|
| Partition location      | Same database instance             | Different database instances       |
| Structure               | Multiple tables/schemas            | Same table name on each shard      |
| Client knowledge        | Must know partition names          | Must know which server to query    |
| Example                 | `customers_east`, `customers_west` | Same `customers` table on each DB  |

---

In short:
- **Horizontal Partitioning:** Same DB, multiple tables/schemas.
- **Sharding:** Same table name, multiple DB servers.

This distinction is subtle but crucial when designing scalable database systems.

Got it! Here’s a refined and concise summary of your sharding pros discussion:

---

### Pros of Sharding

1. **Scalability:**
   Sharding lets you horizontally scale your system by distributing data, CPU, memory, and other resources across multiple machines. Instead of one server handling everything, the load is spread out, improving overall performance and capacity.

2. **Improved Security and Access Control:**
   By partitioning data by shard, you can enforce access controls at the shard level. For example, you could isolate VIP customer data in a specific shard, restricting who can access it. This adds a layer of security by limiting access to sensitive data subsets.

3. **Smaller Index Sizes:**
   Since each shard holds only a portion of the total data, indexes are smaller and more efficient. This leads to faster query execution and better overall database performance.

---

In short: sharding offers scalability, security benefits, and query optimization by splitting data smartly across multiple shards.

---
Here’s a refined and summarized version of the cons you shared about sharding:

---

### Cons of Sharding

1. **Increased Client Complexity:**
   Clients need to be shard-aware, knowing which shard holds which data. This adds complexity to the client logic, making the application harder to maintain. Although consistent hashing can abstract some of this, overall complexity still grows as the system scales.

2. **Difficult Transactions Across Shards:**
   Performing atomic transactions involving multiple shards is nearly impossible. For example, inserting related data into two different shards with transactional guarantees and rollbacks is very challenging, often requiring complex workarounds.

3. **Challenging Rollbacks:**
   Rolling back multi-shard operations is complicated because each shard is a separate database. Coordinating consistent rollbacks across shards for a single logical transaction is tough.

4. **Complex Schema Changes:**
   Any schema change, such as adding a new column, must be applied consistently to all shards. This is error-prone and can cause issues if one shard is missed or updated incorrectly.

5. **No Cross-Shard Joins:**
   Joins across shards/databases are either impossible or extremely difficult, forcing developers to redesign queries or data models to avoid cross-shard operations.

6. **Expensive Non-Key Queries:**
   Queries on fields other than the shard key require querying all shards, which is costly and inefficient. For example, searching by a customer’s name when data is sharded by customer ID requires scanning every shard.

---

In summary: while sharding offers scalability and performance benefits, it introduces significant complexity in client logic, transactions, schema management, and querying.

---

Absolutely! Here’s a refined, punchy, emoji-packed summary of your video script that keeps your vibe intact:

---

# When to Shard Your Database? 🤔💾
**Short answer:**
**Don’t shard your database unless you really have to!** 🚫🛠️ Sharding is super complex and should be your **last resort** after trying simpler fixes.

---

### Why? What’s the deal with sharding? 🤷‍♂️

* Sharding sounds sexy and smart, but it’s a **complex beast** to implement and maintain.
* Before sharding, try optimizing what you can — **indexes, partitioning, replication, caching**.
* Most people jump to sharding way too early and end up making their life harder.

---

### Here’s what to try first:

1. **Partitioning** 🗂️
   Split your big table into smaller chunks (horizontally).
   Smaller tables = smaller indexes = faster queries.
   And this usually happens **in the same database instance** — no extra servers needed!

2. **Replication** 🔄
   Set up one master for writes and multiple replicas for reads.
   Load balance reads across replicas to reduce strain on your master.
   Writes still go to one place — easier to manage consistency.

3. **Multi-region replication** 🌍
   If you write a lot, try geographically splitting masters (East Coast writes here, West Coast writes there).
   This limits conflicts and spreads load.

4. **Caching** 🧠
   Use Redis or similar to cache frequent queries.
   Beware of cache invalidation issues though!

---

### Sharding — The Big Guns 💥

Only **after exhausting all the above**, consider sharding.

* Sharding means splitting your data **across multiple database servers**.
* Your app needs to know which shard to talk to — this adds **complexity and coupling**.
* You **lose ACID transactions** across shards, making rollbacks and consistency a nightmare.
* Schema changes become harder — you must update all shards consistently.
* Joins across shards? Forget it. Queries on non-shard keys? Expensive!

---

### Real-World Example: YouTube 📺

YouTube managed for years on a single MySQL server until writes overwhelmed it.
Then they sharded at the application level — but that got messy and hard to maintain.
So they built **Vitesse** — a middleware that handles sharding transparently on top of MySQL.
You send a normal SQL query, and it routes it to the right shard behind the scenes! Magic ✨

---

### TL;DR 🎯

* Sharding = **complex, costly, last resort**
* Try indexing, partitioning, replication, caching **first**
* Only shard when your single DB or simple solutions just can’t keep up
* Know the trade-offs: No easy transactions, complex app logic, difficult schema changes

---

Stay smart, avoid premature optimization, and keep your software as simple and isolated as possible! 🧩✌️

---




