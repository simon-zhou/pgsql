With a leaf page and its offset, we can get the **IndexTuple** this way:

```markdown
// bufpage.h
#define PageGetItemId(page, offsetNumber)
	((ItemId) (&((PageHeader) (page))->pd_linp[(offsetNumber) - 1]))

// bufpage.h
#define PageGetItem(page, itemId)
(
	AssertMacro(PageIsValid(page)),
	AssertMacro(ItemIdHasStorage(itemId)),
	(Item)(((char *)(page)) + ItemIdGetOffset(itemId))
)

// nbtsearch.h _bt_compare
itup = (IndexTuple) PageGetItem(page, PageGetItemId(page, offnum));
```

Generally a page has line pointers of type **ItemIdData**, which has two bits flags, 15 bits offset within that page and 15 bits byte length of the tuple. By using the offset we can get to **IndexTuple**.

Definition of **ItemIdData** and **IndexTuple**:

```
typedef struct ItemIdData
{
	unsigned	lp_off:15,		/* offset to tuple (from start of page) */
			lp_flags:2,		/* state of line pointer, see below */
			lp_len:15;		/* byte length of tuple */
} ItemIdData;

typedef ItemIdData *ItemId;

typedef struct IndexTupleData
{
	ItemPointerData t_tid;		/* reference TID to heap tuple */

	/* ---------------
	 * t_info is laid out in the following fashion:
	 *
	 * 15th (high) bit: has nulls
	 * 14th bit: has var-width attributes
	 * 13th bit: AM-defined meaning
	 * 12-0 bit: size of tuple
	 * ---------------
	 */

	unsigned short t_info;	/* various info about tuple */

} IndexTupleData;		/* MORE DATA FOLLOWS AT END OF STRUCT */

typedef IndexTupleData *IndexTuple;
```

Finally, this is the structure of a disk page from bufpage.h:
```
 * +----------------+---------------------------------+
 * | PageHeaderData | linp1 linp2 linp3 ...           |
 * +-----------+----+---------------------------------+
 * | ... linpN |                                      |
 * +-----------+--------------------------------------+
 * |		   ^ pd_lower                         |
 * |                                                  |
 * |			 v pd_upper                   |
 * +-------------+------------------------------------+
 * |			 | tupleN ...                 |
 * +-------------+------------------+-----------------+
 * |	   ... tuple3 tuple2 tuple1 | "special space" |
 * +--------------------------------+-----------------+
 *                                  ^ pd_special


typedef struct PageHeaderData
{
	/* XXX LSN is member of *any* block, not only page-organized ones */
	PageXLogRecPtr pd_lsn;		/* LSN: next byte after last byte of xlog
					 * record for last change to this page */
	uint16		pd_checksum;	/* checksum */
	uint16		pd_flags;	/* flag bits, see below */
	LocationIndex pd_lower;		/* offset to start of free space */
	LocationIndex pd_upper;		/* offset to end of free space */
	LocationIndex pd_special;	/* offset to start of special space */
	uint16		pd_pagesize_version;
	TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
	ItemIdData	pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;

typedef PageHeaderData *PageHeader;
```
