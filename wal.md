PostgreSQL starts to recover from REDO point, which is the location to write the XLOG record at the moment when the latest checkpoint is started. In fact, the database recovery processing is strongly linked to the checkpoint processing and both of these processing are inseparable.

LSN (Log Sequence Number) of XLOG record represents the location where its record is written on the transaction log. LSN of record is used as the unique id of XLOG record.

## Full Page Writes

Full page writes are needed because a page write that is in process during an operating system crash might be only partially completed, leading to an on-disk page that contains a mix of old and new data. The row-level change data normally stored in WAL will not be enough to completely restore such a page during post-crash recovery. Storing the full page image guarantees that the page can be correctly restored.

## Internal Layout of WAL Segment

A WAL segment is a 16 MB file, by default, and it is internally divided into pages of 8192 bytes (8 KB). The first page has a header-data defined by the structure XLogLongPageHeaderData, while the headings of all other pages have the page information defined by the structure XLogPageHeaderData. Following the page header, XLOG records are written in each page from the beginning in descending order. See Fig. 9.7.

## Internal Layout of XLOG Record

An XLOG record comprises the general header portion and each associated data portion. The general header portion is defined as XLogRecord:
```
typedef struct XLogRecord
{
   uint32          xl_tot_len;   /* total len of entire record */
   TransactionId   xl_xid;       /* xact id */
   uint32          xl_len;       /* total len of rmgr data */
   uint8           xl_info;      /* flag bits, see below */
   RmgrId          xl_rmid;      /* resource manager for this record */
   /* 2 bytes of padding here, initialize to zero */
   XLogRecPtr      xl_prev;      /* ptr to previous record in log */
   pg_crc32        xl_crc;       /* CRC for this record */
} XLogRecord;
```

Both xl_rmid and xl_info are variables related to resource managers, which are collections of operations associated with the WAL feature such as writing and replaying of XLOG records.
| Operation | Resource Manager |
| --- | --- |
| Heap tuple operations | RM_HEAP, RM_HEAP2 |	
| Index operations	| RM_BTREE, RM_HASH, RM_GIN, RM_GIST, RM_SPGIST, RM_BRIN |
| Sequence operations	| RM_SEQ |
| Transaction operations	| RM_XACT, RM_MULTIXACT, RM_CLOG, RM_XLOG, RM_COMMIT_TS |
| Tablespace operations	| RM_SMGR, RM_DBASE, RM_TBLSPC, RM_RELMAP |
| Replication and hot standby operations	| RM_STANDBY, RM_REPLORIGIN, RM_GENERIC_ID, RM_LOGICALMSG_ID |

Here are some representative examples how resource managers work in the following:

### If INSERT statement is issued, the header variables xl_rmid and xl_info of its XLOG record are set to 'RM_HEAP' and 'XLOG_HEAP_INSERT' respectively. When recovering the database cluster, the RM_HEAP's function heap_xlog_insert() selected according to the xl_info replays this XLOG record.
### Though it is similar for UPDATE statement, the header variable xl_info of the XLOG record is set to 'XLOG_HEAP_UPDATE', and the RM_HEAP's function heap_xlog_update() replays its record when the database recovers.
### When a transaction commits, the header variables xl_rmid and xl_info of its XLOG record are set to 'RM_XACT' and to 'XLOG_XACT_COMMIT' respectively. When recovering the database cluster, the function xact_redo_commit() replays this record.
