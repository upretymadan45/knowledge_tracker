# Database Concurrency Strategies

## Core idea

Database concurrency strategies control how competing operations behave when multiple requests access or modify the same data concurrently. The main approaches are pessimistic locking, optimistic concurrency, atomic conditional updates, serializable isolation, distributed locking, idempotency, and queue-based serialization.

A key production principle is to let the database enforce data invariants whenever possible rather than adding application-level coordination unnecessarily.

## Mental model

Concurrency strategies answer different questions:

- **Prevention:** Should another transaction be prevented from modifying this resource now? → Pessimistic locking.
- **Detection:** Did someone modify the data since I read it? → Optimistic concurrency.
- **Atomic decision:** Can the database validate a condition and modify the row as one operation? → Atomic conditional update/constraint.
- **Isolation:** Should concurrent transactions behave as if executed serially? → Serializable isolation.
- **Coordination:** How do multiple application instances coordinate work? → Distributed locks, queues, or partitioning.
- **Duplicate protection:** Should repeated requests produce only one logical effect? → Idempotency.

These are complementary techniques, not mutually exclusive alternatives.

## Why it exists

Without concurrency control, two requests can read the same state and overwrite each other's work. For example, two users can both read balance 100 and independently write different results. Correct systems need explicit guarantees around conflicting operations.

## Example

### Pessimistic locking

A transaction locks the resource before performing dependent work:

```sql
BEGIN TRANSACTION;

SELECT balance
FROM Account
WHERE Id = 1
WITH (UPDLOCK, HOLDLOCK);

UPDATE Account
SET balance = balance - 80
WHERE Id = 1;

COMMIT;
```

Another transaction attempting to acquire the conflicting lock waits or fails according to the database's locking/deadlock behavior.

### Optimistic concurrency

Store a version and require the version read by the application to still match during the update:

```sql
UPDATE Account
SET Balance = @newBalance,
    Version = @newVersion
WHERE Id = @id
  AND Version = @expectedVersion;
```

If zero rows are affected, a concurrent modification occurred and the application must retry, reject, or otherwise resolve the conflict.

### Atomic conditional update

For an invariant such as inventory must remain non-negative:

```sql
UPDATE Product
SET Inventory = Inventory - 1
WHERE Id = @id
  AND Inventory > 0;
```

One row affected means the operation won; zero rows means the condition was not satisfied. Concurrent requests can arrive at the database at essentially the same time; the database's concurrency-control mechanisms ensure the statement is evaluated atomically.

## Production implications

1. **Prefer database-enforced invariants.** If the database can enforce a rule with an atomic statement or unique constraint, that is usually more reliable than an application-level lock.
2. **Use pessimistic locking when conflicts are common and the critical section is short.** It prevents competing work but can introduce blocking and deadlocks.
3. **Use optimistic concurrency when conflicts are relatively rare.** It improves concurrency but requires explicit conflict handling/retries.
4. **Use serializable isolation deliberately.** It provides strong correctness guarantees but can reduce throughput and cause blocking, deadlocks, or serialization failures.
5. **Do not introduce distributed locks merely because the application has multiple instances.** If the database is authoritative and can perform an atomic conditional write, that is often the simpler design.
6. **Use distributed locking only when coordination spans operations that cannot be safely expressed as a database invariant or when the resource being coordinated is outside the database.**
7. **Use idempotency for retries and duplicate requests.** Concurrency control alone does not solve duplicate message/request delivery.
8. **Queues and partitioning can serialize work for a resource and simplify concurrency, but the database should still protect against duplicate messages and unexpected writers.**

## Common misconceptions

- **"If two requests hit the database at exactly the same time, the database cannot decide which wins."** Wrong. Database concurrency-control mechanisms handle concurrent execution and conflicting writes according to the database engine's semantics.
- **"Optimistic concurrency prevents conflicts."** Incomplete. It normally allows concurrent work and detects a conflict at write time.
- **"Pessimistic locking is always safer."** Wrong. It can create contention and deadlocks and may be unnecessary when an atomic conditional update is sufficient.
- **"Multiple application servers require Redis/distributed locking."** Wrong. Many concurrency problems are better solved atomically in the database.
- **"Serializable means the database literally runs every transaction one after another."** Incomplete. It means the result must be equivalent to some serial execution; the implementation can use locking, MVCC, validation, or other mechanisms.
- **"A queue removes the need for database concurrency controls."** Wrong. Retries, duplicate messages, manual writes, and multiple consumers can still create conflicts.

## Related concepts

- Database isolation levels
- MVCC
- Row and key-range locking
- Deadlocks
- Optimistic concurrency tokens / rowversion
- Unique constraints
- Idempotency keys
- Distributed locking
- Message queues and partitioning

## Open questions

- How SQL Server's `UPDLOCK`, `HOLDLOCK`, and `SERIALIZABLE` interact in concrete scenarios.
- How MVCC/snapshot isolation differs from lock-based concurrency.
- How to choose between optimistic concurrency and atomic conditional updates in EF Core.
