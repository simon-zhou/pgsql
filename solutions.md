## High Availability

By leveraging PostgreSQL’s streaming replication, we can easily create a master and some replicas with asynchronous or synchronous replication. The next step is to automate the failover when the master instance is unhealthy. There are various good tools to achieve this (for example **repmgr** or **governor**).

Stolon is a different approach to set up PG cluster. See introduction [here](https://sgotti.dev/post/stolon-introduction/) and repo [here](https://github.com/sorintlab/stolon)

2ndQuadrant provides [BDR](https://wiki.postgresql.org/wiki/Multimaster) (Bidirectional Replication for PostgreSQL) which is built on top of logical decoding.

[Next-Generation PostgreSQL Replication via Bi-Directional Multi-Master Pub/Sub](https://www.youtube.com/watch?v=PEEsNOrD6BM)
[The Future of PostgreSQL Multi-Master Replication](https://www.youtube.com/watch?v=4klaPUjbMZo)
