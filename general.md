Benefits of using bind parameters:
- Avoid SQL injection, because user input can only be values of certain types, not the entire SQL statement.
- Performance gain, because the execution plan can be cached. However this only works with exactly the same SQL statement.

Not using bind parameters is like recompiling a program every time.

Not sure if this is true. It looks like when using bind parameters, the optimizer has no concrete values available to determine their frequency. It then just assumes an equal distribution and always gets the same row count estimates and cost values. In the end, it may always select the same sub-optimal execution plan. This is my impression after reading Chapter 2 of SQL Performance Explained.


This shows that a single SQL statement can use two indexes.
```
create table emp(id serial primary key, age integer, month integer);
create index emp_age on emp(age);
create index emp_month on emp(month);
insert into emp select i, i%100+1, i%12+1 from generate_series(1,5000000) s(i);
explain analyze select * from emp where month=10 and age > 40;

                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on emp  (cost=12035.00..32452.71 rows=10448 width=12) (actual time=255.396..792.847 rows=16666 loops=1)
   Recheck Cond: ((age > 95) AND (month = 10))
   Heap Blocks: exact=16999
   ->  BitmapAnd  (cost=12035.00..12035.00 rows=10448 width=0) (actual time=252.561..252.561 rows=0 loops=1)
         ->  Bitmap Index Scan on emp_age  (cost=0.00..4605.44 rows=128668 width=0) (actual time=156.433..156.433 rows=255000 loops=1)
               Index Cond: (age > 95)
         ->  Bitmap Index Scan on emp_month  (cost=0.00..7424.08 rows=206887 width=0) (actual time=94.522..94.522 rows=424999 loops=1)
               Index Cond: (month = 10)
 Planning Time: 7.509 ms
 Execution Time: 794.582 ms
```
