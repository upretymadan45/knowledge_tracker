# Database Sharding and Partitioning

## Core idea

Partitioning and sharding both distribute data, but they solve different problems.

- **Partitioning:** splits one logical table into physical partitions, typically within the same database/node.
- **Sharding:** distributes data across independent database instances/nodes.

Partitioning is primarily a database data-organization and lifecycle technique. Sharding is primarily a horizontal scalability and distributed-systems technique.

They are complementary and can be combined: multiple database shards can each contain partitioned tables.

## Mental model

Think of the distinction as:

```text
Partitioning:

Application
    |
    v
  One DB
    |
  Orders
  / | \
 P1 P2 P3
```

versus:

```text
Sharding:

Application
   / | \
 DB1 DB2 DB3
```

A useful decision hierarchy is:

1. Can one database handle the workload? Keep a single database.
2. Is a large table causing query or maintenance problems? Consider partitioning.
3. Is the database server itself the scalability boundary (CPU, memory, IOPS, storage, connections, or throughput)? Consider sharding.

## Why it exists

Partitioning can improve partition pruning, data lifecycle operations, and management of very large tables. Sharding allows database capacity and workload to scale horizontally when a single database node is no longer sufficient.

## Example

### Range partitioning

A time-based Orders table can be partitioned by `OrderDate`:

```text
Orders
  |- 2024
  |- 2025
  |- 2026
  `- 2027
```

A query restricted to a date range can potentially touch only relevant partitions.

### Hash partitioning

Rows can be distributed using a hash of a key:

```text
hash(CustomerId) -> partition
```

This can balance rows well but is less suitable for range queries.

### Tenant sharding

A SaaS system can route tenants to independent databases:

```text
Tenant A -> DB1
Tenant B -> DB1
Tenant C -> DB2
Tenant D -> DB3
```

A large tenant can later be moved to its own shard.

### Logical shard mapping

A production-friendly routing layer can separate logical shards from physical databases:

```text
Customer/Tenant
      |
      v
Logical Shard
      |
      v
Shard Map
      |
      v
Physical DB
```

This makes rebalancing and moving data between physical nodes much easier.

## Available strategies

### Partitioning

- **Range:** split by ranges such as dates or numeric IDs. Excellent for time-series and lifecycle operations.
- **List:** split by explicit categories such as region or tenant group. Useful when categories are stable.
- **Hash:** hash a key to distribute rows relatively evenly and avoid hotspots.
- **Composite:** combine strategies, such as range by year and hash within each range.
- **Vertical partitioning:** split columns into separate tables when different column groups have different access patterns. This is not the same as horizontal partitioning or sharding.

### Sharding

- **Range-based:** ranges of keys map to shards. Simple routing and efficient range queries, but sequential workloads can create hot shards.
- **Hash-based:** a hash distributes keys across shards. Good balance, but naïve `hash(key) % N` makes adding/removing shards expensive.
- **Consistent hashing:** reduces the amount of data that must move when the physical shard set changes.
- **Directory-based:** maintain an explicit key-to-shard mapping. More operational state, but flexible placement and migration.
- **Tenant-based:** route complete tenants to shards. Often a strong fit for SaaS because transactions and data ownership naturally align with tenant boundaries.

## Production implications

### Choosing a partition key or shard key

Do not choose a key only because it distributes rows evenly. Consider:

- access/query patterns
- write distribution
- traffic distribution
- transaction boundaries
- data locality
- tenant boundaries
- range-query requirements
- future growth and resharding

Traffic distribution matters more than merely achieving an even row count. A single high-volume tenant can create a hot shard even when row counts are perfectly balanced.

### Sharding creates distributed-system problems

After sharding, operations that were local become distributed:

- cross-shard queries require fan-out and aggregation
- cross-shard transactions are difficult and may require distributed transaction protocols or application-level workflows such as sagas
- cross-shard foreign keys generally cannot be enforced like local database constraints
- global uniqueness requires additional coordination or a globally routable identity strategy
- schema migrations must be coordinated across many databases
- backup and restore strategy becomes more complex
- shard failure becomes partial system failure
- resharding requires data movement and careful routing changes

Do not shard simply because a table is large. Shard when the database node itself has become the scalability boundary and the workload justifies the distributed-systems cost.

### Prefer database-enforced invariants

If a correctness rule can be enforced locally with a database constraint or atomic operation, keep it local. Sharding should not be used to solve a problem that a single database can solve more simply.

### Avoid making OLTP shards the analytics system

Cross-shard reporting can cause expensive fan-out queries. A common production pattern is to feed data from OLTP shards into a warehouse or analytical system using CDC/events and run reporting there.

### Resharding must be designed before it is needed

A direct `hash(key) % N` scheme is difficult to resize because changing `N` changes the destination for many keys. Logical shards, consistent hashing, or a directory-based shard map provide better migration options.

## Common misconceptions

- **"Partitioning and sharding are the same thing."** Wrong. Partitioning can occur within one database; sharding distributes data across independent database nodes.
- **"A large table automatically needs sharding."** Wrong. A large table may only need indexing, query optimization, partitioning, archival, or more vertical capacity.
- **"Partitioning distributes database CPU across servers."** Not by itself. If all partitions live on one database node, the node remains the capacity boundary.
- **"Hash sharding guarantees good production performance."** Incomplete. It can distribute keys evenly while still creating hot shards due to uneven traffic patterns.
- **"Sharding removes database concurrency problems."** Wrong. Each shard still needs normal transaction, locking, isolation, uniqueness, and idempotency controls.
- **"A queue or shard router eliminates the need for database constraints."** Wrong. Unexpected writers, retries, duplicate messages, and operational actions still require data-level protection.
- **"Cross-shard queries are just normal SQL queries."** Wrong. They require routing, fan-out, aggregation, failure handling, and potentially consistency decisions.

## Related concepts

- Database partition pruning
- Range, list, and hash partitioning
- Horizontal vs vertical partitioning
- Consistent hashing
- Shard maps and routing layers
- Multi-tenant database architecture
- Distributed transactions
- Sagas
- CDC and data warehouses
- Hot partitions and hot shards
- Resharding and data migration

## Open questions

- How SQL Server and PostgreSQL implement table partitioning internally.
- How to build a production shard router and logical shard map.
- How to perform online resharding with minimal downtime.
- How cross-shard queries and transactions should be designed in an application.
- How partitioning interacts with indexes, query plans, and maintenance operations.