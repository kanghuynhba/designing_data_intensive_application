# Chapter 7 
* What is the **Transaction**?
    * Transaction processing just means allowing 
    clients to make low-latency reads and writes
    * Mostly, in **Distributed System**, failures are inevitable.
    * Transaction were created as a deliberated design for 
    the purpose of simplify the programming model.
        *  Logical Unit: A transaction groups multiple reads and writes into a single logical unit.
        *  All-or-Nothing: The entire unit either succeeds (commit) or fails (abort/rollback).
        *  Safety Guarantee: By using transactions, the application application ignores partial failures. 
        If a crash occurs halfway through, the database ensures the partial data is discarded, allowing the application to retry safely.
## The meaning of ACID
### Atomicity
* **Definition**: In ACID, this refers to **abortability**, not concurrency. It ensures that if a client make a transaction
                    and a fault occurs (process crash, disk full, etc) the transaction is aborted and all writes are discarded.
* **Benefit**: It eliminates the problem of partial updates, ensuring the database is not left in an inconsistent state.
### Consistency
* **Definition**: This refers to the application's notion of valid states (invariants), such as "debits must equal credits"".
* **The Distinction**: Unlike the other three properties, **Consistency is the responsibility of the application**, not the database. 
                   The database stores data, but the application defines the rules that make that data valid. 
### Isolation
* **Definition**: Isolation handles concurrency. It ensures that concurrently executing transactions do not interfere with one another (avoiding race conditions).
* **Ideal State**: The highest level is **Serializability**, where the database ensures that the result of concurrent transactions is the same as if they had run one after the other.
* **Reality**: Because serializability hurts performance, many databases use weaker isolation levels (like **Snapshot Isolation**) by default.
### Durability
* **Definition**: The promise that once a transaction commits, the data is saved and will not be lost, even in the event of a crash.
* **Mechanisms**: In single-node systems, this involves writing to non-volatile storage (Disk/SSD) and write-ahead logs (WAL).
                    In distributed systems, it implies replication to multiple nodes.

## Key Takeaway:
* Transactions exist to abstract away the complexity of partial failures and concurrency issues from developers.
* While it comes with the performance cost, it provides essential safety guarantees that prevent data corruption. 
