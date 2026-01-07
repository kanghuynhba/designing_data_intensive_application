# Chapter 5 Replication
* Replication means you want to keep a copy of the same data on multiple machines that are connected via a network.
* There are several reasons determines why you should replicate data:
    * **High Availability**: The system keeps working even if a machine (or datacenter) fails.
    * **Latency**: Data is kept geographically close to user (e.g US users access a US Replica).
    * **Scalability**: You can increase read throughput by adding more machines to server read query.
## Leaders and Followers
* How do we ensure that all the data ends up on the replicas?
    * &rarr; the most common solution called leader-based replication.
        1. **One of the replicas is the leader.** When clients want to write to the database,
        they must send their requests to the leader.
        2. **The other replicas is called followers.** The leader sends the data change to all of its followers.
        3. When a client wants to read from the database, they can read from the leader or follower. 
        **However writes are only accepted on the leader (the follower are read-only from the client's pov).**
### Synchronous vs Asynchronous Replication
* The replication happens synchronous or asynchronous. 
    * The leader waits until follower 1 has confirmed that it received the write
    before reporting success to the user, and before making the write visible to other clients.
        * &rarr; This means synchronous.
    * The leader sends the message, but doesn't wait for a response from the follower.
        * &rarr; This means asynchronous.

