# Database Skew and Isolation Levels

Do you need to worry about concurrency when designing a single-threaded web app? Yeah, if you're planning to ever scale, at all. When you scale, your database will certainly handle concurrent queries. I would wager this occurs from the very outset, because scaling comes in many flavours:
- A single worker on a single server can use async/await to make concurrent requests to the database.
- Even a single web server may use multiple threads or workers to accept traffic in parallel.
- Your app can horizontally scale, so more servers are accepting traffic.

When you interleave concurrent reads and writes to your database the results can be surprising. The results can also break your app. Let's prevent that from happening! All of these common query patterns create concurrency headaches:
- Batch inserting or update of multiple new records in a single transaction.
- Updating multiple tables on an event.
- Enforcing a constraint when reading data across the same or related tables.
- Long running analytic queries.
- Updating a value that requires application logic.
- Updates to a non-scalar data type.
- Reserve a resource for an object if it's available.
- Insert/Updating a row conditional on the aggregate of multiple rows.

I'm sure some look familiar. Some might look innocuous. Transactions give a false sense of security that these types of operations "will just work". But what actually happens when two transactions occur concurrently? We can't say without considering what [isolation level](https://en.wikipedia.org/wiki/Isolation_(database_systems)) is enforced. And that choice can summon phenomena that will haunt your application.

## Dirty writes and dirty reads
### What are they?
Dirty writes and reads are the easiest phenomena to grasp. They occur when a concurrent database transaction can read or write uncommitted data to the database.

### Example query patterns
#### Batch insert or update of multiple records in a single transaction
```sql
BEGIN;
INSERT INTO my_table VALUES (1, 'foo');
INSERT INTO my_table VALUES (2, 'bar');
INSERT INTO my_table VALUES (3, 'baz');
COMMIT;

-- Or even...

INSERT INTO my_table (id, val)
VALUES
    (1, 'foo'),
    (2, 'bar'),
    (3, 'baz');
```
How harmless, you think. But without any enforced isolation, the new records written are available in the table before the transaction has finished (dirty write). These _uncommitted_ updates may be read by other transactions (dirty read). If the transaction writing the data fails, all inserts will be rolled backed. Any transaction that already read these rows will be holding invalid data that no longer exists in the table.

Assuming _no_ isolation protection, this can even occur for a multi-row insert/update query. The database cannot magically insert all the data instantaneously and commit. Data may be scattered across multiple pages in memory. Some rows will be inserted or updated before others. Only once all rows in all pages have been updated will the database commit the result. Don't let the declarative SQL syntax fool you.

### How to ensure consistency
In practice, you don't need to worry about dirty reads/writes when using a popular RDBMS. All use a default isolation level of **READ COMMITTED** at minimum. As the name implies, this ensures transactions can only read _committed_ data, which can't be rolled back. In Postgres, dirty reads are impossible: there is no **READ UNCOMMITTED** isolated level.

But NoSQL databases are different. Often, these do not have transactional guarantees. Even for a batch insert operation, records will probably be available to read one at a time, as each is inserted. This may not be an issue for your application, but worth noting.

## Read skew
### What is it?
**Read skew** is more interesting, and won't be prevented by **READ COMMITTED** isolation. To generalise, one transaction makes multiple read queries and receives data that is inconsistent. The simplest example is a non-repeatable read:
1. Transaction 1 reads row X from a database.
2. Transaction 2 commits an update to row X.
3. Transaction 1 re-reads row X and receives a different value than in step 1.

Even though transaction 1 is wrapped in an atomic transaction, it can view any committed changes to the database, including commits that were made since it started. As described, a non-repeatable might feel contrived. But read skew doesn't only involve re-reading the same row.

### Example query patterns
#### Reading multiple rows while enforcing a business rule across them
**Reading rows across related tables**
We have two tables: `employee` and `salary`. To fetch the salary information for an employee, we might perform a query like:
```sql
SELECT employee.id, employee.name, employee.title, salary.base salary.bonus
FROM employee
JOIN salary ON employee.id = salary.employee_id
WHERE employee.id = 123
```
The company enforces salary bands for each job title. For this example, let's assume the maximum salary for an `analyst` is $70,000. The application code expects the stored data to respect this rule. By distributing job and salary information across two tables, it is possible to read inconsistent data if a concurrent transaction updates these tables at the same time.

Imagine this scenario:
| Time | Transaction 1 | Transaction 2 |
| :- | :------------- | :------------- |
| **T1** | `BEGIN;` | `BEGIN;` |
| **T2** | Database receives `SELECT` query | |
| **T3** | Database reads a row from the `employee` table:<br>`(id=123, name='Jane Smith', job_title='analyst')` | |
| **T4** | | Employee `123` is promoted to **_senior_** analyst and receives a pay rise to $80,000. A concurrent transaction commits these changes to the `employee` and `salary` tables with this information |
| **T5** | | `COMMIT;` |
| **T6** | The ongoing transaction reads a row from the `salary` table:<br>`(salary=80,000, bonus=5,000)` | |
| **T7** | The database joins the two rows:<br>`(id=123, name='Jane Smith', job_title='analyst', salary=80,000, bonus=5,000)` | |
| **T8** | `COMMIT;` ||

The returned data violates the salary bands enforced by the company. An analyst is making $80,000. This could cause confusion in the UI, or a runtime error if the application code enforces this validation. Importantly, this effect is transient. If we performed the same query again, we would receive the correct value.

**Reading rows across a single table**

The issue above is that a JOIN between two tables doesn't occur instantaneously. This is also an issue for self-joins on a single table. The underlying reason is that rows may be distributed across different pages, which will be read in individually if they are not co-located on disk.

Imagine a bank has an `account` table, with the accounts of its customers and their current balance. Customer `123` has two accounts. $1000 in her savings account, and $200 in her spending account. She has $1200 in total. The query to generate this total could look like:
```sql
SELECT SUM(amount) FROM account WHERE customer_id = 123;
```

Even this simple query is susceptible to read skew. The page in memory for the savings account is probably different to the spending account. Imagine this scenario:
| Time | Transaction 1 | Transaction 2 |
| :- | :------------- | :------------- |
| **T1** | `BEGIN;` | `BEGIN;` |
| **T2** | Database receives the `SELECT` query | |
| **T3** | It reads in the spending account row, located in page Y on disk:<br>`(id=5123, customer_id=123, type='spending', balance=200)` | |
| **T4** | | The customer transfers $100 from her savings account to her spending account. he transaction for this transfer commits successfully. |
| **T5** | | `COMMIT;` The savings account now has $900 and the spending account has $300. Still $1200 in total. |
| **T6** | Database reads in the savings account row, located in page Z on disk:<br>`(id=7424, customer_id=123, type='savings', balance=900)` | |
| **T7** | The database sums the two rows - `200 + 900 = 1100`. | |
| **T8** | `COMMIT;` ||

The query suggests the customer only has $1100. If we reran the query, we would see the correct balance of $1200.

#### Long running analytic query
While there are better approaches, it is not uncommon for analytic queries to be run against production databases.

```sql
SELECT
  COALESCE(category, '') category,
  COALESCE(product_id, '') product_id,
  SUM(price) total_sales
FROM
  order
GROUP BY
  ROLLUP (category, product_id)
ORDER BY
  product_id;
```
These analytical queries often involve reading in the whole table to calculate business metrics. Similar to the previous query we looked at, while the database is reading in the rows of the table into memory, other transactions may be committing changes. These subsequent changes can make it seem the table is in an inconsistent state. Because analytic queries are often reading in the _whole_ table, they are running for longer and are more at risk to this effect.

### How to guard against it
If **READ COMMITTED** isolation won't protect us against read skew, what will? The next strongest isolation level for most RDBMSs is **SNAPSHOT** isolation. When a transaction starts, the database takes a "snapshot" of its state at that point. The transaction can only see committed database changes that occurred _before_ it started. Thus, we ignore commits of concurrent transactions and eliminate read skew. Note that there is not consistent labelling of isolation levels between RDBMSs. For example, the Repeatable read in PostgreSQL is actually **SNAPSHOT** isolation. Best read the docs of whichever RDBMS you are using to know what it's protecting you against.

There are different RDBMS approaches to guarantee **SNAPSHOT** isolation, and each imposes a performance cost. So consider how serious read skew is for your application. As noted earlier, the immediate effect of read skew is transient. If a transaction repeats the same query later, it will get the correct result. It may not warrant a fix if the only consequence is minor confusion for a user until they reload the page.

Read skew is more serious if it leads to side-effects that introduce **permanent changes** elsewhere. The protection provided by **SNAPSHOT** isolation may be worth the cost.

It is also possible to reconsider the queries being executed. For example, if the only query that requires **SNAPSHOT** isolation is a long-running analytic query, I would recommend ingesting this data to a downstream platform optimised for business analytics.

## Lost updates
### What are they?
Lost updates occur when one transaction clobbers the update of another:
1. Transaction 1 and 2 start around the same time.
2. Transaction 1 and 2 read the same row to update it.
3. Transaction 1 writes its update to the table and commits.
4. Transaction 2 writes its update to the table and commits.

The update from transaction 2 (step 4) overwrites the changes from transaction 1 (step 3). This could be expected and fine. Or it may lead to incorrect data. Suppose both transactions read and incremented a counter value. Transaction 1 increments 10 to 11, and transaction 2 increments 10 to 11. The final value in the table is 11, when the two increment operations should have incremented the original value of 10 to 12.

Lost updates typically follow a read-modify-write cycle, seen below.

### Example query patterns
#### Updates to a non-primitive data type
We don't usually increment counters in the application code these days. Any popular RDBMS can update integers atomically, which avoids this race condition. But there are some data types that must be updated in application code. An object serialised as anything other than JSON or XML (e.g. Protobufs, Avro) will be stored as a binary large object (`BLOB`). To update these objects, they must first be read from the database and deserialised in application code. This risks a lost update because two concurrent transactions may read and update the same record.

#### Updating a value that requires application logic
Another reason the data is read into the application is when an update is derived from business logic. This makes the data vulnerable to lost updates.

Say an e-commerce site imposes a constraint to prevent the stock count of its items being negative:
```sql
BEGIN TRANSACTION;

SELECT count FROM inventory WHERE product_id = 123;

-- Check if purchase order is still valid for current stock level.
-- If so, calculate and set new stock level.
UPDATE inventory SET count = 17 WHERE product_id = 123;

COMMIT;
```

Imagine this scenario:

| Time | Transaction 1 (Order for 10) | Transaction 2 (Order for 20) |
| :--- | :--- | :--- |
| **T1** | `BEGIN;` | `BEGIN;` |
| **T2** | Database receives the `SELECT` query for Product 123. | |
| **T3** | It reads the inventory row from disk:<br>`(product_id=123, count=25)` | |
| **T4** | | Database receives the `SELECT` query for Product 123 from a second concurrent order. |
| **T5** | | It reads the same inventory row from disk:<br>`(product_id=123, count=25)` |
| **T6** | The application calculates the new stock level: `25 - 10 = 15`. It issues the `UPDATE` query: <br>`UPDATE inventory SET count = 15 WHERE product_id = 123;` | |
| **T7** | `COMMIT;` Transaction 1 commits successfully. The database now reflects 15 items in stock. | |
| **T8** | | The application for Transaction 2 calculates its new stock level: `25 - 20 = 5`. It issues the `UPDATE` query: <br>`UPDATE inventory SET count = 5 WHERE product_id = 123;` |
| **T9** | | `COMMIT;` Transaction 2 commits successfully. The database now reflects 5 items in stock. |

The second transaction clobbers the first, so the final result in the database is a stock level of five. In fact, the real value is -5 and there isn't enough stock to fulfil both orders!

As written, a constraint on the database table won't protect you. Each update would appear valid for a non-negative `CHECK` constraint.

I'm sure you can think of a few ways to avoid this issue. The main point is that queries are vulnerable to lost updates when application logic must occur between the initial read and the update. We assume the time between the initial read and update will be short. But pauses in the application and network can occur for any number of reasons. There can be delays in packet delivery, or the application process pauses for [stop-the-world garbage collection](https://en.wikipedia.org/wiki/Tracing_garbage_collection#Stop-the-world_vs._incremental_vs._concurrent). These leave this query pattern vulnerable to lost updates. And the risk increases with the volume of traffic.

### How to guard against it
In both the examples we've seen, there is a common read-write-update cycle. If possible, avoid this pattern and favour atomic updates. When this isn't possible, other strategies are required:
- Avoiding unnecessary application logic by enforcing constraints on database (Does this actually work?).
- Uniqueness constraints.
- Pessimistic locking of the relevant rows. Often can be achieved using a statement like `SELECT ... FOR UPDATE`.
- Compare and set
    - Some databases automatically do this (Postgres in **SNAPSHOT** isolation)
TODO: Probably want to flesh out some of the above points.

TODO: Could the point below be useful in closing arguments instead of here? Make a point about how this is not just a concern of databases. It extends to all distributed systems.
Imagine a user updating a text document stored on the cloud. The browser fetches the document to display for the user. The users makes there edits, then saves the update. Their update would clobber any updates saved by other users in between. In this case, even transactions and isolation can't help you. It is not feasible to maintain a lock of an indefinite amount of time, determined by the users workflow.

## Write skew
Read skew is when the application read inconsistent data. Write skew is when the application writes to a database and leaves it in an inconsistent state. The traditional definition describes when two transactions write to different objects, and don't observe the effects of each other. This is best illustrated with an example.

### Example query patterns
#### Writing data with a business rule

A bank has a policy that an account can be negative **if** the total balance of that customer's accounts is greater than $0.

```sql
SELECT SUM(balance) FROM account WHERE customer_id = 123;
-- Application checks the total balance won't be negative after the withdrawal.
UPDATE SET balance=balance - 150 WHERE account_id = 567
```

| Time | Transaction 1 (Order for 10) | Transaction 2 (Order for 20) |
| :--- | :--- | :--- |
| **T1** | `BEGIN;` | |
| **T2** | Query `SELECT SUM(balance)...` to get customer's net balance. Currently at $250. | |
| **T3** | Verify withdrawl of $100 won't cause the total balance to be negative. | |
| **T4** | | `BEGIN;` |
| **T5** | | An automated direct debit removes $500 from one of the customer's accounts. The net balance is now -$250. |
| **T6** | | `COMMIT;`. The net balance is now -$250 |
| **T7** | Withdraw $100 from account. | |
| **T8** | `COMMIT;`. The net balance is now -$350 | |


Either the withdrawl or the automated direct debit should have been blocked. This is the example used by Berenson et al (1995) when critiquing the ANSI SQL phenomena and isolation levels.

### How to guard against it
As described by Berenson et al (1995), this phenomena is not prevented by using **SNAPSHOT** isolation. Write skew is tricky, in that the two transactions write to _different_ rows, which typically escapes conflict detection methods used during commits. Theoretically, write skew is prevented by the ANSI defined **REPEATABLE READ** isolation. But I found that the only popular RDBMS that implement this are SQL Server and IBM db2.

For other RDBMSs, there are fewer options for preventing write skew than for lost updates. You must typically resort to pessimistic locking (`SELECT ... FOR UPDATE`) or **SERIALIZABLE** isolation. This could drastically reduce performance if you need to lock a large range of rows.

Instead of looking for a technological solution, check if the phenomena can be side-stepped by improving the data modelling. For the above example, instead of constantly locking all accounts during reads with a `SELECT ... FOR UPDATE`, the aggregated net value of a customer's holdings could be stored in a separate `customer_holdings` table, which has a constraint that `total_amount` cannot be negative. The `account` and `customer_holdings` tables are updated together.
```sql
BEGIN;
UPDATE SET balance=balance - 150 FROM account WHERE account_id = 567;
UPDATE SET total_amount=total_amount - 150 FROM customer_holdings WHERE account_id = 567;
COMMIT;
```

Updates to balances are still effectively serialised, because they are all trying to update the same row in `customer_holdings`, but at least we are not blocking reads of all the customer's accounts the `account` table.

## Phantoms
### What is it?

Phantoms are changes in predicate sets, which cause a precondition to an action to become invalid. While that sounds cryptic, phantoms are real and they can break your application. Unhelpfully, they are also difficult to guard against.

### Example query patterns
#### Enforcing a uniqueness constraint
Requiring a unique username is a common constraint on the internet. Imagine two users updating their existing name to a new conflicting name: `sudo_man`.
```sql
SELECT * FROM users
  WHERE username = 'sudo_man'

-- PRECONDITION: if query result is empty, go ahead and commit the username

UPDATE users SET username='sudo_man' WHERE ...
```

If the two transactions started at the same time, the precondition would pass for both for them to take this new name. When the first transaction commits, it has changed the size of the predicate set (`WHERE username = sudo_man`) from zero to one. This addition to the set isn't seen by the second ongoing transaction, but invalidates its own precondition check. When the second transaction commits, the uniqueness constraint has been violated. If these two transactions ran serially, one after the other, the second transaction would have respected the constraint.

Clearly, this example can be avoided with a uniqueness constraint on the database column. But other phantoms don't have such convenient solutions.

#### Reserve a resource if it exists
TODO: It would be good to have an example of removing a value from a predicate set, but it isn't critical.

#### Reserve a booking if there is availability
Sometimes you need to hold or reserve a resource:
- An apartment booking on AirBnB.
- A meeting room in the office.
- A car for a car hire service.
- A seat at a concert.

There is a obvious constraint: no two bookings can clash or even overlap. A query for seating at a concert could look like this:
```sql
SELECT * FROM booking
  WHERE event_id = 123
  AND location = 'A3';

-- if query result is empty, add the new booking
INSERT INTO booking ...
```

Or if booking a resource for a set period of time, you may use a query like so:
```sql
SELECT * FROM booking
  WHERE resource_id = 123
  AND start_date < '2025-05-08'
  AND end_date > '2025-05-01';

-- if query result is empty, add the new booking
INSERT INTO booking ...
```

Imagine two clients reserving overlapping periods, concurrently.

| Time | Transaction 1 (User A: May 1st–8th) | Transaction 2 (User B: May 4th–10th) |
| :--- | :--- | :--- |
| **T1** | `BEGIN;` | `BEGIN;` |
| **T2** | `SELECT * FROM booking WHERE id = 123`<br>`AND start_date < '2025-05-08'`<br>`AND end_date > '2025-05-01';` | |
| **T3** | Database returns and empty set. The precondition passes. | |
| **T4** | | `SELECT * FROM booking WHERE id = 123`<br>`AND start_date < '2025-05-10'`<br>`AND end_date > '2025-05-04';` |
| **T5** | | Database returns an empty set. The precondition passes. |
| **T6** | `INSERT INTO booking (id, start_date, end_date)`<br>`VALUES (123, '2025-05-01', '2025-05-08');` | |
| **T7** | `COMMIT;` | |
| **T8** | | `INSERT INTO booking (id, start_date, end_date)`<br>`VALUES (123, '2025-05-04', '2025-05-10');` |
| **T9** | | `COMMIT`; |

Again, one transaction has added to a predicate set that invalidated the other transaction's precondition. To each transaction, it appears as if there is availability, so they both commit their booking. The two booking records are distinct, so there is no direct conflicts when the transactions are committed and no error is thrown by the system. However, the constraint that no two bookings overlap is broken. This couldn't have occurred if the transactions ran serially.

### How to guard against it

Our first example for creating usernames showed that, sometimes, we can lean on database guardrails to prevent phantoms. Here, all we need is a uniqueness constraint on the database column. This case is not the norm and, typically, this phenomena is difficult to prevent using anything other than pessimistic locking. Why? Because we are trying to prevent changes to predicate sets. Different rows are being updated between the concurrent transactions, which evades automatic conflict detection.

Pessimistic locking is a rather blunt tool to reach for, and for phantoms it is particularly destructive. Think about our booking example. With this approach, we would have to lock the entire `booking` table to prevent new overlapping bookings being added! The only other alternative is to create a separate table to allow us to materialise conflicts. That would be a new table where each row in the database represented a date in the calendar. The new table provides a specific materialised resource to lock during the update, instead of the entire `booking` table. There are drawbacks to this approach. Firstly, it is only suitable for the class of phantoms where the precondition is checking for an empty set. Further, materialised conflicts leak concurrency control into the data model.

Again, I recommend trying to see if the whole issue can be avoided by modelling your user journey better. For example, when making a booking, rarely would a user expect, or even want, the entire record being saved at the very end, after they've made their payment. If they accidentally closed their browser early, all their hard work would be lost. Instead, why not save a pending booking while they continue through the final confirmation steps? That would involve adding a `status` column to the `booking` table, which could take values like `confirmed` and `in_progress` or `pending`. When the user finalises their booking we again check no overlapping bookings exist. Now we can see if another pending booking exists. If that pending booking was created earlier, we return an error to the user like `The selected period is no longer available`. This would be frustrating to the user, sure. But it would occur infrequently, and allows the rest of the application to run without the burden of serialisation.

# Wrap up
A notable point is that your database performance is strongly tied to your modelling. Understand what your users need, the journeys they go through, and design your app, and the database supporting it, accordingly. Many of the thornier phenomena can be avoided entirely by modelling the data more appropriately. We often fall into a trap by assuming the data we are updating data MUST be consistent at all times. That is the default mental model of developers, but is only true in the minority of cases. The inevitable conclusion of consistency is synchronicity, which can slow down and complicate your application.
