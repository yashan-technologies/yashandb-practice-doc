## Testing Environment Information

### Server-side Environment Information

::: tabs
== ARM

- Hardware environment information is as follows:

    - CPU: Kunpeng-920 HiSilicon (4 nodes)
    - Memory: 509G

    - Network Card: 10GE
    - Hard Disk: NVME SSD / Huawei Dorado 6800 NVMe shared storage

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
    kernel.numa_balancing = 0
    kernel.core_pattern=/home/jenkins/core-%e-%p-%t
    net.core.somaxconn = 65535
    net.ipv4.tcp_max_syn_backlog = 65535
    ```

== Hygon

- Hardware environment information is as follows:

    - CPU: Hygon C86-3G 7390 32-core Processor (4 nodes)

    - Memory: 502G
    - Network Card: 10GE
    - Hard Disk: NVME SSD / Huawei Dorado 6800 NVMe shared storage

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
    kernel.pid_max = 1310712
    fs.file-max = 6815744
    fs.aio-max-nr = 1048576
    ```

:::

###  Client-side Environment Information

- CPU：Intel(R) Xeon(R) Gold 6338 CPU @ 2.00GHz  vCPU = 128

- Memory: 377G

### Software Information

- Database: YashanDB v23.4.14.100

- Testing Tool: Sysbench 1.1.0
- Deployment Forms: YAC Deployment (single cluster) / Primary-Standby Cluster Deployment

The database instance creation parameters used in this test are as follows:

```sql
cluster = "yashandb"
create_simple_schema = false
mode = "YASHAN"
uuid = "bf60fcb174f637f4fca353b018759c98"
yas_type = "CE"
```

The configuration of the database system parameters is as follows:

```bash
DATA_BUFFER_SIZE = "300G"
_DATA_BUFFER_PARTS=8
VM_BUFFER_SIZE="50G"
VM_BUFFER_PARTS=12
_REPLICATION_BUFFER_SIZE="128M"
REDO_BUFFER_SIZE="64M"
REDO_BUFFER_PARTS = 12
LARGE_POOL_SIZE="2G"
UNDO_RETENTION=30
WORK_AREA_POOL_SIZE="3G"
UNDO_SHRINK_ENABLED="FALSE"
_SESSION_RESERVED_CURSORS=64
WORK_AREA_HEAP_SIZE="2M"
INTERCONNECT_LINKS = 16
LOCK_POOL_SIZE="3G"
SHARE_POOL_SIZE="30G"
SQL_POOL_PARTS = 8
DBWR_COUNT=16
DBWR_BUFFER_SIZE="32M"
CHECKPOINT_INTERVAL=0
RECOVERY_PARALLELISM = 128
MAX_SESSIONS = 8192
_UNDO_MAX_AUTOEXTEND_SEGMENTS = 256
STATISTICS_LEVEL="BASIC"
_SESSION_RESERVED_LOCKS=32
_WAIT_BOC = "FALSE"
OPEN_CURSORS=4096
```

## Test Metrics Reference

The following test metrics are test data under the standard Sysbench OLTP scenario. The test configuration is 128 tables with 250,000 rows per table, and the test duration is 5 minutes.

### Single Cluster Read-Only Scenario

|Chip Platform |Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| --- | --- | --- | --- | --- | --- |
| ARM | 256 | 109769.68   | 1756314.94  | 2.33                                    | 3.89                               |
| ARM | 512 | 126111.69   | 2017787.03  | 4.05                                    | 4.65                               |
| Hygon | 256 | 98383.11    | 1574129.75  | 2.60                                    | 3.55                               |
| Hygon | 512 | 123593.75   | 1977499.95  | 4.14                                    | 5.99                               |

### Single Cluster Read-Write Mixed Scenario

|Chip Platform |Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| --- | --- | --- | --- | --- | --- |
| ARM | 256 | 28672.43    | 573448.54   | 8.92                                    | 14.73                              |
| ARM                        | 512                    | 18411.06    | 548531.26   | 18.63                                   | 33.12                              |
| Hygon                      | 256                    | 20029.35    | 352936.40   | 14.50                                   | 27.66                              |
| Hygon                      | 512                    | 24254.18    | 520710.56   | 19.64                                   | 47.47                              |

### Primary-Standby Cluster Read-Write Mixed Scenario

|Chip Platform |Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| --- | --- | --- | --- | --- | --- |
| Hygon | 256 | 13067.53    | 261350.52   | 19.57                                   | 71.83                              |
| Hygon | 512 | 15827.68    | 316553.68   | 32.29                                   | 179.94                             |
