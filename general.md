Benefits of using bind parameters:
- Avoid SQL injection, because user input can only be values of certain types, not the entire SQL statement.
- Performance gain, because the execution plan can be cached. However this only works with exactly the same SQL statement.

Not sure if this is true. It looks like when using bind parameters, the optimizer has no concrete values available to determine their frequency. It then just assumes an equal distribution and always gets the same row count estimates and cost values. In the end, it may always select the same sub-optimal execution plan. This is my impression after reading Chapter 2 of SQL Performance Explained.
