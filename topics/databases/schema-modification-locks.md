# Schema Modification Locks and ALTER Operations

## Core idea

In SQL Server, schema-changing operations such as `ALTER TABLE` can require a **Schema Modification (`Sch-M`) lock**. Normal queries generally acquire a **Schema Stability (`Sch-S`) lock** while compiling or executing.

`Sch-M` and `Sch-S` are incompatible. Therefore, a schema modification may have to wait for active queries holding `Sch-S`, and once the `Sch-M` request is waiting or acquired, other operations requiring `Sch-S` can also be blocked.

The important point is that DDL is part of the database's concurrency and locking system; it is not automatically isolated from application traffic.

## Mental model

Think of the locks as two different requirements:

- **Sch-S:** "The schema must remain stable while I use it."
- **Sch-M:** "I need exclusive control to modify the schema."

Because these requirements conflict, an `ALTER` operation can participate in blocking chains involving ordinary application queries.

## Why it exists

SQL Server needs to prevent a query from relying on a schema definition while another operation is changing that definition. Schema locks provide coordination between metadata/schema consumers and schema-changing operations.

## Example

Suppose an application is continuously querying a table:

```sql
SELECT * FROM Orders;
```

A concurrent deployment executes:

```sql
ALTER TABLE Orders ADD CustomerReference varchar(100);
```

The `ALTER` operation may need `Sch-M`. If an active query is holding `Sch-S`, the schema modification must wait. Conversely, while the `Sch-M` request is waiting/acquired, subsequent requests that need `Sch-S` may also be blocked depending on lock acquisition and scheduling.

This can make an application appear to stop responding even though the application queries themselves did not explicitly request an exclusive data lock.

## Production implications

Schema changes must be treated as production concurrency events.

- A migration can block application queries.
- Long-running queries or transactions can delay schema changes.
- A waiting schema modification can contribute to a wider blocking chain.
- Deployment pipelines should account for database lock behavior.
- Large or expensive schema changes require careful planning, especially on high-traffic tables.
- Monitor blocking and lock waits during production migrations.
- Do not assume that a DDL statement is harmless simply because it changes metadata or is executed only once.

The exact locking behavior depends on the specific DDL operation, SQL Server version, transaction state, indexes, table size, and execution plan. The `Sch-M`/`Sch-S` model is the core mental model, not a claim that every `ALTER` statement has identical runtime behavior.

## Common misconceptions

- **"ALTER only changes metadata, so it cannot block SELECT."** Wrong. Schema locks can cause blocking.
- **"SELECT only takes data locks."** In SQL Server, queries also commonly acquire `Sch-S` locks.
- **"Once ALTER starts, every request is immediately blocked."** Not necessarily. Blocking depends on when locks are requested, held, and granted.
- **"Sch-M is the same as an exclusive row/data lock."** Wrong. `Sch-M` is a schema-level lock with different semantics.
- **"All ALTER statements behave identically."** Wrong. The locking and execution characteristics depend on the operation and database state.

## Related concepts

- SQL Server lock compatibility
- Shared (`S`) and exclusive (`X`) data locks
- Schema Stability (`Sch-S`)
- Schema Modification (`Sch-M`)
- Blocking
- Deadlocks
- Transactions
- Online schema changes
- Database migration strategies

## Open questions

- How lock compatibility determines the exact blocking chain.
- How long-running transactions interact with `Sch-M` acquisition.
- How online versus offline index/schema operations change blocking behavior.
- How SQL Server's lock manager handles queued incompatible lock requests.
