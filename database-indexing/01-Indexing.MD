# Creating a Large Test Table in PostgreSQL

```bash
# Start PostgreSQL in Docker
docker run --name pg1 -e POSTGRES_PASSWORD=postgres -d postgres:13

# Enter container
docker exec -it pg1 psql -U postgres

-- Create table & insert 1M random values (0–100)
CREATE TABLE temp (t INTEGER);
INSERT INTO temp (t)
SELECT (random() * 100)::int
FROM generate_series(1, 1000000);

-- Verify
SELECT COUNT(*) FROM temp;
SELECT * FROM temp LIMIT 10;
```

# Indexing
- An index is a data structure (e.g., B-Tree, LSM Tree) built on top of a table to speed up data retrieval. Think of it like a phone book with alphabet tabs—it lets you jump to relevant rows without scanning the whole table.

# Note before proceeding forward
```sql
– 
--employees table for the indexing lecture
--paste these commands into the postgres 
--start the docker instance
docker run --name pg -e POSTGRES_PASSWORD=postgres -d postgres

docker start pg
--run postgres command shell
docker exec -it pg psql -U postgres
--the command should switch to 
--postgres=#
-- paste these sql
create table employees( id serial primary key, name text);  ( index applied on id)

create or replace function random_string(length integer) returns text as 
$$
declare
  chars text[] := '{0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W,X,Y,Z,a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z}';
  result text := '';
  i integer := 0;
  length2 integer := (select trunc(random() * length + 1));
begin
  if length2 < 0 then
    raise exception 'Given length cannot be less than 0';
  end if;
  for i in 1..length2 loop
    result := result || chars[1+random()*(array_length(chars, 1)-1)];
  end loop;
  return result;
end;
$$ language plpgsql;


insert into employees(name)(select random_string(10) from generate_series(0, 1000000));

```

## Key Takeaways
- Primary keys automatically have an index (usually B-Tree).
- Indexed lookups are much faster than full table scans.

Example: 
```sql
SELECT id FROM employees WHERE id = 2000;
```
Uses index → ~0.1 ms.

- If the column is not indexed, the DB must scan every row (sequential scan) → slow.
- Example: Searching name without an index → ~3 seconds for 11M rows.
- Index-only scans are fastest (data is fully in the index → no table lookup needed).
- Expression searches (e.g., LIKE '%abc%') often bypass indexes → still slow.
---

## Adding an index can greatly improve performance:

```sql
CREATE INDEX employees_name_idx ON employees(name);
```
> It would take sometime to create this as B+ needs to constructed for the records we have inplace
>  Earlier same operation took 36ms now it takes 0.054ms - as we’re doing a bitmap heap scan on employees name
- After indexing name, exact matches are much faster.
- Even with an index, queries like LIKE '%term%' or other expressions can’t use it effectively unless you create a specialized index (e.g., GIN, trigram).

### Best Practices
- Only SELECT the columns you need.
- Avoid SELECT *—it forces extra table lookups.
- Use indexes for frequently searched columns.
- Check execution plans with:

```sql
EXPLAIN ANALYZE your_query;
```
Consider multi-column or covering indexes for common queries.

# 📊 Understanding `EXPLAIN` in PostgreSQL

`EXPLAIN` shows PostgreSQL’s estimated query execution plan — a preview of **how** the query will run, without actually executing it.

> It's like asking Postgres: *“What’s your plan, buddy?”*

---

## 🧪 Example Table: `grades`

- **Columns**: `id`, `grade`, `name`
- **Indexes**: on `id` and `grade`
- **No index** on `name`
- **Rows**: ~200 million

---

## 📌 Key Concepts

### 1. 🔍 Sequential Scan
```sql
EXPLAIN SELECT * FROM grades;
```
- Performs a full table scan (reads every row).
- Triggered when there's no helpful index or filter.
- Cost format: startup_cost..total_cost
> **startup_cost**: Time to return the first row ;
> **total_cost**: Time to return all rows ;
> **rows**: Estimated number of rows returned.
> **width**: Estimated average row size (in bytes).

### 2. 🧮 Impact of ORDER BY
- Indexed column (grade): Sorting is fast — the index is already ordered.
- Non-indexed column (name): Sorting is expensive — requires scanning all rows and sorting in memory or disk.

### 3. 📦 Column Selection & Width
- Selecting only required columns reduces row width, improving performance and reducing data transfer.

  | Column  | Data Type | Avg Size   |
  | ------- | --------- | ---------- |
  | `id`    | integer   | \~4 bytes  |
  | `name`  | text      | \~19 bytes |
  | `grade` | double    | \~8 bytes  |

> 🎯 Avoid SELECT * to minimize unnecessary data transfer.

### 4. ⚙️ Index Scans
#### Index Scan
```sql
 EXPLAIN SELECT * FROM grades WHERE id = 10;
```
> Uses index to locate rows, then fetches them from the heap.

### Index-Only Scan
```sql
EXPLAIN SELECT id FROM grades WHERE id = 10;
```
> Uses index directly without accessing the heap — faster when the index contains all required data.

### 🧭 Reading Execution Plans
- Always read from bottom to top.
- The bottom of the plan shows the most granular (inner) operations.

## ✅ TL;DR
- **EXPLAIN** shows the estimated plan.
- **EXPLAIN** **ANALYZE** shows actual execution time.
- Avoid SELECT * — fetch only what you need.
- Index columns that are frequently filtered or sorted.

Be cautious with large TEXT/BLOB fields — they’re expensive to transfer.

