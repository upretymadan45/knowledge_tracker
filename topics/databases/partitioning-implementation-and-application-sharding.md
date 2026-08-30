# Partitioning Internals and Application Sharding

## Core idea

Database partitioning and application-level sharding operate at different layers.

- SQL Server and PostgreSQL keep the logical table abstraction while physically storing rows in partitions. The database owns tuple/row routing and partition pruning.
- Application sharding moves ownership of data across independent database instances. The application or a routing layer must determine the target shard, maintain shard metadata, and handle distributed operations.

This means database partitioning is mostly a database-engine concern, while sharding becomes an application/distributed-systems concern.

## SQL Server partitioning internals

SQL Server uses three primary objects:

1. **Partition function** — defines how values of the partitioning column map to partition numbers and defines range boundaries.
2. **Partition scheme** — maps those partition numbers to filegroups.
3. **Partitioned table/index** — stores the table or index using the partition scheme and partitioning column.

SQL Server can place partitions on one filegroup or multiple filegroups. Multiple filegroups can be useful for storage tiering and independent backup/restore, but simply having multiple filegroups is not required to obtain the normal benefits of partitioning. citeturn0search1turn0search2

Conceptually:

```text
partition column value
        |
        v
Partition Function
        |
        v
partition number
        |
        v
Partition Scheme
        |
        v
Filegroup / storage
```

`RANGE LEFT` and `RANGE RIGHT` determine which side of a boundary owns the boundary value. SQL Server's `$PARTITION` function exposes the partition number to which a value maps. citeturn0search2turn0search7

### Partition elimination

When a query predicate references the partitioning column in a form the optimizer can reason about, SQL Server can eliminate partitions that cannot contain qualifying rows. This is commonly called **partition elimination**. citeturn0search2

Partitioning does not automatically make every query faster. A query that does not constrain the partition key may still need to access many or all partitions.

### Index alignment

SQL Server indexes can be aligned with the table's partitioning scheme. Alignment is important for operations such as partition switching and maintenance. Unique indexes have additional requirements involving the partitioning column when uniqueness must be preserved across the partitioned object. Nonaligned indexes are possible but add design complexity. citeturn0search2

## PostgreSQL partitioning internals

PostgreSQL declarative partitioning uses a **partitioned table** as a logical/virtual parent. The parent itself does not contain the table's row storage; ordinary child tables called partitions contain the actual data. The parent defines the partitioning method, key, and partition bounds. citeturn0search0

Conceptually:

```text
Partitioned Table (logical parent)
          |
     partition bounds
      /      |      \
   child1  child2  child3
    table   table   table
```

Supported declarative methods include range, list, and hash partitioning. Partitions can themselves be partitioned, allowing sub-partitioning. Inserts into the parent are internally routed to the partition whose bounds accept the row; if no matching partition exists, the insert fails unless an appropriate partition/default partition is available. Updating a partition key can move a row to another partition when its old partition no longer accepts it. citeturn0search0

### Partition pruning

PostgreSQL's optimizer can inspect partition bounds and eliminate partitions that cannot satisfy a query predicate. PostgreSQL can perform pruning at planning time and, in supported cases, during execution when values are only known at execution time, such as prepared-statement parameters. citeturn0search0turn0search3

Partition pruning is based on partition bounds, not on indexes. Indexes on individual partitions are still useful when the query accesses a small fraction of rows within a selected partition. citeturn0search3

### Legacy inheritance partitioning

PostgreSQL historically supported manual partitioning using table inheritance and `CHECK` constraints. Constraint exclusion can eliminate child tables based on those constraints, but it is a different mechanism from declarative partition pruning and is generally less attractive for new designs. Declarative partitioning performs tuple routing internally and avoids the need for user-written triggers/rules used by older approaches. citeturn0search0turn0search3

## What changes when the application is sharded

With one database, the database can resolve:

```text
logical table -> physical partition
```

With sharding, the application must resolve:

```text
business key -> logical shard -> physical database
```

A production architecture should avoid embedding physical database names directly throughout application code.

Preferred model:

```text
Request
  |
  v
Shard Key
  |
  v
Shard Router
  |
  v
Shard Map
  |
  v
Logical Shard
  |
  v
Physical Database Endpoint
```

## Real application shard router

A router should have one responsibility: determine the correct database target from the shard key and current shard metadata.

Example interface:

```csharp
public interface IShardRouter
{
    ValueTask<ShardTarget> ResolveAsync(
        ShardKey key,
        CancellationToken cancellationToken);
}
```

Application services should depend on the router rather than constructing connection strings from tenant IDs themselves.

Example flow:

```text
HTTP request
    |
    v
TenantId = 42
    |
    v
ShardRouter.Resolve(42)
    |
    v
LogicalShard = 17
    |
    v
ShardMap[17]
    |
    v
DB-03
```

The router should cache shard metadata but must have a reliable invalidation/refresh strategy because migrations and resharding change ownership.

## Shard map

A shard map is authoritative metadata describing ownership and routing. A useful model is:

```text
LogicalShardId
ShardKeyRange / TenantId
PhysicalShardId
Endpoint
State
Version
```

Typical states include:

```text
Active
Moving
ReadOnly
Draining
Offline
```

A logical shard layer is preferable to directly mapping customer IDs to physical database names because physical placement can then change without changing business-level routing semantics.

For example:

```text
Tenant 42
   |
Logical Shard 17
   |
   +---- currently ---> DB03
   |
   +---- after migration ---> DB08
```

## Resharding

Resharding is the process of moving ownership of keys/data between physical shards while the system continues operating.

A production migration should not be treated as a single database rename. It is a state transition with data movement and routing coordination.

A safer high-level flow is:

```text
1. Select logical shard(s) to move
2. Mark shard as MOVING
3. Establish destination
4. Copy historical data
5. Capture/copy changes made during migration
6. Validate source and destination
7. Quiesce or coordinate writes for cutover
8. Atomically update shard-map ownership
9. Route new requests to destination
10. Verify production traffic
11. Retain source for rollback window
12. Remove old copy after validation/retention period
```

For large datasets, online migration generally needs a mechanism equivalent to change capture/change replication so that the destination catches up while the source remains live.

### The key invariant during migration

At cutover, there must be one authoritative owner for a logical shard.

Avoid a period where both source and destination independently accept writes unless the design explicitly implements conflict resolution. Otherwise the migration can create divergent state.

### Migration metadata

Treat shard-map changes as versioned configuration/data, not an ad-hoc cache mutation.

A useful model is:

```text
ShardMapVersion = 184
LogicalShard 17 -> DB08
```

Routers can refresh when their local version is stale.

## Cross-shard queries

The easiest cross-shard query is one that can be routed to a single shard.

For example, if all tenant data is colocated:

```sql
SELECT *
FROM Orders
WHERE TenantId = @tenantId;
```

The router resolves the tenant before opening the connection.

The difficult case is a query without a shard key:

```sql
SELECT SUM(Amount)
FROM Orders
WHERE OrderDate >= @from
  AND OrderDate < @to;
```

If Orders are sharded by TenantId, the application may need to fan out:

```text
                 Query
                   |
        +----------+----------+
        |          |          |
       DB1        DB2        DB3
        |          |          |
      SUM1       SUM2       SUM3
        \          |          /
         +---------+---------+
                   |
                SUM(total)
```

The production pattern is usually:

1. Resolve the target shard set from the shard map.
2. Send independent queries in parallel with bounded concurrency.
3. Apply per-shard timeouts/cancellation.
4. Collect partial results.
5. Aggregate results deterministically.
6. Define what happens when one shard fails.

Do not blindly fan out to hundreds of shards from a request thread. Use bounded concurrency, timeouts, circuit breaking where appropriate, and preferably a separate analytical path for broad reporting workloads.

## Cross-shard consistency

A fan-out query is not automatically one globally consistent snapshot. Each database can observe a different point in time unless the architecture provides a distributed consistency mechanism.

For dashboards and analytics, eventual consistency through CDC into a warehouse is often preferable to forcing globally consistent OLTP queries across shards.

## Cross-shard transactions

A local transaction such as:

```text
Create Order
+ Reserve Inventory
```

is simple when both records live on one database.

If they live on different shards, ordinary local database transactions cannot provide the same boundary. Options include:

- redesigning the shard key to colocate transactionally related data
- distributed transactions/two-phase commit where justified
- saga/workflow patterns
- transactional outbox plus asynchronous processing
- compensating actions

The first option is often the best architectural answer: choose the shard key around transaction locality rather than only around even distribution.

## Production design rules

1. **Keep the shard key explicit.** Every service that accesses sharded data should know which business key determines ownership.
2. **Centralize routing.** Do not scatter shard-selection logic through repositories and controllers.
3. **Separate logical and physical shard identity.** This makes resharding possible without changing business identifiers.
4. **Make shard-map changes versioned and auditable.** Routing is critical infrastructure.
5. **Design for partial failure.** A failed shard should affect only the data/workloads dependent on it where possible.
6. **Prefer shard-local transactions.** If a transaction routinely crosses shards, revisit the shard key.
7. **Keep global reporting off the OLTP request path.** Use CDC/events and a warehouse for broad analytics where possible.
8. **Plan resharding before production scale requires it.** A scheme that cannot move data safely is an operational dead end.
9. **Monitor both row distribution and traffic distribution.** Even row counts do not prevent hot shards.
10. **Test migration and rollback.** Resharding is a production data-movement operation, not just configuration deployment.

## Common misconceptions

- **"A partition is another database server."** Wrong. Database partitions are normally storage/organization units within a database; sharding distributes across independent database nodes.
- **"The database router and application shard router are the same thing."** Wrong. SQL Server/PostgreSQL internally route rows to partitions; application sharding requires application/infrastructure-level routing between independent databases.
- **"Partition pruning means the query is automatically fast."** Wrong. It only reduces the partitions considered; indexes, row counts, predicates, joins, and physical resources still determine performance.
- **"Hash modulo is enough for production sharding."** Often wrong. Changing shard count can remap a large percentage of keys, making resharding painful.
- **"Resharding is just copying tables and changing a connection string."** Wrong. Safe resharding requires ownership state, data synchronization, cutover, validation, and rollback planning.
- **"Cross-shard aggregation is just a normal database query."** Wrong. It is a distributed fan-out operation with network, timeout, partial-failure, and consistency concerns.
- **"A distributed transaction is the normal answer to every cross-shard transaction."** Wrong. Prefer shard-key redesign/co-location or asynchronous workflow patterns when possible.

## Related concepts

- SQL Server partition functions and partition schemes
- SQL Server partition elimination and partition switching
- PostgreSQL declarative partitioning
- PostgreSQL partition pruning
- Logical shard maps
- Consistent hashing
- Tenant isolation
- Online data migration
- CDC and change capture
- Distributed transactions
- Sagas and transactional outbox
- Fan-out/fan-in queries

## Open questions

- Build a concrete SQL Server example showing partition function, scheme, aligned indexes, and partition switching.
- Build a concrete PostgreSQL example showing range/hash partitions, pruning, indexes, and detach/attach operations.
- Design a production .NET shard-router implementation with connection pooling and shard-map caching.
- Design an online resharding workflow with CDC, cutover, validation, and rollback.
- Design bounded-concurrency cross-shard query execution and failure semantics.