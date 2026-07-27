This article will introduce the optimization strategy for Sysbench performance testing, including tuning at the database layer and the database operating environment.

## Environment Preparation

- Hardware environment information is as follows:

    - Server: Kunpeng 920, 128-core CPU, 4 NUMA nodes

    - Memory: 381G
    - Network card: 10GE
    - Hard disk: NVMe SSD

- Software environment information is as follows:

    - Database: YashanDB v23.3 and above

    - Test tool: Sysbench 1.1.0

## Tuning Operations

### Operating System Tuning

In a NUMA architecture, when performing Sysbench performance tests, operating system-level tuning is crucial for fully utilizing database performance.

#### Disable Transparent Huge Pages

Transparent huge pages may cause memory allocation delays. It is recommended to disable transparent huge pages:

```bash
# Temporarily disable transparent huge pages (ineffective after reboot)
$ echo never > /sys/kernel/mm/transparent_hugepage/enabled
$ echo never > /sys/kernel/mm/transparent_hugepage/defrag

# # To permanently turn off transparent huge pages, the GRUB configuration needs to be modified.

# CentOS/RHEL
$ vi /etc/default/grub
# Add to GRUB_CMDLINE_LINUX: transparent_hugepage=never

# KylinOS V10
# View the current grub configuration
$ cat /boot/efi/EFI/*/grubenv
# Edit the grub configuration
$ vi /etc/default/grub
# Add "transparent_hugepage=never" to GRUB_CMDLINE_LINUX
# Regenerate the grub configuration
$ grub2-mkconfig -o /boot/efi/EFI/*/grub.cfg
```

#### Adjust CPU Scheduling Policy

For latency-sensitive applications like databases, it is recommended to use the performance scheduling policy:

```bash
# Set CPU scheduling policy to performance
$ cpupower frequency-set -g performance

# View current CPU scheduling policy
$ cpupower frequency-info
```

#### Network Configuration Tuning

Similar to TPC-C testing, when the client and server are deployed on different servers, network transmission will become a performance bottleneck. Network performance can be improved by optimizing network card interrupt queues.

```bash
# View network card device name
$ ifconfig

# View network card interrupt queue information
$ ethtool -l enp1s0f0np0

# Set network card interrupt queue number
$ ethtool -L enp1s0f0np0 combined 32
```

### Database Tuning

#### Database Process Core Binding

In multi-NUMA node environments, binding the database process to specific NUMA nodes can improve cache hit rate:

```bash
# Bind database process to NUMA node 1,2,3
$ numactl --cpunodebind=1,2,3 --membind=1,2,3 yasdb open
```

#### Database File Partition Storage

Rationally plan the storage location of data files, deploy redo files and data files on separate disks:

```sql
-- When creating a database, store redo files and data files on separate disks
CREATE DATABASE yashandb
LOGFILE(
    '/data1/redo1' size 50G BLOCKSIZE 512,
    '/data2/redo2' size 50G BLOCKSIZE 512,
    '/data3/redo3' size 50G BLOCKSIZE 512,
    '/data4/redo4' size 50G BLOCKSIZE 512,
    '/data5/redo5' size 50G BLOCKSIZE 512,
    '/data6/redo6' size 50G BLOCKSIZE 512,
    '/data7/redo7' size 50G BLOCKSIZE 512,
    '/data8/redo8' size 50G BLOCKSIZE 512,
    '/data9/redo9' size 50G BLOCKSIZE 512,
    '/data10/redo10 size 50G BLOCKSIZE 512)
UNDO TABLESPACE DATAFILE '/data2/undo' size 40G
DEFAULT TABLESPACE DATAFILE '/data2/users' size 500G;
```

| Configuration Item | Recommended Value |
| --- | --- |
| Redo disk-separated deployment | Size: 50G * 10, BLOCKSIZE 512 |
| Undo size | Increased to 40G |
| tablespace | Create according to actual data volume, need to create in advance |

#### Database Core Parameter Tuning

The following lists the recommended database core configuration parameters for Sysbench testing:

| Parameter | Description | Recommended Value |
| --- | --- | --- |
| DATA_BUFFER_SIZE | Affects database cache buffer size | Configure according to memory |
| MAX_SESSIONS | Affects actual concurrency | Configure according to concurrency requirements |
| MAX_REACTOR_CHANNELS | Recommended to enable shared mode under high concurrency | Larger value |
| MAX_WORKERS | Worker thread count | Depends on actual test results |
| SHARE_POOL_SIZE | Shared pool size, needs to be set larger under high concurrency | 10G recommended for 2000 concurrency |
| OPEN_CURSORS | Need to increase under high concurrency | Larger value |
| UNDO_RETENTION | Undo retention time | Configure according to requirements |
| UNDO_SHRINK_ENABLED | Whether to enable Undo auto-shrink | FALSE |
| _CONSISTENT_WRITE | Statement-level consistent write, avoid index inconsistency | TRUE |

#### Performance Optimization Parameters

To pursue higher performance, consider setting the following parameters:

| Parameter | Description | Recommended Value |
| --- | --- | --- |
| DOUBLE_WRITE_ENABLED | Whether to enable double write | FALSE |
| COMMIT_WAIT | Commit wait strategy | NOWAIT |

> **Caution**:
>
> Disabling double write (DOUBLE_WRITE_ENABLED=FALSE) can improve performance but increases the risk of data loss. It is recommended to use only in test environments.

#### Database Parameter Tuning

The following lists the recommended database configuration parameters for Sysbench testing:

```bash
# Data buffer size, recommended to configure 80% of planned memory
DATA_BUFFER_SIZE=180G

# Data buffer partitions, can reduce buffer lock conflicts in high concurrency scenarios
_DATA_BUFFER_PARTS=8

# VM buffer size
VM_BUFFER_SIZE=30G

# VM buffer partitions
VM_BUFFER_PARTS=8

# Undo retention time
UNDO_RETENTION=30

# Disable undo auto-shrink
UNDO_SHRINK_ENABLED=FALSE

# Checkpoint timeout
CHECKPOINT_TIMEOUT=1000000000

# Share pool size
SHARE_POOL_SIZE=2G

# Session reserved cursors
_SESSION_RESERVED_CURSORS=64

# Large page memory
USE_LARGE_PAGES=ONLY
LARGE_POOL_SIZE=4G

# Redo log direct write mode
REDOFILE_IO_MODE=DIRECT
```

### Sysbench Test Parameter Tuning

#### Concurrency Thread Count Tuning

The number of concurrent threads is a key factor influencing the results of Sysbench tests. Generally, it is recommended to start testing with a relatively low number of concurrent threads and then gradually increase it to find the optimal performance point. It is recommended to increase in a stepped manner according to `--threads=32 → 64 → 128 → 256 → 512 → 1024`, and closely monitor the inflection points of TPS and changes in latency.

```bash
# Progressive stress testing, starting from 32 threads, gradually increasing
for threads in 32 64 128 256 512; do
    echo "Testing with $threads threads..."
    /home/yashan/sysbench/src/sysbench /home/yashan/sysbench/src/lua/oltp_read_write.lua \
        --report-interval=5 \
        --tables=10 \
        --table_size=100000 \
        --yashan-db="192.168.1.2:1688" \
        --yashan-user="sysbench" \
        --yashan-password="$YASHAN_PASSWORD" \
        --time=300 \
        --threads=$threads \
        --thread-init-timeout=1000 \
        run \
        --percentile=99
done
```

#### Table Count and Size Tuning

Select appropriate table count and size according to test objectives:

```bash
# Reduce table count, increase rows per table, suitable for testing large table performance
--tables=5
--table_size=200000

# Increase table count, reduce rows per table, suitable for testing high-concurrency small table access
--tables=20
--table_size=50000
```

#### Test Scenario Selection

Select appropriate test scenario according to test objectives:

| Scenario | Focus |
| --- | --- |
| point_select | Primary key single-row query performance |
| read_only | Complex read-only query and index scan capability |
| write_only | Pure write (INSERT/UPDATE/DELETE) performance |
| read_write | Comprehensive OLTP transaction processing capability |
| index_update | Index column update performance |
| non_index_update | Non-index column update performance |

### Metrics to Focus on During Tuning

During the optimization process, pay attention to changes in the following key metrics:

- **TPS (transactions per second)**: Transactions per second, core metric for measuring system throughput

- **QPS (queries per second)**: Queries per second, reflects database's ability to handle queries

- **latency**: Transaction response time, including average response time and percentile response time

- **threads fairness**: Thread fairness metric, smaller stddev indicates more balanced thread loads

By analyzing changes in the above metrics, you can determine whether the current configuration has achieved optimal performance and guide further optimization directions.

## Performance Problem Location

Locate performance problems based on monitoring data:

| Phenomenon | Possible Cause | Solution |
| ---------------- | -------------------- | ---------------- |
| Low TPS, high CPU usage | Low SQL efficiency, full table scans | Add indexes, optimize SQL |
| Low TPS, high IO usage | Disk IO bottleneck                                           | Use higher performance SSDs, optimize data layout |
| Low TPS, high memory usage | Insufficient memory, frequent paging | Increase memory, enlarge buffer |
| High latency, long response time | Lock contention or wait events | Analyze wait events, optimize concurrency control |
| Unbalanced thread load | Database process not bound to cores or CPU scheduling issues | Bind database process to specific NUMA nodes |