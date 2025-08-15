# Database Replication Notes

[Replication.pdf](../../../../../../Downloads/Replication.pdf)

## Overview
Database replication involves sharing data between redundant database instances to ensure **consistency**, **reliability**, **fault tolerance**, and **accessibility**.

## Topics Covered

### 1. Replication Configurations
- **Master-Standby (Master-Follower/Backup)**: One primary node handles writes, and secondary nodes replicate data for redundancy.
- **Multi-Master Replication**: Multiple nodes can accept writes and replicate changes across each other.
- Differences between the two configurations are subtle but important.

### 2. Replication Types
- **Synchronous Replication**: Transactions are committed on both primary and replica before completing. Ensures consistency but may impact performance.
- **Asynchronous Replication**: Transactions complete on the primary before being sent to replicas. Better performance but may lead to lag or data loss in failures.

### 3. Demo
- Practical demonstration using **PostgreSQL 13** to showcase how replication works.

### 4. Pros and Cons of Replication
- **Pros**: High availability, improved read performance, disaster recovery.
- **Cons**: Complexity, potential consistency issues, increased operational overhead.

---

A concise but content-rich overview of replication in modern database systems.

## Master-Standby Replication

### Overview
Master-Standby (also known as Master-Backup or Master-Follower) replication is a model where:

- A **single master node** handles **all write operations** (DML and DDL).
- One or more **standby nodes** (replicas) receive and apply changes from the master.
- **Clients write only to the master**, while **reads can be performed on either** the master or standby nodes.

---

### Write vs Read Behavior

- **Write Operations (DML/DDL)**:
    - Performed **only on the master node**.
    - Includes operations like `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`, `ALTER DATABASE`, etc.
    - Standby nodes receive changes from the master via **replication streams** over TCP.

- **Read Operations**:
    - Can be done from either the master or any standby node.
    - Useful for distributing read-heavy workloads.

---

### Key Advantages

- **Simplicity**: No write conflicts since only one node accepts writes.
- **Read Scalability**: Read traffic can be distributed across replicas.
- **Geographic Distribution**:
    - Replicas can be placed in different regions (e.g., Asia, Europe, US).
    - Offers low-latency reads across the globe.
- **Eventual Consistency**:
    - Standby nodes may lag slightly behind the master.
    - Suitable for applications that can tolerate slight delays in data propagation.

---

### Replication Mechanics

- The master node writes changes to a **Write-Ahead Log (WAL)**.
- Standby nodes **stream and apply** these WAL changes via a continuous TCP connection.
- Replication users with appropriate privileges handle this synchronization.

---

### Synchronous vs Asynchronous

- **Synchronous Replication**:
    - The client waits until the data is written to both the master and one or more replicas.
    - Ensures stronger consistency.

- **Asynchronous Replication**:
    - The client proceeds after writing to the master.
    - Replicas catch up later, resulting in faster performance but possible lag.

> ⚠️ The replication mode (sync vs async) affects latency, consistency, and failure handling.
> This will be discussed in more detail separately.

---

### Summary

- Master-Standby is ideal for high availability, fault tolerance, and scaling reads.
- It introduces **eventual consistency**, even in relational databases like PostgreSQL.
- Write simplicity and read distribution make it a popular replication model.

## Multi-Master Replication

### Overview
Multi-Master Replication allows **multiple nodes to accept write operations**, unlike the Master-Standby model where only one node handles writes.

### Purpose
- Designed to **scale write operations** across multiple database nodes.
- Useful when a single master node becomes a bottleneck for write-heavy workloads.

---

### Key Characteristics

- **Multiple Master Nodes**:
  - Each node can accept writes and propagate changes to the others.
  - All nodes are considered peers in terms of write capability.

- **Conflict Possibility**:
  - Since multiple nodes can handle writes independently, **write conflicts** can occur.
    - Example: Two nodes writing different values for the same row at the same time.
  - Requires **conflict detection and resolution** mechanisms, which add complexity.

- **Complexity**:
  - Significantly more complex to implement and manage than Master-Standby.
  - Increased chances of data inconsistency if not handled properly.

---

### Implementation Considerations

- **Conflict Resolution**:
  - Must define rules or logic for resolving data conflicts.
  - Options include last-write-wins, custom merge logic, or application-level resolution.

- **Use Cases**:
  - Rarely used in general-purpose relational databases due to complexity.
  - More common in specialized distributed databases or NoSQL systems that support eventual consistency.

- **Alternative Strategy**:
  - You can still have multiple **Master-Standby** setups.
  - Each master node has its own set of followers, avoiding direct write conflicts.

---

### Summary

- **Pros**:
  - Enables horizontal scaling of write workloads.
  - Supports high availability in geographically distributed systems.

- **Cons**:
  - Complex to implement and maintain.
  - Prone to **write conflicts** and **data inconsistency** if not carefully managed.
  - Most relational systems prefer Master-Standby due to its simplicity.

> ⚠️ Multi-Master is powerful but should be used only when absolutely necessary and with full awareness of its trade-offs.


## Synchronous vs Asynchronous Replication

### Overview
The difference between synchronous and asynchronous replication lies in **when** the client is notified that a write operation is complete, and **how** consistency is enforced between the master and replicas.

---

### 🔄 Synchronous Replication

- **Client write flow**:
  1. Client writes to the **master** node.
  2. The write is committed on the master.
  3. The **replication is sent to standby nodes**.
  4. Only **after replicas acknowledge the write**, the client is unblocked.

- **Consistency**:
  - Ensures **strong consistency**.
  - All nodes have the latest data before the client proceeds.
  - Ideal for critical systems where **data accuracy** is more important than latency.

- **Latency Trade-off**:
  - Increases latency due to waiting on replicas.
  - Can affect write throughput.

- **Replica Acknowledgment Options** (e.g., in PostgreSQL):
  - Wait for **all** replicas (high consistency, low performance).
  - Wait for **a subset** (e.g., 1 or 2 replicas).
  - Assign **priority levels** to replicas (e.g., important vs. secondary replicas).

- **Example Analogy**:
  - Similar to Cassandra's **quorum-based writes**.

---

### 🚀 Asynchronous Replication

- **Client write flow**:
  1. Client writes to the **master** node.
  2. As soon as the write is committed on the master, the client is unblocked.
  3. **Replication happens in the background** to the standby nodes.

- **Consistency**:
  - Results in **eventual consistency**.
  - Standby nodes **may lag behind** the master.
  - Acceptable if the application can tolerate stale reads.

- **Performance**:
  - Higher write throughput and lower latency.
  - Suitable for systems where performance matters more than strict consistency.

- **Resource Usage**:
  - Background processes handle replication, which can consume CPU and I/O.

---

### Summary

| Feature                   | Synchronous                     | Asynchronous                      |
|--------------------------|----------------------------------|-----------------------------------|
| Client Wait Time         | Waits for replicas to commit     | Returns after master commit       |
| Consistency              | Strong consistency               | Eventual consistency              |
| Performance              | Slower writes                    | Faster writes                     |
| Use Case                 | Critical systems (e.g., finance) | High-throughput, low-latency apps |
| Resource Demand          | Higher latency, less CPU use     | Lower latency, more CPU use       |

> ⚠️ Choose the replication mode based on the application's tolerance for **latency vs. consistency**. Some databases even allow mixing the two for different replicas.

