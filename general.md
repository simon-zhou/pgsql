Benefits of using bind parameters:
- Avoid SQL injection, because user input can only be values of certain types, not the entire SQL statement.
- Performance gain, because the execution plan can be cached. However this only works with exactly the same SQL statement.
