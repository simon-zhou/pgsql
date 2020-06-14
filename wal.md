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


