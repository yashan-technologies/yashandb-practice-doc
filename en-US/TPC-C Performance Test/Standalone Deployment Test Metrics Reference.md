## Testing Environment Information

### Basic Environment Information

::: tabs

== ARM

- Hardware environment information is as follows:

    - Server: Kunpeng 920, 128-core CPU, 4 NUMA nodes

    - Memory: 381G

    - Network card: 10GE

    - Hard disk: NVME SSD

- OS information is as follows:

    ```bash
    kernel.sysrq=0
    net.ipv4.ip_forward=0
    net.ipv4.conf.all.send_redirects=0
    net.ipv4.conf.default.send_redirects=0
    net.ipv4.conf.all.accept_source_route=0
    net.ipv4.conf.default.accept_source_route=0
    net.ipv4.conf.all.accept_redirects=0
    net.ipv4.conf.default.accept_redirects=0
    net.ipv4.conf.all.secure_redirects=0
    net.ipv4.conf.default.secure_redirects=0
    net.ipv4.icmp_echo_ignore_broadcasts=1
    net.ipv4.icmp_ignore_bogus_error_responses=1
    net.ipv4.conf.all.rp_filter=1
    net.ipv4.conf.default.rp_filter=1
    net.ipv4.tcp_syncookies=1
    kernel.dmesg_restrict=1
    net.ipv6.conf.all.accept_redirects=0
    net.ipv6.conf.default.accept_redirects=0
    vm.nr_hugepages=480
    net.ipv4.tcp_syncookies = 1
    net.ipv4.conf.default.accept_source_route = 0
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.conf.all.arp_filter = 1
    net.ipv4.ip_local_port_range = 10000 65535
    kernel.pid_max = 1310712
    fs.file-max = 6815744
    fs.aio-max-nr = 1048576
    ```

== INTEL

- Hardware environment information is as follows:

    - CPU：Intel(R) Xeon(R) Gold 6230R CPU @ 2.10GHz  vCPU = 104

    - Memory: 376G

    - Network card: 10GE

    - Hard disk: NVME SSD

- OS information is as follows:

    ```bash
    kernel.shmmni = 4096
    kernel.sysrq = 1
    kernel.core_uses_pid = 1
    kernel.msgmnb = 65536
    kernel.msgmax = 65536
    kernel.msgmni = 2048
    net.ipv4.tcp_syncookies = 1
    net.ipv4.conf.default.accept_source_route = 0
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.conf.all.arp_filter = 1
    net.ipv4.ip_local_port_range = 10000 65535
    kernel.pid_max = 1310712
    fs.file-max = 6815744
    fs.aio-max-nr = 1048576
    vm.swappiness = 10
    ```

== Hygon

- Hardware environment information is as follows:

    - CPU：Hygon C86-3G 7390 32-core Processor  vCPU = 128

    - Memory: 376G

    - Network card: 10GE

    - Hard disk: NVME SSD

- OS information is as follows:

    ```bash
    net.ipv4.tcp_max_tw_buckets = 10000
    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_keepalive_time = 30
    net.ipv4.tcp_keepalive_intvl = 30
    net.ipv4.tcp_retries2 = 12
    net.ipv4.ip_local_reserved_ports = 15400-15407,20050-20057
    net.core.wmem_max = 21299200
    net.core.rmem_max = 21299200
    net.core.wmem_default = 21299200
    net.core.rmem_default = 21299200
    net.sctp.sctp_mem = 94500000 915000000 927000000
    net.sctp.sctp_rmem = 8192 250000 16777216
    net.sctp.sctp_wmem = 8192 250000 16777216
    kernel.sem = 250 6400000 1000 25600
    net.ipv4.tcp_rmem = 8192 250000 16777216
    net.ipv4.tcp_wmem = 8192 250000 16777216
    vm.min_free_kbytes = 39543473
    net.core.netdev_max_backlog = 65535
    net.ipv4.tcp_max_syn_backlog = 65535
    net.core.somaxconn = 65535
    kernel.shmall = 1152921504606846720
    kernel.shmmax = 18446744073709551615
    ```

::: 

### Software Information

- Database: YashanDB v23.4.6.100

- TPC-C Client: BenchMarkSQL5.0

The database instance creation parameters used in this test are as follows. You can either create the database instance directly during deployment, or deploy the database first, then delete the instance `DROP DATABASE` and recreate a new instance with `CREATE DATABASE`.

```sql
create database tpcc
logfile('/data2/jenkins/data/redo1' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo2' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo3' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo4' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo5' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo6' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo7' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo8' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo9' size 30G BLOCKSIZE 512,
        '/data2/jenkins/data/redo10' size 30G BLOCKSIZE 512)
UNDO TABLESPACE DATAFILE '?/data/undo' size 40G autoextend on next 256M maxsize 64G
SWAP TABLESPACE TEMPFILE '?/data/swap' size 40G autoextend on next 256M maxsize 64G
DEFAULT TABLESPACE DATAFILE '?/data/users' size 500G autoextend on next 256M;

create user tpcc identified by tpcc;
grant dba to tpcc;

create user regress identified by regress;
grant dba to regress;

DECLARE
  jobid INT;
BEGIN
  SELECT job INTO jobid FROM dba_jobs WHERE what LIKE '%GATHER_DATABASE_STATS%';
  DBMS_JOB.BROKEN(jobid,true);
  COMMIT;
END;
/
```

The configuration of the database system parameters is as follows:

::: tabs

== ARM

```bash
DATA_BUFFER_SIZE=200G
_DATA_BUFFER_PARTS=12
VM_BUFFER_SIZE=30G
VM_BUFFER_PARTS=12
_REPLICATION_BUFFER_SIZE=128M
REDO_BUFFER_SIZE=128M
REDO_BUFFER_PARTS=12
LARGE_POOL_SIZE=4G
WORK_AREA_POOL_SIZE=2G
UNDO_RETENTION=30
UNDO_SHRINK_ENABLED=FALSE
_SESSION_RESERVED_CURSORS=64
WORK_AREA_HEAP_SIZE=2M
CHECKPOINT_TIMEOUT=1000000000
MAX_WORKERS=1000
MAX_SESSIONS=2048
SHARE_POOL_SIZE=2G
STATISTICS_LEVEL=BASIC
USE_LARGE_PAGES=ONLY
REDOFILE_IO_MODE=DIRECT
```

== Intel

```bash
DATA_BUFFER_SIZE=180G
_DATA_BUFFER_PARTS=8
VM_BUFFER_SIZE=30G
VM_BUFFER_PARTS=8
_REPLICATION_BUFFER_SIZE=128M
REDO_BUFFER_SIZE=128M
REDO_BUFFER_PARTS=8
LARGE_POOL_SIZE=2G
WORK_AREA_POOL_SIZE=2G
UNDO_RETENTION=30
UNDO_SHRINK_ENABLED=FALSE
_SESSION_RESERVED_CURSORS=64
WORK_AREA_HEAP_SIZE=2M
CHECKPOINT_TIMEOUT=1000000000
LISTEN_ADDR=0.0.0.0:1689
SHARE_POOL_SIZE=2G
STATISTICS_LEVEL=BASIC
DDL_LOCK_TIMEOUT=3
REDOFILE_IO_MODE=DIRECT
MAX_SESSIONS=2048
MAX_WORKERS=300
```

== Hygon

```bash
REDOFILE_IO_MODE=DIRECT
CHECKPOINT_TIMEOUT=1000000000
MAX_SESSIONS=2048
RECOVERY_PARALLELISM=64
WORK_AREA_POOL_SIZE=2G
STATISTICS_LEVEL=BASIC
REDO_BUFFER_SIZE=128M
RUN_LOG_LEVEL=INFO
VM_BUFFER_SIZE=30G
CHECKPOINT_INTERVAL=128M
DATA_BUFFER_SIZE=180G
REDO_BUFFER_PARTS=8
_DATA_BUFFER_PARTS=8
_SPLIT_BRAIN_DATA_LIMIT=1024G
UNDO_RETENTION=30
UNDO_SHRINK_ENABLED=FALSE
LARGE_POOL_SIZE=2G
_REPLICATION_BUFFER_SIZE=128M
_SESSION_RESERVED_CURSORS=64
SHARE_POOL_SIZE=2G
VM_BUFFER_PARTS=8
WORK_AREA_HEAP_SIZE=2M
MAX_WORKERS=512
```
::: 

## Testing Environment Information

The following test metrics are uninterrupted testing of 1000 warehouses over a duration of 10 minutes.


|Chip Platform|Concurrency|Test Results (tpmC)|
|-----|----|------|
|ARM|256|166.3w|
|X86|256|145.2w|
|Hygon|512|143.2w|