## Welcome to PGSQL

PG uses a variant of L&Y algorithm for concurrent access to B-tree, which is also referred as BLink-tree. Specifically, L&Y requires a B-tree to have a **high-key** which is an upper bound on the keys on that page and a **next** link to the right sibling. These two additions make it possible detect a concurrent page split, which allows the tree to be searched without holding any read locks.

When a search follows a downlink to a child page, it compares the page's high key with the search key. If the search key is greater than the high key, the page must have been split concurrently, and you must follow the right-link to find the new page containing the key range you're looking for. This might need to be repeated, if the page has been split more than once.

In addition to that, PG also have a **prev** link to the left sibling for reverse scan. There are a few optimizations in PG to make B-tree more efficient:
 - HOT (Heap Only Tuple)
 - Index-only Scan (Covering Index)
 - Suffix Truncation

## B-tree Implementation

[btree_source.md](btree_source.md)

## Open Questions

### Q: How does index-only scan (by leveraging covering index) works while the index doesn't store row visibility info? How are visibility map and vacuuming coming into play?

A: Visibility info is stored on heap. When doing index-only scan, it checks heap page visibility info in visibility map, which is much smaller than heap thus smaller IO cost. Each page has two bits in the visibility map, the first one is the page visibility info and the 2nd one is about whether all tuples on the page have been frozen (anti-wraparound vacuum doesn't need to visit the page). Visibility map bits are only set by vacuum, but are cleared by any data-modifying operations on a page ([source](https://www.postgresql.org/docs/current/storage-vm.html)). More about index-only scan [here](https://www.postgresql.org/docs/10/indexes-index-only-scans.html).

### Q: What is INCLUDE index then?

A: Before INCLUDE index and if you want to query non-key columns, you have to check the heap. Or maybe the workaround is to have the non-key columns part of multi-column index. This is OK but with some limitations, eg, you cannot enforce uniqueness on the keys and the non-key columns can only be trailing columns (see [here](https://www.postgresql.org/docs/12/indexes-multicolumn.html)). All these limitations are gone with INCLUDE index. Details can be found [here](https://www.postgresql.org/docs/12/indexes-index-only-scans.html).

The INCLUDE columns exist solely to allow more queries to benefit from index-only scans.  Also, such columns don't need to have appropriate operator classes.  Expressions are not supported as INCLUDE columns since they cannot be used in index-only scans. In B-tree indexes INCLUDE columns are truncated from pivot index tuples (tuples located in non-leaf pages and high keys).  Therefore, B-tree indexes now might have variable number of attributes.  This patch also provides generic facility to support that: pivot tuples contain number of their attributes in t_tid.ip_posid. See details from commit 8224de4f42ccf98e08db07b43d52fed72f962ebb. Proposal is [here](https://www.postgresql.org/message-id/flat/56168952.4010101@postgrespro.ru).

### Q: What does vacuuming do?

A: Vacuuming does three things: to remove dead tuples created for MVCC (defragmentation), to update data statistics for planner and to avoid transaction ID wraparound, which involves freeze tuples old enough. This is best explained [here](https://www.postgresql.org/docs/13/routine-vacuuming.html).

### Q: Why HOT (Heap Only Tuple) and how does it work?

A: Without HOT, a new index tuple needs to be inserted every time an update happens, due to MVCC. The old index tuple points to the old heap tuple and the new index tuple points to the new heap tuple. With HOT, only new heap tuple is inserted and index tuple doesn't need to be inserted. This way we can save some IO. It works when the new heap tuple is on the same page as the old heap tuple. The old heap tuple will have "pointer" pointing to the new tuple. This is explained [here](http://www.interdb.jp/pg/pgsql07.html).

### Q: How does TOAST (The Oversized Attribute Storage Technique) work?

A: PostgreSQL uses a fixed page size (commonly 8 kB), and does not allow tuples to span multiple pages. Therefore, it is not possible to store very large field values directly. When a row is attempted to be stored that exceeds this size, TOAST basically breaks up the data of large columns into smaller "pieces" and stores them into a TOAST table. Each table you create has its own associated (unique) TOAST table, which may or may not ever end up being used, depending on the size of rows you insert. All of this is transparent to the user and enabled by default. The mechanism is accomplished by splitting up the large column entry into 2KB bytes and storing them as chunks in the TOAST tables. It then stores the length and a pointer to the TOAST entry back where the column is normally stored. Because of how the pointer system is implemented, most TOAST'able column types are limited to a max size of 1GB.

## References
[Indexes in PostgreSQL](https://postgrespro.com/blog/pgsql/3994098)


For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).
