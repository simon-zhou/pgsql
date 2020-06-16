## High Availability

By leveraging PostgreSQL’s streaming replication, we can easily create a master and some replicas with asynchronous or synchronous replication. The next step is to automate the failover when the master instance is unhealthy. There are various good tools to achieve this (for example **repmgr** or **governor**).

Stolon is a different approach to set up PG cluster. See introduction [here](https://sgotti.dev/post/stolon-introduction/) and repo [here](https://github.com/sorintlab/stolon)

2ndQuadrant provides [BDR](https://www.2ndquadrant.com/en/resources/postgres-bdr-2ndquadrant/) (Bidirectional Replication for PostgreSQL) which is built on top of logical decoding.

[PostgreSQL XL](https://www.postgres-xl.org/)

[PostgreSQL XC](https://postgresxc.fandom.com/wiki/Postgres-XC_Wiki)

PostgreSQL Pro

Citus DB

[Next-Generation PostgreSQL Replication via Bi-Directional Multi-Master Pub/Sub](https://www.youtube.com/watch?v=PEEsNOrD6BM)

[The Future of PostgreSQL Multi-Master Replication](https://www.youtube.com/watch?v=4klaPUjbMZo)

## Comparison of Different Solutions [source](https://www.postgresql.org/docs/12/different-replication-solutions.html)
**Shared Disk Failover**

Shared disk failover avoids synchronization overhead by having only one copy of the database. It uses a single disk array that is shared by multiple servers. If the main database server fails, the standby server is able to mount and start the database as though it were recovering from a database crash. This allows rapid failover with no data loss.

Shared hardware functionality is common in network storage devices. Using a network file system is also possible, though care must be taken that the file system has full POSIX behavior (see Section 18.2.2.1). One significant limitation of this method is that if the shared disk array fails or becomes corrupt, the primary and standby servers are both nonfunctional. Another issue is that the standby server should never access the shared storage while the primary server is running.

**File System (Block Device) Replication**

A modified version of shared hardware functionality is file system replication, where all changes to a file system are mirrored to a file system residing on another computer. The only restriction is that the mirroring must be done in a way that ensures the standby server has a consistent copy of the file system — specifically, writes to the standby must be done in the same order as those on the master. DRBD is a popular file system replication solution for Linux.

**Write-Ahead Log Shipping**
Warm and hot standby servers can be kept current by reading a stream of write-ahead log (WAL) records. If the main server fails, the standby contains almost all of the data of the main server, and can be quickly made the new master database server. This can be synchronous or asynchronous and can only be done for the entire database server.

A standby server can be implemented using file-based log shipping (Section 26.2) or streaming replication (see Section 26.2.5), or a combination of both. For information on hot standby, see Section 26.5.

**Logical Replication**

Logical replication allows a database server to send a stream of data modifications to another server. PostgreSQL logical replication constructs a stream of logical data modifications from the WAL. Logical replication allows the data changes from individual tables to be replicated. Logical replication doesn't require a particular server to be designated as a master or a replica but allows data to flow in multiple directions. For more information on logical replication, see Chapter 30. Through the logical decoding interface (Chapter 48), third-party extensions can also provide similar functionality.

**Trigger-Based Master-Standby Replication**

A master-standby replication setup sends all data modification queries to the master server. The master server asynchronously sends data changes to the standby server. The standby can answer read-only queries while the master server is running. The standby server is ideal for data warehouse queries.

Slony-I is an example of this type of replication, with per-table granularity, and support for multiple standby servers. Because it updates the standby server asynchronously (in batches), there is possible data loss during fail over.

**Statement-Based Replication Middleware**

With statement-based replication middleware, a program intercepts every SQL query and sends it to one or all servers. Each server operates independently. Read-write queries must be sent to all servers, so that every server receives any changes. But read-only queries can be sent to just one server, allowing the read workload to be distributed among them.

If queries are simply broadcast unmodified, functions like random(), CURRENT_TIMESTAMP, and sequences can have different values on different servers. This is because each server operates independently, and because SQL queries are broadcast (and not actual modified rows). If this is unacceptable, either the middleware or the application must query such values from a single server and then use those values in write queries. Another option is to use this replication option with a traditional master-standby setup, i.e. data modification queries are sent only to the master and are propagated to the standby servers via master-standby replication, not by the replication middleware. Care must also be taken that all transactions either commit or abort on all servers, perhaps using two-phase commit (PREPARE TRANSACTION and COMMIT PREPARED). Pgpool-II and Continuent Tungsten are examples of this type of replication.

**Asynchronous Multimaster Replication**

For servers that are not regularly connected or have slow communication links, like laptops or remote servers, keeping data consistent among servers is a challenge. Using asynchronous multimaster replication, each server works independently, and periodically communicates with the other servers to identify conflicting transactions. The conflicts can be resolved by users or conflict resolution rules. Bucardo is an example of this type of replication.

**Synchronous Multimaster Replication**

In synchronous multimaster replication, each server can accept write requests, and modified data is transmitted from the original server to every other server before each transaction commits. Heavy write activity can cause excessive locking and commit delays, leading to poor performance. Read requests can be sent to any server. Some implementations use shared disk to reduce the communication overhead. Synchronous multimaster replication is best for mostly read workloads, though its big advantage is that any server can accept write requests — there is no need to partition workloads between master and standby servers, and because the data changes are sent from one server to another, there is no problem with non-deterministic functions like random().

PostgreSQL does not offer this type of replication, though PostgreSQL two-phase commit (PREPARE TRANSACTION and COMMIT PREPARED) can be used to implement this in application code or middleware.

**Commercial Solutions**

Because PostgreSQL is open source and easily extended, a number of companies have taken PostgreSQL and created commercial closed-source solutions with unique failover, replication, and load balancing capabilities.
