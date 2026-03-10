## Testing Environment Information

### Server Information

::: tabs

- Hardware environment information is as follows:

== ARM

- Server: Kunpeng 920, 128-core CPU, 4 NUMA nodes

- Memory: 512G

- Network card: 10GE

- Hard disk: NVME SSD

== Hygon

- CPU：Hygon C86-3G 7390 32-core Processor  vCPU = 128

- Memory: 512G

- Network card: 10GE

- Hard disk: NVME SSD

::: 

### Server Information of Client

- CPU：Intel(R) Xeon(R) Gold 6338 CPU @ 2.00GHz  vCPU = 128

- Memory: 377G

### Software Information

- Database: YashanDB v23.4.6.100

- TPC-C Client: BenchMarkSQL5.0

The database instance creation parameters used in this test are as follows. You can either create the database instance directly during deployment, or deploy the database first, then delete the instance `DROP DATABASE` and recreate a new instance with `CREATE DATABASE`.

```sql
create diskgroup DG0 external redundancy failgroup FG0 disk 
'/dev/yfs/datadisk1' force,
'/dev/yfs/datadisk2' force,
'/dev/yfs/datadisk4' force attribute 'au_size' = '32M';

create cluster database tpcc NOARCHIVELOG instance (
  logfile (
    '+DG0/redo11' size 100G BLOCKSIZE 512,
    '+DG0/redo12' size 100G BLOCKSIZE 512,
    '+DG0/redo13' size 100G BLOCKSIZE 512,
    '+DG0/redo14' size 100G BLOCKSIZE 512,
    '+DG0/redo15' size 100G BLOCKSIZE 512,
    '+DG0/redo16' size 100G BLOCKSIZE 512,
    '+DG0/redo17' size 100G BLOCKSIZE 512,
    '+DG0/redo18' size 100G BLOCKSIZE 512,
    '+DG0/redo19' size 100G BLOCKSIZE 512,
    '+DG0/redo20' size 100G BLOCKSIZE 512
  ) UNDO TABLESPACE DATAFILE '+DG0/undo1' size 64G
) instance (
  logfile (
    '+DG0/redo21' size 100G BLOCKSIZE 512,
    '+DG0/redo22' size 100G BLOCKSIZE 512,
    '+DG0/redo23' size 100G BLOCKSIZE 512,
    '+DG0/redo24' size 100G BLOCKSIZE 512,
    '+DG0/redo25' size 100G BLOCKSIZE 512,
    '+DG0/redo26' size 100G BLOCKSIZE 512,
    '+DG0/redo27' size 100G BLOCKSIZE 512,
    '+DG0/redo28' size 100G BLOCKSIZE 512,
    '+DG0/redo29' size 100G BLOCKSIZE 512,
    '+DG0/redo30' size 100G BLOCKSIZE 512
  ) UNDO TABLESPACE DATAFILE '+DG0/undo4' size 64G
) SWAP TABLESPACE TEMPFILE '+DG0/swap' size 64G DEFAULT TABLESPACE DATAFILE '+DG0/users' size 50G SYSTEM TABLESPACE DATAFILE '+DG0/system' size 2G SYSAUX TABLESPACE DATAFILE '+DG0/sysaux' size 1G TEMPORARY TABLESPACE TEMPFILE '+DG0/temp' size 1G UNDO_SEGMENTS 1024;
```

## Testing Environment Information

The following test metrics are uninterrupted testing over a duration of 10 minutes.


|Chip Platform|Instance Number|Warehouses|Concurrency|Test Results (tpmC)|
|-----|----|------|---|---|
|ARM|1|1000|400|140w|
|ARM|2|1000|800|217w|
|ARM|4|3000|1600|399w|
|Hygon|1|1000|400|93w|
|Hygon|2|1000|800|152w|
|Hygon|4|3000|1600|286w|