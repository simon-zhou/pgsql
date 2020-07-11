Pagination can be very useful when fetching large amount of data. The resulting challenge is that it has to skip the rows from the previous pages. There are two different methods to meet this challenge: firstly the offset method, which numbers the rows from the beginning and uses a filter on this row number to discard the rows before the requested page. The second method, searches the last entry of the previous page and fetches only the following rows.

The offset method is handy with databases that have a dedicated keyword for it. Example query:
```
select * from sales order by sale_date DESC offset 10 fetch next 10 rows only
```

Besides simplicity, another advantage of this method is that you just need the row offset to fetch an arbitrary page. However, this comes with two disadvantages:
1. The pages drift when inserting new sales because the numbering is always done from scratch
2. The response time increases linearly when browsing further back.

The second method is usally called "seek" and it avoids both problems because it uses the values of the previous page as a delimiter. That means it searches for the values that must come behind the last entry from the previous page. This can be expressed by a simple where clause. To put it the other way around: the seek method simply doesn't select already shown values.

The below example shows how seek method works. For the sake of demonstration, we assume that there is only one sale per day. This makes the SALE_DATE a unique key. To select the sales that must come behind a particular date you must use a less than condition because of the descending sort order. The "fetch first" clause is just used to limit the result to ten rows.
```
select * from sales where sale_date < ? order by sale_date desc fetch first 10 rows only
```

Instead of a row number, you use the last value of the previous page to specify the lower bound. This has a huge benefit in terms of performance because the database can use the sale_date < ? condition for index access. This means that the database can truly skip the rows from the previous pages. On top of that, you will also get stable results if new rows are inserted.

Nevertheless, this method doesn't work if there is more than one sale per day - because the last data from the first page is used to fetch the 2nd page which means all results from that day will be skipped, not just the ones already shown in the first page. The problem is that the order by clause doesn't establish a deterministic row sequence.

Without a deterministic order by clause, the database by definition does not deliver a deterministic row sequence. The only reason you usually get a consistent row sequence is that the database usually executes the query in the same way. The database could in fact shuffle the rows having the same sale_date and still fulfill the order by clause. In recent releases it might indeed happen that you get the result in a different order every time you run the query, not because the database shuffles the result intentionally but because the database might utilize parallel query execution. That means the same execution plan can result in a different row sequence because the executing threads finish in a non-determinsitic order.

To produce deterministic row sequence, we can extend the order by clause with arbitrary columns. If the index that is used for the pipelined order by clause has additional columns, it is a good start to add them to the order by clause so that we can continue using this index for the pipelined order by. If this still does not yield a deterministic sort order, we can just add any unique columns and extend the index accordingly. Assume the below example uses index idx_created_at_id(created_at, id) or idx_created_at(created_at):
```
select * from contact_type where (created_at,id) < ('2018-11-28 19:39:45.109256', 7) order by created_at desc, id desc fetch first 5 rows only;
```
