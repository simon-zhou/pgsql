**TL;DR:** Logical replication sends row-by-row changes, physical replication sends disk block changes. Logical replication is better for some tasks, physical replication for others.

As of 9.5 logical replication is fairly immature. Use physical replication if you are asking this question.

**Streaming replication can be logical replication.**

## WAL-shipping vs. streaming

There are two main ways to send data from master to replica:
- WAL-shipping or continuous archiving, where individual write-ahead-log files are copied from pg_xlog by the archive_command running on the master to some other locations. A restore_command configured in the replica's recovery.conf runs on the replica to fetch the archives so the replica can replay the WAL.

  This is what's used for PITR (point-in-time replication), which is used as a method of continuous backup.
  
  No direct network connection is required to the master server. Replication can have long delays, especially without an archive_timeout set. WAL shipping cannot be used for synchronous replication.
  
- streaming replication, where each change is sent to one or more replica servers directly over a TCP/IP connection as it happens. The replicas must have a direct network connection the master configured in their recovery.conf's primary_conninfo option.

  Streaming replication has little or no delay so long as the replica is fast enough to keep up. It can be used for synchronous replication. You cannot use streaming replication for PITR so it's not much use for continuous backup. If you drop a table on the master, oops, it's dropped on the replicas too.
  
Thus, the two methods have different purposes.

## Asynchronous vs synchronous streaming

On top of that, there's synchronous and asynchronous streaming replication:

- In asynchronous streaming replication the replica(s) are allowed to fall behind the master in time when the master is faster/busier. If the master crashes you might lose data that wasn't replicated yet.

  If the asynchronous replica falls too far behind the master, the master might throw away information the replica needs if wal_keep_segments is too low and no slot is used, meaning you have to re-create the replica from scratch. Or the master's pg_xlog might fill up and stop the master from working until disk space is freed if wal_keep_segments is too high or a slot is used.

- In synchronous replication the master doesn't finish committing until a replica has confirmed it received the transaction2. You never lose data if the master crashes and you have to fail over to a replica. The master will never throw away data the replica needs or fill up its xlog and run out of disk space because of replica delays. In exchange it can cause the master to slow down or even stop working if replicas have problems, and it always has some performance impact on the master due to network latency.

  When there are multiple replicas, only one is synchronous at a time. See synchronous_standby_names.

You can't have synchronous log shipping.

You can actually combine log shipping and asynchronous replication to protect against having to recreate a replica if it falls too far behind, without risking affecting the master. This is an ideal configuration for many deployments, combined with monitoring how far the replica is behind the master to ensure it's within acceptable disaster recovery limits.

## Logical vs physical

On top of that we have logical vs physical streaming replication, as introduced in PostgreSQL 9.4:

- In physical streaming replication changes are sent at nearly disk block level, like "at offset 14 of disk page 18 of relation 12311, wrote tuple with hex value 0x2342beef1222....".

  Physical replication sends everything: the contents of every database in the PostgreSQL install, all tables in every database. It sends index entries, it sends the whole new table data when you VACUUM FULL, it sends data for transactions that rolled back, etc. So it generates a lot of "noise" and sends a lot of excess data. It also requires the replica to be completely identical, so you cannot do anything that'd require a transaction, like creating temp or unlogged tables. Querying the replica delays replication, so long queries on the replica need to be cancelled.

  In exchange, it's simple and efficient to apply the changes on the replica, and the replica is reliably exactly the same as the master. DDL is replicated transparently, just like everything else, so it requires no special handling. It can also stream big transactions as they happen, so there is little delay between commit on the master and commit on the replica even for big changes.

  Physical replication is mature, well tested, and widely adopted.

- logical streaming replication, new in 9.4, sends changes at a higher level, and much more selectively.

  It replicates only one database at a time. It sends only row changes and only for committed transactions, and it doesn't have to send vacuum data, index changes, etc. It can selectively send data only for some tables within a database. This makes logical replication much more bandwidth-efficient.

  Operating at a higher level also means that you can do transactions on the replica databases. You can create temporary and unlogged tables. Even normal tables, if you want. You can use foreign data wrappers, views, create functions, whatever you like. There's no need to cancel queries if they run too long either.

  Logical replication can also be used to build multi-master replication in PostgreSQL, which is not possible using physical replication.

  In exchange, though, it can't (currently) stream big transactions as they happen. It has to wait until they commit. So there can be a long delay between a big transaction committing on the master and being applied to the replica.

  It replays transactions strictly in commit order, so small fast transactions can get stuck behind a big transaction and be delayed quite a while.

  DDL isn't handled automatically. You have to keep the table definitions in sync between master and replica yourself, or the application using logical replication has to have its own facilities to do this. It can be complicated to get this right.

  The apply process its self is more complicated than "write some bytes where I'm told to" as well. It also takes more resources on the replica than physical replication does.

  Current logical replication implementations are not mature or widely adopted, or particularly easy to use. There are also some [limitations](https://www.postgresql.org/docs/10/logical-replication-restrictions.html).

## Too many options, tell me what to do

Phew. Complicated, huh? And I haven't even got into the details of delayed replication, slots, wal_keep_segments, timelines, how promotion works, Postgres-XL, BDR and multimaster, etc.

So what should you do?

There's no single right answer. Otherwise PostgreSQL would only support that one way. But there are a few common use cases:

For backup and disaster recovery use pgbarman to make base backups and retain WAL for you, providing easy to manage continuous backup. You should still take periodic pg_dump backups as extra insurance.

For high availability with zero data loss risk use streaming synchronous replication.

For high availability with low data loss risk and better performance you should use asynchronous streaming replication. Either have WAL archiving enabled for fallback or use a replication slot. Monitor how far the replica is behind the master using external tools like Icinga.

## References
[continuous archiving and PITR](http://www.postgresql.org/docs/current/static/continuous-archiving.html)

[high availability, load balancing and replication](http://www.postgresql.org/docs/current/static/high-availability.html)

[replication settings](http://www.postgresql.org/docs/current/static/runtime-config-replication.html)

[pgbarman](http://www.pgbarman.org/)

[repmgr](http://www.repmgr.org/)

[wiki: replication, clustering and connection pooling](https://wiki.postgresql.org/wiki/Replication,_Clustering,_and_Connection_Pooling)

**[source](https://stackoverflow.com/questions/33621906/difference-between-stream-replication-and-logical-replication)**
