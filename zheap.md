Zheap is a new storage engine format with a few goals:
- better bloat control, by performing in-place update when possible and reusing space right after commit or abort.
- Fewer writes
- smaller in size

References:
[Zheap: The Next Generation Storage Engine for Postgres](https://www.percona.com/live/19/sites/default/files/slides/Zheap-%20The%20Next%20Generation%20Storage%20Engine%20for%20Postgres.pdf)
[Zheap on Github](https://github.com/EnterpriseDB/zheap)
