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
Q: How does index-only scan (by leveraging covering index) works while the index doesn't store row visibility info? How are visibility map and vacuuming coming into play?

A: Visibility info is stored on heap. When doing index-only scan, it checks heap page visibility info in visibility map, which is much smaller than heap thus smaller IO cost. Each page has two bits in the visibility map, the first one is the page visibility info and the 2nd one is about whether all tuples on the page have been frozen (anti-wraparound vacuum doesn't need to visit the page). Visibility map bits are only set by vacuum, but are cleared by any data-modifying operations on a page ([source](https://www.postgresql.org/docs/current/storage-vm.html)). More about index-only scan [here](https://www.postgresql.org/docs/10/indexes-index-only-scans.html).

Q: What is INCLUDE index then?

A: Before INCLUDE index and if you want to query non-key columns, you have to check the heap. Or maybe the workaround is to have the non-key columns part of multi-column index. This is OK but with some limitations, eg, you cannot enforce uniqueness on the keys and the non-key columns can only be trailing columns (see [here](https://www.postgresql.org/docs/12/indexes-multicolumn.html)). All these limitations are gone with INCLUDE index. Details can be found [here](https://www.postgresql.org/docs/12/indexes-index-only-scans.html).

The INCLUDE columns exist solely to allow more queries to benefit from index-only scans.  Also, such columns don't need to have appropriate operator classes.  Expressions are not supported as INCLUDE columns since they cannot be used in index-only scans. In B-tree indexes INCLUDE columns are truncated from pivot index tuples (tuples located in non-leaf pages and high keys).  Therefore, B-tree indexes now might have variable number of attributes.  This patch also provides generic facility to support that: pivot tuples contain number of their attributes in t_tid.ip_posid. See details from commit 8224de4f42ccf98e08db07b43d52fed72f962ebb. Proposal is [here](https://www.postgresql.org/message-id/flat/56168952.4010101@postgrespro.ru).

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/simon-zhou/pgsql.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
