With a leaf page and its offset, we can get the **IndexTuple** this way:

```markdown
// bufpage.h
#define PageGetItemId(page, offsetNumber) \
	((ItemId) (&((PageHeader) (page))->pd_linp[(offsetNumber) - 1]))

// bufpage.h
#define PageGetItem(page, itemId) \
( \
	AssertMacro(PageIsValid(page)), \
	AssertMacro(ItemIdHasStorage(itemId)), \
	(Item)(((char *)(page)) + ItemIdGetOffset(itemId)) \
)

// nbtsearch.h _bt_compare
itup = (IndexTuple) PageGetItem(page, PageGetItemId(page, offnum));
```
