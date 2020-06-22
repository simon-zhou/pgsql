Zheap is a new storage engine format with a few goals:
- better bloat control, by performing in-place update when possible and reusing space right after commit or abort.
- Fewer writes
- smaller in size

## References
[Zheap: The Next Generation Storage Engine for Postgres](https://www.percona.com/live/19/sites/default/files/slides/Zheap-%20The%20Next%20Generation%20Storage%20Engine%20for%20Postgres.pdf)

[Zheap on Github](https://github.com/EnterpriseDB/zheap)

[Future of Storage](https://wiki.postgresql.org/wiki/Future_of_storage)

[DO or UNDO - there is no VACUUM](http://rhaas.blogspot.com/2018/01/do-or-undo-there-is-no-vacuum.html)

[Proposal](https://www.postgresql.org/message-id/flat/CAA4eK1%2BYtM5vxzSM2NZm%2BpC37MCwyvtkmJrO_yRBQeZDp9Wa2w%40mail.gmail.com)
