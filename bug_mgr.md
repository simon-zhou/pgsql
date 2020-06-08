# Fun Facts

## Buffer manager uses clock sweep (sometimes refers as 2nd chance) algorithm for page eviction, instead of LRU. Clock sweep doesn't involve coordination/synchronization.

## Checkpointer and bgwriter are two different processes. They were one but were separated in 9.2.

## In certain cases, PG uses ring buffer instead of buffer pool, eg, when bulk reading large files, using copy from command, creating MV and altering table.
