File paths consist of OIDs:
```
sampledb=# SELECT pg_relation_filepath('sampletbl');
 pg_relation_filepath 
----------------------
 base/16384/18812
(1 row)
```

Each file has a size limit 1GB. After that, there will be <relfilenode>.2 and so on. There is _fsm file and _vm file, for free space mapping and (page) visibility mapping.

A _tablespace_ in PG is an additional data area outside the base directory. The feature was implemented in version 8.0.

PostgreSQL also supports TID-Scan, Bitmap-Scan, and Index-Only-Scan. TID-Scan is a method that accesses a tuple directly by using TID of the desired tuple. For example, to find the 1st tuple in the 0-th page within the table, issue the following query:

```
qzhou=# SELECT ctid,id,name FROM fast WHERE ctid = '(0,1)';
 ctid  | id | name
-------+----+------
 (0,1) |  1 | ha
(1 row)
```
