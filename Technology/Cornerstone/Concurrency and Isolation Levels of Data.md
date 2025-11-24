Concurrency and Isolation Levels of Database Transactions

Nowadays, by default, data reading and writing rely on databases, which have become a basic infrastructure. Transactions within databases provide a reliable guarantee for the correctness of applications. If we trace back to how developers worked before transactions existed, we can understand the tricky problems that transactions solve for us.
Suppose there were no transactions and we were developing a client application. What problems would we encounter when reading from and writing to the database?
The database or the machine it runs on may crash at any time.
The application may crash at any time.
The network may slow down or be completely disconnected at any time.
When a client reads data, it may only read partially successful data.
Concurrency issues. Multiple clients may read and write the same data simultaneously, leading to race conditions.
If each client had to consider the above issues, the development burden on programmers would be unimaginable. In addition to dealing with business requirements, they would also have to cope with these challenges. Fortunately, these challenges are relatively low-level and have nothing to do with specific businesses. Every application involving data reading and writing will encounter them. Database designers have abstracted the concept of transactions and made a series of guarantees. When operating the database based on transactions, there is no need to worry about hardware failures, software crashes, network problems, and concurrency issues.
However, do all databases support transactions? What guarantees do transactions provide? Are transactions a silver bullet that is required in all scenarios? What costs are involved in implementing transactions? There are so many isolation levels for transactions; which scenarios are they suitable for respectively? These are all issues that need to be considered when using transactions.
The History of Transactions
Early databases did not have the concept of transactions, such as CODASYL’s IDS in 1964 or IBM’s IMS in 1968.
In 1976, Jim Gray of IBM first formally proposed the concept of "Transaction" in the report Notes on Data Base Operating Systems. He defined a set of operations that are either fully executed or not executed at all, and discussed concurrency control and log recovery.
In 1983, German scholars Theo Härder and Andreas Reuter first summarized the four major characteristics of transactions as Atomicity, Consistency, Isolation, and Durability (ACID) in their paper Principles of Transaction-Oriented Database Recovery, laying the foundation for the design of all subsequent databases.
1980s-1990s: Popularization of commercial Relational Database Management Systems (RDBMS). Products such as Oracle (since 1979), IBM DB2 (since 1983), and Microsoft SQL Server (since 1989) were successively launched, and ACID became the standard configuration. During this period, optimizations such as Multi-Version Concurrency Control (MVCC) and more fine-grained locking strategies were also introduced one after another.
MySQL was first developed by Michael "Monty" Widenius and David Axmark in 1994, and its first version (MySQL 1.0) was officially released in May 1995. MySQL AB Company was formally established around 1996, and then gradually developed the database.
Developers are very familiar with the four characteristics of transactions. Here, I will only briefly mention them. Atomicity ensures the ability of a transaction to recover in case of a failure, meaning "either all operations are executed or none are executed". Consistency ensures the consistency of data within the database, such as uniqueness constraints and foreign key constraints. However, this part of consistency is more related to applications, such as maintaining the balance of debits and credits in an accounting system, preventing over-selling of inventory, and ensuring that the user's wallet balance is greater than 0. Isolation ensures the correctness of operations when multiple clients read and write the same data concurrently. Durability ensures that after a transaction is committed, the data is definitely written to disk and will not be lost.
Developers do not need to be involved much in Atomicity and Durability, as the database can handle them well. The most relevant characteristics for developers are Isolation and Consistency. Although Isolation is named as such, it is actually a concurrency issue. It is also the most complex scenario and the most prone to problems, leading to issues such as dirty writes, dirty reads, lost updates, and phantom reads. Data consistency at the application level must be guaranteed by developers for correctness, and its implementation is based on characteristics such as Atomicity, Isolation, and Durability.
Concurrency and Isolation Levels
Dirty Reads
As shown in Figure 7-4 (No dirty reads: user 2 sees the new value for x only after user 1's transaction has committed), the timeline is as follows: User 1 performs "set x=3", "set y=3", and then commits. During this process, User 2 performs multiple "get x" operations. Initially, the database shows x=2 and y=2; after User 1's operations and commit, the database updates x to 3.
The problem is: A dirty read occurs when one transaction reads data that has not been committed by another transaction. This is easy to understand. Dirty reads can cause two problems:
If other transactions involve multiple operations, the current transaction may only read partial data, which may lead to data inconsistency and incorrect programs. For example, in a transaction that displays a user's fund list, if a transfer operation (userA -> userB) is in progress, the read transaction may only see that userA's balance has decreased while userB's balance remains unchanged in the intermediate state.
After the current transaction reads data from other transactions, if those other transactions are rolled back, the previously read data becomes invalid, which is likely to cause problems. Such problems seem abnormal and are difficult to troubleshoot.
Dirty Writes
As shown in Figure 7-5 (With dirty writes, conflicting writes from different transactions can be mixed up), the timeline is as follows: Alice initiates operations: "update listings set buyer = 'Alice' where id=1234" and "update invoices set recipient = 'Alice' where listing_id = 1234". During this process, Bob performs "update listings set buyer = 'Bob' where id=1234" and "update invoices set recipient = 'Bob' where listing_id = 1234", and then commits. Finally, the "listings" table shows buyer = "Bob" and the "invoices" table shows recipient = "Alice".
Take this example: Two users are buying the same car with id 1234. There are two steps: one is to set the buyer of the car, and the other is to set the recipient of the invoice. While Alice is initiating the operation, Bob overwrites the buyer to himself, but the recipient of the invoice is Alice. This leads to a data inconsistency problem and causes a bug.
This phenomenon is called a dirty write, which occurs when another transaction writes to the same piece of data before a transaction is committed. When a transaction has multiple operations, dirty writes are likely to cause data inconsistency and directly lead to incorrect programs. Dirty writes also make program development unpredictable—you clearly write content to the database, but later find that the content has been modified...
Read Committed
The database defines the Read Committed isolation level and guarantees that there will be no dirty reads or dirty writes under this isolation level.
So how is this implemented? The specific implementation method varies among different databases, but for dirty writes, row locks are generally used. A transaction that wants to modify data in a certain row must first acquire the lock for that row and will not release the lock until the transaction exits or is committed. During this period, other transactions that want to read or write to that row will wait until the lock is released.
There are generally two implementation methods for preventing dirty reads: read locks and snapshots. A read lock means adding a temporary share-lock to a record, which ensures that dirty data cannot be read. However, this causes read transactions to be blocked by write transactions. Especially when a write transaction takes a long time, a chain reaction may occur, leading to a serious decline in system throughput.
Read Skew
As shown in Figure 7-6 (Read skew: Alice observes the database in an inconsistent state), the timeline is as follows: Alice performs "select balance from accounts where id=1" and "select balance from accounts where id=2" successively. During this process, a transfer transaction is executed: "update accounts set balance = balance + 100 where id = 1" and "update accounts set balance = balance - 100 where id = 2", and then commits. Initially, Account 1 and Account 2 both have a balance of 500; after the transfer transaction, Account 1's balance becomes 600 and Account 2's balance becomes 400. Finally, Alice reads 500 for Account 1 and 400 for Account 2.
When Alice was checking the balances of her two accounts, a transfer operation happened. Since no locks are added for reads under the Read Committed isolation level, she saw that the balances of Account 1 and Account 2 were 500 and 400 respectively, and the total amount of money decreased.
This is called Read Skew or Non-Repeatable Read. Personally, I think it is better to understand "Skew" as deviation or bias. We may have different views on a person or a thing from different perspectives or in different eras, which means bias occurs. The Read Skew here refers to a deviation in the time dimension. The data Alice saw before the transfer is a deviation in the time dimension; if she checked after the transfer, she would see the latest data.
The Read Skew phenomenon is generally tolerable in the Internet environment. When a user performs a read-only operation, they only need to refresh to see the latest data. However, read transactions in the following two scenarios generally need to prevent Read Skew:
If we have a long-running read transaction for analytical query operations, the queried data may even need to meet certain business constraints. For example, the total balance of Account 1 and Account 2 mentioned above should remain unchanged. In this case, we should not perform the analysis based on data that changes at any time, but rather based on a snapshot.
For snapshot data backup, the data read multiple times needs to be from the same point in time. (Of course, this problem can be bypassed using binlog or mysqldump.)
This reminds us that if a job takes a long time to execute, we need to pay attention to whether the Read Skew phenomenon is tolerable.
Repeatable Read (Snapshot Isolation)
This isolation level adds another guarantee on the basis of Read Committed, which is to prevent the above-mentioned Read Skew. The implementation method of most databases is the well-known MVCC (Multi-Version Concurrency Control). I will not elaborate on the specific mechanism here.
Lost Updates
As shown in Figure 7-1 (A race condition between two clients concurrently incrementing a counter), the timeline is as follows: User 1 performs "get counter" (obtaining 42), calculates 42 + 1 = 43, and then executes "set counter = 43". During this process, User 2 also performs "get counter" (obtaining 42), calculates 42 + 1 = 43, and executes "set counter = 43". The final value of the counter in the database is 43.
Except for dirty writes, the previous concurrency issues all refer to phenomena such as dirty reads and Read Skew that occur when read-only transactions encounter concurrent writes. If the current transaction includes both read and write operations, what other concurrency issues may occur? Lost updates are one such issue. A typical example is the counter in the above figure, where the final result is 43, and one update operation initiated by User 1 is lost. In summary, it means that a later write operation overwrites an earlier write operation.
This type of problem follows a pattern: read -> in-memory modification -> write, and these three operations are connected together.
Atomic Operations
For example: If the above three operations can be converted into a single atomic operation and turned into a single SQL statement for the database to execute, that would be ideal.
Code block:
sql
UPDATE counters SET value = value + 1 WHERE key = 'foo';
Explicit Locks
However, there are many cases where operations cannot be converted into atomic operations. Generally, after reading the data, the in-memory modification logic is very complex and requires calculation, which cannot be done in a single query. For another example, if we need to call a third-party service in the middle, explicit locks can be used.
Code block:
sql
BEGIN TRANSACTION;
-- 1. Lock the relevant record to prevent concurrent modifications
SELECT *FROM figures WHERE name = 'robot' AND game_id = 222 FOR UPDATE;
-- 2. Check whether the move is valid, then update the position of the piece that was returned by the previous SELECT
-- 3. (Business logic for validating the move, e.g., checking if the position is within bounds, movement speed does not exceed the limit, etc.)
UPDATE figures SET position = 'c4' WHERE id = 1234;
-- 4. Commit the transaction to release the lock
COMMIT;
Here, we first need to query the figure and perform some logical validation. When moving the figure, it must comply with certain game-defined rules, such as the movement position not being out of bounds and the movement speed not exceeding the specified limit. Only when these rules are satisfied can we modify the figure's position.
Many databases can automatically detect whether a lost update has occurred on a certain row based on MVCC at the snapshot read level, and automatically abort the current transaction if it occurs. However, MySQL does not do this, but we can achieve the same effect in the application through a version mechanism.
Write Skew & Phantoms
As shown in Figure 7-8 (Example of write skew causing an application bug), the process is as follows:
Alice initiates a transaction: First, execute "select count(*) from doctors where on_call = true and shift_id=1234", and the result shows currently_on_call = 2 (Alice and Bob are on call, Carol is not). Then, since currently_on_call >= 2, execute "update doctors set on_call = false where name = 'Alice' and shift_id=1234", and finally commit the transaction.
Bob initiates a transaction at the same time: First, execute "select count(*) from doctors where on_call = true and shift_id=1234", and the result also shows currently_on_call = 2. Then, since currently_on_call = 2, execute "update doctors set on_call = false where name = 'Bob' and shift_id=1234", and finally commit the transaction.
After both transactions are committed, the on_call status of Alice, Bob, and Carol is all false.
Take this example of doctors on duty: Doctors on duty can take leave, but it must be ensured that there is at least one doctor on duty for each shift. Unfortunately, both Alice and Bob felt unwell at the same time. When they both queried the shift 1234, the database returned that there were two doctors on call. So they both took leave. The final result violated the business constraint.
First, let's analyze this concurrency problem. It is neither a dirty write nor a lost update, because the two transactions operate on different data. Database researchers define this phenomenon as Write Skew. I think this definition is derived from Read Skew. Strictly speaking, Read Skew is for read-only transactions, where the same query returns results from two different times. If a write operation is added to the transaction, the write operation based on the previous bias (stale query results) will lead to Write Skew, thereby violating data consistency constraints and causing bugs.
The "previous bias (stale query results)" is also called a Phantom, which is what we often refer to as a phantom read. It exists in both Read Skew and Write Skew. Therefore, if we directly say that a phantom read is a concurrency problem, I think it is inaccurate, or at least incomplete.
A read-only transaction only affects the current transaction, and the impact is relatively small. It can be tolerated or resolved through a snapshot mechanism. However, a write transaction affects the consistency of the entire database. Therefore, phenomena like Write Skew must be eliminated, and transactions should not be based on stale data for updates.
Lost updates can be regarded as a special case of Write Skew:
Write Skew: Two transactions read the same set of objects, make some modifications, and write to different objects.
Lost updates: Two transactions read the same set of objects, make some modifications, and write to the same object.
Write Skew seems to be an abstract concept. It is abstract because it describes a dynamic event—a write operation after a phantom read occurs. It seems far away from us, but as long as we are aware of it, we will find it exists in all aspects of development. Let's look at a few more examples:
Conference room booking scenario: Two bookings cannot be for the same conference room at the same time.
Code block:
sql
BEGIN TRANSACTION;
-- 1. Check for any existing bookings that overlap with the period of noon-1pm
SELECT COUNT(*) FROM bookings WHERE room_id = 123 AND
end_time > '2015-01-01 12:00' AND start_time < '2015-01-01 13:00';
-- 2. If the previous query returned zero (no overlapping bookings)
INSERT INTO bookings
(room_id, start_time, end_time, user_id)
VALUES (123, '2015-01-01 12:00', '2015-01-01 13:00', 666);
-- 3. Commit the transaction
COMMIT;
Website usernames cannot be duplicated. When inserting a new user, it is necessary to verify whether the username already exists.
In a game, two players cannot move to the same position. When moving a player, it is necessary to verify whether the target position is occupied.
All these examples may lead to Write Skew caused by phantom reads in concurrent situations. They all follow a pattern: the client reads data that meets certain conditions from the database, performs calculations based on this data, and then writes data to the database.
Views
The process between the client and the database is divided into two phases:
Phase 1: Data reading. The client executes "SELECT ... WHERE ...", and the database returns data that meets the conditions.
Since the operations involve different data, atomic operations cannot solve the Write Skew problem.
Similarly, automatic version conflict detection or some optimistic locking mechanisms based on CAS (Compare-and-Swap) are all for specific rows and cannot solve this problem either.
Explicit locks can only be applied to rows that already exist. They can solve the above problem of doctors on duty because the doctor shifts already exist. By using the following SQL to lock all rows with shift_id = 1234, other transactions that want to modify shift 1234 are blocked, ensuring that this part of the logic is executed serially.
Code block:
sql
BEGIN TRANSACTION;
-- 1. Lock all doctors on duty for shift 1234
SELECT *FROM doctors
WHERE on_call = true
AND shift_id = 1234 FOR UPDATE;
-- 2. Update Alice's on-call status to false
UPDATE doctors
SET on_call = false WHERE name = 'Alice' AND shift_id = 1234;
-- 3. Commit the transaction to release the lock
COMMIT;
Explicit locks are ineffective for the other examples mentioned earlier. For conference room booking, user creation, and player movement, the query conditions in the first step may return no data or only partial data. However, when specific writes are performed in the third step, the results returned by the query conditions change (data exists), which makes it impossible for us to lock in advance.
Serializable
The Serializable isolation level provides a guarantee: even if transactions are executed concurrently, the execution result is the same as if the transactions were executed serially.
Actual Serial Execution (e.g., Redis)
This approach avoids all problems caused by concurrent execution. However, there are certain constraints—it can only execute small and short transactions. Otherwise, a long transaction will affect the performance of the entire system. Since execution is single-threaded, all write requests depend on a single CPU, resulting in limited throughput.
Two-Phase Locking (2PL)
In simple terms, a read lock is added during reading, and a write lock is added during writing. Reads do not block other reads, but reads block writes and writes block reads/writes. Once a transaction acquires a lock, it holds the lock until the transaction is committed or aborted. Two-Phase Locking gets its name from this process, which is divided into two phases: locking and unlocking.
Performance of Two-Phase Locking
In practice, the performance of Two-Phase Locking is poor. There is an overhead for locking and unlocking operations, but the more important reason is the reduction in concurrency. Under the snapshot read mechanism, read transactions do not block write transactions, and write transactions do not block read transactions. However, with Two-Phase Locking, read transactions block write transactions, and write transactions block read transactions. The database does not limit the execution time of transactions, so a long transaction can bring down the entire system. Under Two-Phase Locking, the probability of deadlocks is higher, and the number of retries is also greater.
Predicate Locks
As mentioned earlier, the concurrency problem of conference room booking cannot be solved by existing solutions, but it can be solved by 2PL. Let's see how this works:
Code block:
sql
-- 1. Query for overlapping bookings for room 123 between 12:00 and 13:00 on 2018-01-01
SELECT* FROM bookings WHERE room_id = 123 AND
end_time > '2018-01-01 12:00' AND start_time < '2018-01-01 13:00';
The database uses predicate locks to solve this problem.
When Transaction A queries based on a certain condition, the database attempts to add a shared predicate lock based on that query condition. If another Transaction B performs a write operation at the same time, and the record written by B meets the query condition of A, A will wait until B is completed.
After Transaction A adds the predicate lock, if another Transaction C attempts to perform a write operation, it needs to wait until A is completed.
The core idea is that predicate locks also apply to records that do not exist in the database, which can solve the Write Skew problem and other concurrency issues.
Range Locks (Index-Range Locks)
Predicate locks are very time-consuming because they require checking the specified query conditions for all ongoing read and write transactions. Gap locks are a simplification of predicate locks. In the example of conference room booking, you can create an index for room_id. Then, when adding a read lock, a read lock is added to the index record of room_id = 123. You can also add a lock for time—when adding a read lock, a gap lock is added for a specific time range.
After simplification, coarse-grained locks replace predicate locks, but they can still prevent Write Skew, and query performance is significantly improved.
Summary

1. Isolation Levels Defined by ANSI SQL
The isolation levels discussed above are mainly those defined by ANSI SQL:
Read Uncommitted: The database does nothing (no isolation guarantees).
Read Committed: Prevents dirty reads and dirty writes.
Repeatable Read: Prevents Read Skew but does not guarantee prevention of Write Skew.
Serializable: Prevents Write Skew (using strict Two-Phase Locking; all reads add index-range locks).
The specific implementation of isolation levels varies among different database vendors, especially for Repeatable Read. Developers need to understand the details when using a specific database. However, only by understanding the concurrency issues that may occur and combining them with the guarantees provided by the database vendor for isolation levels can developers make better decisions on which isolation level to use.
Concurrency issues are inevitable when the database reads and writes concurrently. Isolation levels are the effects and goals that the database aims to achieve. Only Serializable can truly achieve serialization and eliminate all concurrency issues. The other isolation levels are weak isolation levels, which are trade-offs between performance and data consistency. Except for actual physical serialization, other implementations prevent concurrency issues through a series of algorithms, and all transactions read and write the same set of data.
2. Isolation Levels in MySQL InnoDB
The isolation levels in MySQL InnoDB are basically consistent with those defined by SQL, but there are also many differences. The more important ones are:
Repeatable Read in InnoDB is implemented through a combination of MVCC and Two-Phase Locking. Its requirements are stricter than those defined by ANSI SQL, and it can prevent phantom reads in some cases.
For MVCC snapshot reads (plain SELECT), InnoDB does not add any locks. Therefore, changes in the results of query conditions can lead to Write Skew in write operations.
Only when you use locking reads (SELECT … FOR UPDATE / LOCK IN SHARE MODE) or perform UPDATE/DELETE operations will InnoDB add next-key locks (record locks + gap locks) on the index to "lock" the entire scanned range, preventing others from inserting rows and thus avoiding phantom reads.
3. Distributed Locks for Preventing Concurrency
If the database isolation level used is Read Committed, in practice, we often use business identifiers such as merchant IDs or order numbers as keys to lock specific operations during write operations.
Advantages
Flexibility: It does not depend on a specific database engine.
Cross-domain Locking: It can perform unified locking across databases, tables, and even services.
Costs
Developers must identify all "risk points" (places where phantom reads/Write Skew may occur) and add locks manually. Any omission may lead to consistency issues.
Additional handling is required for issues such as the stability of distributed locks, timeouts, retries, and performance overhead.
I think the main problem lies in the first point mentioned above—it requires developers to pre-judge the points that may lead to phantom reads. As the system becomes more complex, this becomes more difficult and makes the system difficult to maintain. Let's take the example of VIP benefit cards: distributed locks are added based on merchant IDs for operations such as using card quotas, refunding card quotas, and transferring benefits. However, no lock is added during card purchase. Although this is acceptable in business (new cards are not used temporarily), the card purchase transaction may cause Write Skew in the card usage transaction.
