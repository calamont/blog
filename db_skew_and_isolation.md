# Database Skew and Isolation Levels

## Notes from Ch.7 DDIA
> Transactions allow db to take care of potential error scenarios and concurrency issues.

> Some safety properties can be achieved without transactions.

Need multi-object transaction if inserting into several related tables at once.
- But it does call on us to minimise the number of related objects being inserted/updated by limiting the scope of the aggregate.

Also need multi-object transactions if updating secondary indexes.

Error handling becomes much more complicated without atomicity.

> Concurrency issues only come into play when one transaction reads data that is concurrently modified by another transaction, or when two transactions try to simultaneously modify the same data.

> In theory, isolation should make your life easier by letting you pretend that no concurrency is happening.

> Several other interesting kinds of conflicts that can occur between concurrently writing transactions.

> Unfortunately, object-relational frameworks make it easy to accidentally write code that performs unsafe read-modify-write cycles instead of using atomic operations provided by the database.

> This effect, where a write in one transaction changes the result of a search query in another transaction, is called a _phantom_.

> Repeatable read isolation only prevents phantoms in read-only transactions/queries.

---

## Format
### Intro
Concurrency isn't limited to multi-threaded applications. Think about design of system.
As your application servers horizontally scale, the db will start handling concurrent reads and writes.
The db tries to simplify this by providing varying isolation guarantees.
Each of these guarantees has trade-offs.
It is not clear when each is needed.
And it is not always obvious the issues concurrency will give you.
I will list a number of scenarios where concurrency could cause you headaches, and what level of isolation is needed to avoid it.
TODO: And I will offer an alternative system design solution to avoid this altogether? Might be too awkward or contrived.

### Isolation
Describe isolation levels

Read uncommitted
- Not particularly useful?
- Allows dirty reads and writes.
- Not actually implemented in PostgreSQL.

Read committed
- Prevents dirty reads and writes.
- PostgreSQL actually uses same MVCC that repeatable read uses, but makes the rules of what a transaction can see less strict.

Repeatable read
- Prevents non-repeatable reads, such as read skew.
- Useful if transaction has long running read-only transactions.
  - Back ups and analytical queries.
- In some engines (e.g. PostgreSQL) it will prevent lost updates.

### TODO: Concurrency issues
Maybe section explaining concurrency issues.
- Dirty reads
  - Prevented by read committed
- Dirty writes
  - Prevented by read committed
- Non-repeatable read
  - A transaction re-reads data and the value has been changed from a concurrent transaction committing.
  - Prevented by repeatable-read isolation?
  - Read skew is a type of non-repeatable read. Your system may be able to tolerate this temporary inconsistency.
- Write skew
  - Includes lost updates.
    - Caused by read-modify-write cycle. This may not be avoidable, such as updating a complex value, like JSON. Not sure if PostgreSQL has support for updating this in place.
    - Some approaches for avoiding it:
      - Atomic commits. Depends on the database support for this operation. This blocks readers. Managed by the database.
      - Explicit locking. Instead of the database providing support, the application explicitly locks the objects that are going to be updated e.g. `FOR UPDATE` - This blocks readers. Managed by the application.
      - Automatically detecting lost updates. This allows concurrent transactions to execute and one is rolled back if the transaction manager detects a lost update. TODO: Does this check if constraints are invalidated though? Managed by the database.
      - Compare and set. Managed by the application.
  - Write skew is a generalisation of the lost update problem. But is not always detectable. As such implementing repeatable read in PostgreSQL won't prevent it.
    - Some of the discussed approaches above for lost update _might_ prevent it, but it depends on the situation.
      - A `FOR UPDATE` clause might prevent it if the search query is not looking for the _absence_ of a value. But if the positive result that then allows a write to occur is if there is an absence of a value, it won't work, because the `FOR UPDATE` has no rows to lock.
- Phantom read
- Phantom write?

### Data access patterns and concurrency
Go through various examples of accessing data and what issues concurrency can bring.
TODO: Provide step by step examples to replicate locally?

### TODO: Improve model design
Maybe a section about how you should think about your model domain. If multiple parts of your app are accessing the same table, maybe your domain models haven't been designed well?

---

The previous sections discussed factors that influence what database to use. From here we will consider how to best configure that database for your use case.

TODO: Below would be useful in the introduction. It sets up why this is important. Even single threaded applications face issues with concurrency because of the database.
Postgres forks a new process on each new connection, up to a configurable limit. If our backend app is multi-threaded, multi-processed, or horizontally scales, multiple concurrent connections can be made to the database. If we are running a multithreaded or horizontally scaled app, we can still serve multiple requests against a single database instance.

TODO: Give definitions of skew?
- Read skew
- Write skew
  - Lost update
  - Phantom
  - More?
- More?

We start with the simplest scenario, our app connecting to a single database instance. But a single instance can receive multiple connections and we must consider how it manages concurrent updates.

What level is suitable is affected by our read/write appetite.

The database isolation level describes how it manages concurrent updates.
If our application is write-heavy, we need to consider how data will be accessed, and how frequently.

The isolation controls how the database preserves data integrity in the face of concurrency. The common options are:
**Read committed**
- It is only possible to read values from committed transactions. It is also not possible to overwrite values that haven’t been committed yet.

**Repeatable read (or snapshot isolation)**
- Readers never block writers, and writers never block readers

**Serialisable**
- If using two phase locking, writers will block readers, and vice versa.

If our application is read-heavy, less consideration about isolation is required. However, if you require read only transactions (e.g. fetching many values from un-joinable tables), repeatable read allows you to avoid read skew.

If we have a read-heavy workload, it may make sense to only use read committed. As the write load increases, so too does the chance of introducing read skew at this isolation level.

## Common access patterns
### Inserting or updating individual new records in a single query.
Read committed is probably suitable.

### Inserting or updating multiple new records in a single query.
Need read committed. Otherwise another part of your app may only see part of the updates. But why is this an issue? Maybe because it is a related model and foreign keys will be pointing to something that doesn't exist?

### ???
TODO: have example where repeatable read is useful, but don’t need serialisable

### Read a record into memory then update it’s value.

```sql
BEGIN TRANSACTION;

SELECT * FROM accounts WHERE id = 123;

-- Calculate new balance as prev + 100 USD

UPDATE accounts SET balance = 500 WHERE id = 123;

COMMIT;
```

This risks a lost update, a type of _write skew_.

Two overlapping transactions do not see their respective updates, causing the update from the first transaction to be lost once the second transaction is committed.

If not managed, this could cause lost updates. Two transactions update the same record with different values. The persisted value will be from the transaction that commits last. Importantly, the same record is being updated by both transactions.

This could be avoided by:
- Using an explicit lock using FOR UPDATE.
- Serialisable isolation
- Using an atomic write operation: UPDATE accounts SET balance = balance + 100 WHERE id = 123;

This may not be possible due to the nature of the update, or constraints from the database.

### Read a set of records and update one

```sql
BEGIN TRANSACTION;

SELECT * FROM users WHERE username = "calamont"; --TODO: check which quotes

-- Check if no users are using that name, if so, abort.

UPDATE users SET username = "calamont" WHERE id = 123;

COMMIT;
```

The above query pattern exposes you to a phenomenon known as write skew. Two concurrent transactions:
1. Read an overlapping set of records.
2. Check some invariant (e.g. uniquness of username).
3. Updates a different item from the set of records read (depending on the outcome of Step 2).

The issue is that each transaction is unaware of what the other transaction is doing, and that the result of one may invalidate the invariant. Generally, it is only possible to maintain such an invariant with serialisable isolation.

But serialisable isolation imposes a severe performance penalty. In the above example it would lock the entire users table for every write. Try to enforce this invariant through other means, such as a unique constraint on a table. But if this is not possible, and the invariant is critical to your application, then serialisability is your only option.

If the stored data must maintain some invariant, the above type of transaction can cause write skews.

### Updating existing records, using values from another record.

### Updating existing records, using an aggregated value from another record.

### Inserting new records, conditional on the result of a previous select query.

### Writes to a table after performing a SELECT to check if a constraint is satisfied.
