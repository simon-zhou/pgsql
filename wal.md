PostgreSQL starts to recover from REDO point, which is the location to write the XLOG record at the moment when the latest checkpoint is started. In fact, the database recovery processing is strongly linked to the checkpoint processing and both of these processing are inseparable.

LSN (Log Sequence Number) of XLOG record represents the location where its record is written on the transaction log. LSN of record is used as the unique id of XLOG record.

## Full Page Writes

Full page writes are needed because a page write that is in process during an operating system crash might be only partially completed, leading to an on-disk page that contains a mix of old and new data. The row-level change data normally stored in WAL will not be enough to completely restore such a page during post-crash recovery. Storing the full page image guarantees that the page can be correctly restored.

## Internal Layout of WAL Segment

A WAL segment is a 16 MB file, by default, and it is internally divided into pages of 8192 bytes (8 KB). The first page has a header-data defined by the structure **XLogLongPageHeaderData**, while the headings of all other pages have the page information defined by the structure **XLogPageHeaderData**. Following the page header, XLOG records are written in each page from the beginning in descending order. See Fig. 9.7.

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

- If INSERT statement is issued, the header variables xl_rmid and xl_info of its XLOG record are set to 'RM_HEAP' and 'XLOG_HEAP_INSERT' respectively. When recovering the database cluster, the RM_HEAP's function heap_xlog_insert() selected according to the xl_info replays this XLOG record.

- Though it is similar for UPDATE statement, the header variable xl_info of the XLOG record is set to 'XLOG_HEAP_UPDATE', and the RM_HEAP's function heap_xlog_update() replays its record when the database recovers.

- When a transaction commits, the header variables xl_rmid and xl_info of its XLOG record are set to 'RM_XACT' and to 'XLOG_XACT_COMMIT' respectively. When recovering the database cluster, the function xact_redo_commit() replays this record.

Data portion of XLOG record is classified into either backup block (entire page) or non-backup block (different data by operation).

```
typedef struct BkpBlock @ include/access/xlog_internal.h
{
  RelFileNode node;        /* relation containing block */
  ForkNumber  fork;        /* fork within the relation */
  BlockNumber block;       /* block number */
  uint16      hole_offset; /* number of bytes before "hole" */
  uint16      hole_length; /* number of bytes in "hole" */

  /* ACTUAL BLOCK DATA FOLLOWS AT END OF STRUCT */
} BkpBlock;

typedef struct xl_heap_insert
{
   xl_heaptid      target;              /* inserted tuple id */
   bool            all_visible_cleared; /* PD_ALL_VISIBLE was cleared */
} xl_heap_insert;
```

There are different WAL operations defined in heapam_xlog.h:
```
/*
 * WAL record definitions for heapam.c's WAL operations
 *
 * XLOG allows to store some information in high 4 bits of log
 * record xl_info field.  We use 3 for opcode and one for init bit.
 */
#define XLOG_HEAP_INSERT                0x00
#define XLOG_HEAP_DELETE                0x10
#define XLOG_HEAP_UPDATE                0x20
#define XLOG_HEAP_TRUNCATE              0x30
#define XLOG_HEAP_HOT_UPDATE    0x40
#define XLOG_HEAP_CONFIRM               0x50
#define XLOG_HEAP_LOCK                  0x60
#define XLOG_HEAP_INPLACE               0x70

#define XLOG_HEAP_OPMASK                0x70
/*
 * When we insert 1st item on new page in INSERT, UPDATE, HOT_UPDATE,
 * or MULTI_INSERT, we can (and we do) restore entire page in redo
 */
#define XLOG_HEAP_INIT_PAGE             0x80
/*
 * We ran out of opcodes, so heapam.c now has a second RmgrId.  These opcodes
 * are associated with RM_HEAP2_ID, but are not logically different from
 * the ones above associated with RM_HEAP_ID.  XLOG_HEAP_OPMASK applies to
 * these, too.
 */
#define XLOG_HEAP2_REWRITE              0x00
#define XLOG_HEAP2_CLEAN                0x10
#define XLOG_HEAP2_FREEZE_PAGE  0x20
#define XLOG_HEAP2_CLEANUP_INFO 0x30
#define XLOG_HEAP2_VISIBLE              0x40
#define XLOG_HEAP2_MULTI_INSERT 0x50
#define XLOG_HEAP2_LOCK_UPDATED 0x60
#define XLOG_HEAP2_NEW_CID              0x70
```

In version 9.4 or earlier, there was no common format of XLOG record, so that each resource manager had to define one’s own format. In such a case, it became increasingly difficult to maintain the source code and to implement new features related to WAL. In order to deal with this issue, a common structured format, which does not depend on resource managers, has been introduced in version 9.5.

Data portion of XLOG record can be divided into two parts: header and data. Header part contains zero or more XLogRecordBlockHeaders and zero or one XLogRecordDataHeaderShort (or XLogRecordDataHeaderLong); it must contain at least either one of those. When its record stores full-page image (i.e. backup block), XLogRecordBlockHeader includes XLogRecordBlockImageHeader, and also includes XLogRecordBlockCompressHeader if its block is compressed. Data part is composed of zero or more block data and zero or one main data, which correspond to the XLogRecordBlockHeader(s) and to the XLogRecordDataHeader respectively. Here are a few examples (credit: www.interdb.jp):

<img src="xlog.png" alt="hi" class="inline"/>

Below explains the order of a tuple insert. Be noted that XLogInsert happens within heap_insert and finish_xact_command, XLogWrite happens within finish_xact_command.

```
exec_simple_query() @postgres.c

(1) ExtendCLOG() @clog.c                  /* Write the state of this transaction
                                           * "IN_PROGRESS" to the CLOG.
                                           */
(2) heap_insert()@heapam.c                /* Insert a tuple, creates a XLOG record,
                                           * and invoke the function XLogInsert.
                                           */
(3)   XLogInsert() @xlog.c (9.5 or later, xloginsert.c)
                                          /* Write the XLOG record of the inserted tuple
                                           *  to the WAL buffer, and update page's pd_lsn.
                                           */
(4) finish_xact_command() @postgres.c     /* Invoke commit action.*/   
      XLogInsert() @xlog.c  (9.5 or later, xloginsert.c)
                                          /* Write a XLOG record of this commit action 
                                           * to the WAL buffer.
                                           */
(5)   XLogWrite() @xlog.c                 /* Write and flush all XLOG records on 
                                           * the WAL buffer to WAL segment.
                                           */
(6) TransactionIdCommitTree() @transam.c  /* Change the state of this transaction 
                                           * from "IN_PROGRESS" to "COMMITTED" on the CLOG.
                                           */
```

There are a few cases PG writes xlog records into the WAL segment:
- One running transaction has committed or has aborted (abort has xlog?)
- The WAL buffer has been filled up with many tuples. (wal_buffers)
- WAL writer process writes periodically.

Be noted that SELECT statement may also write xlog records, when deletion of unnecessary tuples and defragmentation of the necessary tuples in pages with HOT.

WAL writer is a background process to check the WAL buffer periodically to write all unwritten xlog records into the WAL segments. The purpose of this process is to avoid burst of xlog records. In version 9.1 or earlier, the background writer process did both checkpointing and dirty-page writing.

Not to be confused with WAL writer process, checkpointing process prepares database recovery (redo point) and clean dirty pages in the shared buffer pool. Here are what checkpointing process does:

- After a checkpoint process starts, the REDO point is stored in memory; REDO point is the location to write the XLOG record at the moment when the latest checkpoint is started, and is the starting point of database recovery.
- A XLOG record of this checkpoint (i.e. checkpoint record) is written to the WAL buffer. The data-portion of the record is defined by the structure CheckPoint, which contains several variables such as the REDO point stored with step (1).
In addition, the location to write checkpoint record is literally called the checkpoint.
- All data in shared memory (e.g. the contents of the clog, etc..) are flushed to the storage.
- All dirty pages on the shared buffer pool are written and flushed into the storage, gradually.
- The pg_control file is updated. This file contains the fundamental information such as the location where the checkpoint record has written (a.k.a. checkpoint location). The details of this file later.

To summarize the description above from the viewpoint of the database recovery, checkpointing creates the checkpoint record which contains the REDO point, and stores the checkpoint location and more into the pg_control file. Therefore, PostgreSQL enables to recover itself by replaying WAL data from the REDO point (obtained from the checkpoint record) provided by the pg_control file.

```
typedef struct CheckPoint
{
  XLogRecPtr      redo;           /* next RecPtr available when we began to
                                   * create CheckPoint (i.e. REDO start point) */
  TimeLineID      ThisTimeLineID; /* current TLI */
  TimeLineID      PrevTimeLineID; /* previous TLI, if this record begins a new
                                   * timeline (equals ThisTimeLineID otherwise) */
  bool            fullPageWrites; /* current full_page_writes */
  uint32          nextXidEpoch;   /* higher-order bits of nextXid */
  TransactionId   nextXid;        /* next free XID */
  Oid             nextOid;        /* next free OID */
  MultiXactId     nextMulti;      /* next free MultiXactId */
  MultiXactOffset nextMultiOffset;/* next free MultiXact offset */
  TransactionId   oldestXid;      /* cluster-wide minimum datfrozenxid */
  Oid             oldestXidDB;    /* database with minimum datfrozenxid */
  MultiXactId     oldestMulti;    /* cluster-wide minimum datminmxid */
  Oid             oldestMultiDB;  /* database with minimum datminmxid */
  pg_time_t       time;           /* time stamp of checkpoint */

 /*
  * Oldest XID still running. This is only needed to initialize hot standby
  * mode from an online checkpoint, so we only bother calculating this for
  * online checkpoints and only when wal_level is hot_standby. Otherwise
  * it's set to InvalidTransactionId.
  */
  TransactionId oldestActiveXid;
} CheckPoint;
```

## Database Recovery in PostgreSQL

The first thing is how PostgreSQL begin the recovery process. When PostgreSQL starts up, it reads the pg_control file at first. The followings are the details of the recovery processing from that point.

- PostgreSQL reads all items of the pg_control file when it starts. If the state item is in 'in production', PostgreSQL will go into recovery-mode because it means that the database was not stopped normally; if 'shut down', it will go into normal startup-mode.
- PostgreSQL reads the latest checkpoint record, which location is written in the pg_control file, from the appropriate WAL segment file, and gets the REDO point from the record. If the latest checkpoint record is invalid, PostgreSQL reads the one prior to it. If both records are unreadable, it gives up recovering by itself. (Note that the prior checkpoint is not stored from PostgreSQL 11.)
- Proper resource managers read and replay XLOG records in sequence from the REDO point until they come to the last point of the latest WAL segment. When a XLOG record is replayed and if it is a backup block, it will be overwritten on the corresponding table's page regardless of its LSN. Otherwise, a (non-backup block's) XLOG record will be replayed only if the LSN of this record is larger than the pd_lsn of a corresponding page.

