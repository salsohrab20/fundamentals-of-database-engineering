## PostgreSQL Replication Demo with Docker

This demo walks through setting up **synchronous streaming replication** between two PostgreSQL 13 instances using Docker — one **master** and one **standby**.

> ⚠️ This is a **simplified setup for learning purposes only**, not production best practices.

---

### 🛠️ Prerequisites

- Docker installed
- Basic knowledge of Postgres and Linux file systems

---

## 🔧 Step-by-Step Setup

### 1. Create Folders for Data

```bash
mkdir -p ~/Postgres/replication/master_data
mkdir -p ~/Postgres/replication/standby_data
cd ~/Postgres/replication
````

---

### 2. Run Master PostgreSQL Instance

```bash
docker run -d \
  --name p_master \
  -p 5432:5432 \
  -v "$PWD/pmaster_data:/var/lib/postgresql/data" \
  -e POSTGRES_PASSWORD=postgres \
  postgres:13
```
---

### 3. Run Standby PostgreSQL Instance

```bash
docker run -d \
  --name p_standby \
  -p 5433:5432 \
  -v "$PWD/pstandby_data:/var/lib/postgresql/data" \
  -e POSTGRES_PASSWORD=postgres \
  postgres:13
```

---

### 4. Prepare Standby by Copying Master Data

```bash
docker stop p_master p_standby

mv pstandby_data pstandby_data_backup
cp -R pmaster_data pstandby_data
```

---

### 5. Configure Master for Replication

Edit `pg_hba.conf` in `master_data/` and append below lines to the end of file:

```text
host replication postgres all md5
```

Edit `postgresql.conf` and append below lines to the end of file:

```text
wal_level = replica
max_wal_senders = 5
wal_keep_segments = 32
synchronous_standby_names = 'FIRST 1 (standby1)'
```

---

### 6. Configure Standby

Edit `postgresql.conf` in `standby_data/`:

```text
primary_conninfo = 'application_name=standby1 host=host.docker.internal port=5432 user=postgres password=postgres'
```

Create the `standby.signal` file to signal standby mode:

```bash
touch standby_data/standby.signal
```

---

### 7. Start Both Instances

```bash
docker start p_master
docker start p_standby
```

Check logs to confirm streaming:

```bash
docker logs p_master
docker logs p_standby
```

You should see logs like:

* `standby1 now a synchronous standby with priority 1`
* `standby is streaming WAL from primary`

---

### 8. Test Replication

#### Connect to master:

```bash
docker exec -it p_master psql -U postgres
```

```sql
CREATE TABLE test (id INT, t TEXT);
INSERT INTO test VALUES (1, 'Hello from master');
```

#### Connect to standby:

```bash
docker exec -it p_standby psql -U postgres
```

```sql
SELECT * FROM test; -- Should return the inserted row
```

#### Try writing to standby:

```sql
INSERT INTO test VALUES (2, 'This will fail'); -- Error: read-only instance
```

---

### ✅ Summary

| Role    | Instance    | Port | Data Dir        |
| ------- | ----------- | ---- | --------------- |
| Master  | `p_master`  | 5432 | `master_data/`  |
| Standby | `p_standby` | 5433 | `standby_data/` |

* Write operations go to master only.
* Standby reflects real-time changes via **streaming replication**.
* Setup uses **synchronous mode** for stronger consistency.

---

### ⚠️ Notes

* Using `postgres` user and `postgres` as password is **not recommended** in production.
* Use `pg_basebackup` or `pgBackRest` for production-grade base replication.
* The `standby.signal` file is key in PostgreSQL 12+ for enabling standby mode.

---

### 📚 Additional References

* [PostgreSQL Docs – Streaming Replication](https://www.postgresql.org/docs/current/warm-standby.html)
* [Docker Hub – postgres](https://hub.docker.com/_/postgres)
