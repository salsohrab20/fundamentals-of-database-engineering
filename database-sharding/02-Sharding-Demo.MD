# 🛠️ Building a Sharded URL Shortener with PostgreSQL, Docker & Node.js

Alright folks—now that we've covered the theory behind **sharding**, **consistent hashing**, and **horizontal partitioning**, let’s **build something real**!

---

* Source Code : https://github.com/hnasr/javascript_playground/tree/master/sharding

* Docker commands (including pgadmin) : https://github.com/hnasr/javascript_playground/blob/master/sharding/shards/commands.txt

* Dockerfile & init.sql : https://github.com/hnasr/javascript_playground/tree/master/sharding/shards
## 🚀 What We're Going to Build

We'll build a simple **URL shortener** service, but with a twist:

- It will **shard data across 3 PostgreSQL instances**
- We'll use **consistent hashing** to route write/read requests to the correct shard
- We'll build the backend in **Node.js with Express**
- We'll expose **REST APIs** to create and fetch short URLs
- We'll use **pgAdmin** to visualize the shards

---

## 🧱 Architecture Overview

### ✅ Components

| Component   | Description                                  |
|-------------|----------------------------------------------|
| Docker      | To spin up 3 PostgreSQL instances + pgAdmin  |
| PostgreSQL  | Database engine, one per shard               |
| pgAdmin     | DB management UI to inspect shards           |
| Node.js     | Backend logic with Express + pg driver       |
| Consistent Hashing | To determine which shard to write/read |

---

## 🐳 Docker Setup

We’ll spin up **three PostgreSQL containers**, each running on:

- `5432` → Shard 1
- `5433` → Shard 2
- `5434` → Shard 3

Each DB will be initialized with a `urls` table for storing:

```sql
CREATE TABLE urls (
    id SERIAL PRIMARY KEY,
    code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
````

We'll also spin up **pgAdmin** to inspect these shards.

---

## 📦 Backend: Node.js + Express

We’ll build a small REST API with two endpoints:

* `POST /shorten` → Shortens a URL

    * Accepts `long_url` in the body
    * Uses consistent hashing to determine the shard
    * Stores the short code + URL in the correct shard
* `GET /:code` → Expands the short URL

    * Hashes the code
    * Queries the correct shard to retrieve the original URL

### Consistent Hashing

We'll implement or use a small consistent hashing utility in Node.js to route traffic to the correct shard based on the short code.

---

## 🧪 Goals

By the end, we will:

* Understand how to **shard data in PostgreSQL**
* See **consistent hashing** in action
* Be able to **read/write** to specific shards from a single app
* Visualize the data distribution using **pgAdmin**

---

Awesome! You're setting the foundation really well. Here's a cleaned-up, structured Markdown version of everything you've just walked through — perfect for project documentation or tutorial steps:

---

````markdown
# ⚙️ Step-by-Step: Setting Up PostgreSQL Shards with Docker

Now that we're ready to build our sharded URL shortener, let's start by spinning up **three PostgreSQL containers**, each representing a shard.

---

## 📁 Project Structure

Start by creating a project folder:

```bash
mkdir sharding-video
cd sharding-video
````

Inside this folder, we'll create a few key files:

* `init.sql` – SQL script to initialize the database
* `Dockerfile` – Dockerfile to create a custom PostgreSQL image
* (Later) `docker-compose.yml` – Optional for managing multiple containers easily
* (Later) Node.js code to interface with the shards

---

## 🗃️ Step 1: SQL Initialization Script

Create a file named `init.sql`:

```sql
-- init.sql

CREATE TABLE url (
    id SERIAL PRIMARY KEY NOT NULL,
    long_url TEXT NOT NULL,
    url_id CHAR(5) NOT NULL UNIQUE
);
```

This script defines the schema we'll use in each shard: a simple URL shortener table with a unique short code and long URL.

---

## 🐳 Step 2: Dockerfile for PostgreSQL Shard

Create a file named `Dockerfile`:

```Dockerfile
# Dockerfile

FROM postgres
COPY init.sql /docker-entrypoint-initdb.d/
```

**What this does:**

* Builds on top of the official Postgres image
* Copies `init.sql` into Postgres’s auto-run initialization folder
* When the container spins up, Postgres will automatically execute this script

---

## 🏗️ Step 3: Build the Custom Docker Image

```bash
docker build -t pg-shard .
```

This creates a custom Docker image named `pg-shard` with our schema preloaded.

---

## 🚢 Step 4: Spin Up Shard Containers

Spin up three PostgreSQL containers using the new image:

```bash
docker run -d --name=pg-shard-1 -p 5432:5432 pg-shard
docker run -d --name=pg-shard-2 -p 5433:5432 pg-shard
docker run -d --name=pg-shard-3 -p 5434:5432 pg-shard
```

Each container:

* Runs in detached mode (`-d`)
* Has a unique name
* Maps a unique host port to container port `5432`
* Uses our `pg-shard` image

---

## 🧰 Step 5: Connect with pgAdmin (Optional UI)

If you already have `pgAdmin` running (e.g., on port `5555`), you can use it to manage and inspect your shards.

### Add Servers in pgAdmin:

| Shard  | Host        | Port | Username | Password | DB Name  |
| ------ | ----------- | ---- | -------- | -------- | -------- |
| Shard1 | `localhost` | 5432 | postgres | postgres | postgres |
| Shard2 | `localhost` | 5433 | postgres | postgres | postgres |
| Shard3 | `localhost` | 5434 | postgres | postgres | postgres |

Once added, you should see the `url` table inside each shard with the correct schema (but empty).

---
Here's a **refined and summarized version** of your notes, formatted cleanly in Markdown (`.md`) for documentation or future study. The goal here is clarity, conciseness, and completeness, focusing on the **key concepts and implementation steps** from your project.

---

# 🧪 URL Shortener with Consistent Hashing (Sharding Example)

## 📦 Requirements

* **Node.js**
* **Visual Studio Code**
* **PostgreSQL** (3 separate databases with different ports)
* Libraries:

  ```bash
  npm install express pg consistent-hash crypto
  ```

---

## 🗂️ Project Setup

1. **Initialize Project**

   ```bash
   mkdir sharding && cd sharding
   npm init -y
   touch index.js
   ```

2. **Dependencies**

   ```bash
   npm install express pg consistent-hash crypto
   ```

---

## 🧱 Architecture Overview

We're building a basic **URL shortener** service with **3 PostgreSQL database shards**. Each URL is:

* **Hashed consistently** to determine the target database.
* Stored in the correct DB with a short hash ID.
* Retrieved based on the short hash ID.

---

## ⚙️ Database Clients Setup

```js
const { Client } = require("pg");

const clients = {
  5432: new Client({ host: "localhost", port: 5432, user: "postgres", password: "postgres", database: "postgres" }),
  5433: new Client({ host: "localhost", port: 5433, user: "postgres", password: "postgres", database: "postgres" }),
  5434: new Client({ host: "localhost", port: 5434, user: "postgres", password: "postgres", database: "postgres" }),
};

async function connect() {
  await Promise.all(Object.values(clients).map(c => c.connect()));
}
connect();
```

---

## 🚀 Express App

```js
const express = require("express");
const crypto = require("crypto");
const ConsistentHash = require("consistent-hash");
const app = express();
const port = 8081;
```

---

## 🔄 Consistent Hash Ring Setup

```js
const hashRing = new ConsistentHash(["5432", "5433", "5434"]);
```

---

## 📝 POST `/shorten?url=<URL>`

1. **Get URL from query param.**
2. **Generate SHA256 hash** using `crypto`.
3. **Generate short ID** from hash (first 5 chars).
4. **Get target DB shard** using `consistent-hash`.
5. **Insert into `url_table`** in the selected database.

```js
app.post("/shorten", async (req, res) => {
  const url = req.query.url;
  const hash = crypto.createHash("sha256").update(url).digest("base64");
  const url_id = hash.substring(0, 5);

  const targetPort = hashRing.getNode(url_id);
  const client = clients[targetPort];

  await client.query(
    `INSERT INTO url_table (url, url_id) VALUES ($1, $2)`,
    [url, url_id]
  );

  res.json({ url_id, url, server: targetPort });
});
```

---

## 🔍 GET `/expand/:id`

(Later implementation idea)

* Look up `url_id` in all 3 databases (or implement consistent hash on `url_id` again).
* Return full original URL.

---

## 🧪 Testing

```js
app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
```

### Sample Fetch Call

```js
fetch("http://localhost:8081/shorten?url=https://wikipedia.com/sharding", {
  method: "POST"
})
.then(res => res.json())
.then(console.log);
```

---

## 🧠 Concepts Used

* **Consistent Hashing** for sharding logic
* **Node.js & Express** to build the API
* **PostgreSQL Clients** for multiple DB connections
* **Crypto module** for URL hashing
* **Base64 hashing + Substring** for short ID generation
* **Client map** to maintain DB connections

---

## 🛢️ SQL Schema (example)

```sql
CREATE TABLE url_table (
  id SERIAL PRIMARY KEY,
  url TEXT NOT NULL,
  url_id VARCHAR(10) UNIQUE NOT NULL
);
```

---

## ✅ Notes

* **Consistent Hashing** helps to avoid rehashing everything when nodes are added/removed.
* Always use `async/await` with DB queries.
* Don't forget **CORS** if accessing from browser clients.
* Use `try/catch` for production-grade error handling.
* Inserted rows can be verified via direct DB access (e.g., `psql` or PGAdmin).

---

Sure! Here’s a refined and summarized version of your explanation about building the GET “gate” endpoint for a sharded URL shortener:

---

### Building the GET Route (The Gate) for Sharded URL Shortener

We already covered the POST (write) part. Now let’s handle the GET (read) route, which fetches a URL based on an ID passed in the URL path.

1. **Extract the ID from the URL params:**

```js
const id = req.params.urlId;
```

This `id` is the key part of the URL, e.g. `http://localhost:8081/<urlId>`, where `<urlId>` is the short URL identifier.

2. **Determine which shard (database) to query:**

We use a consistent hashing function (I switched to a better library called `hash-rank` because the previous one was buggy and inconsistent) to get the right shard for this ID.

```js
const shard = hashRank.getShard(id);
```

3. **Query the shard for the URL record:**

```js
const result = await shard.query('SELECT * FROM urls WHERE id = $1', [id]);
```

Always use parameterized queries to avoid SQL injection risks.

4. **Return the results or a 404 if not found:**

```js
if (result.rowCount > 0) {
  res.json(result.rows[0]);
} else {
  res.sendStatus(404); // Not Found
}
```

5. **Debugging tips:**

* Make sure your GET handler is async since you await DB calls.
* Verify the hash function returns consistent shards for the same key.
* Test with existing IDs in your shards to confirm correct routing.
* Handle the case where no record is found by returning 404.

---

### Notes on Sharding and This Implementation

* This is a basic but functional sharded URL shortener.
* The consistent hashing ensures the same ID always maps to the same shard.
* Adding shards dynamically requires careful resharding or migration.
* This example doesn’t handle duplicates or shard failures—those are more advanced topics.
* Always avoid raw string interpolation in SQL queries to prevent injection.

---

### Summary

* Extract URL ID from request params.
* Use consistent hashing to pick the correct database shard.
* Query the shard with a parameterized query.
* Return the found URL or 404 if missing.
* Switched to a reliable hashing library (`hash-rank`) for consistent shard selection.
* Tested and verified with sample data across shards.

---
