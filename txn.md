## Isolation Levels

## Repeatable Read and Naming Confusion (Designing Data Intensive Applications)

Snapshot isolation is a useful isolation level, especially for read-only transactions. However, many databases that implement it call it by different names. In Oracle it is called serializable, and in PostgreSQL and MySQL it is called repetable read.

The reason for this naming confusion is that the SQL standard doesn't have the concept of snapshot isolation, because the standard is based on System R's 1975 definition of isolation levels and snapshot isolation hadn't yet been invented then. Instead, it defines repeatable read, which looks superficially similar to snapshot isolation. PostgreSQL and MySQL call their snapshot isolation level repeatable read because it meets the requirements of the standard, and so they can claim standards compliance.

Unfortunately, the SQL standard's definition of isolation levels is flawed - it is ambiguous, imprecise, and not as implementation-independent as a standard should be. Even though several databases implement repeatable read, there are big differences in the guarantees they actually provide, despite being ostensibly standardized. There has been a formal definition of repeatable read in the research literature, but most implementations don't satisfy that formal definition. And to top it off, IBM DB2 uses "repeatable read" to refer to serializability.

Different databases use different techniques to implement serializable isolation. PG uses MVCC and serialization graph cycle detection while other databases (InnoDB, SQL Server) use MVCC and lock-based serializable. See details [here](https://www.slideshare.net/MarkusWinand/sql-transactions-what-they-are-good-for-and-how-they-work).

## References:

[Consistency Models](https://jepsen.io/consistency), from [jepsen](https://jepsen.io/)
