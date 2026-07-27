## Testing Environment Information

### Basic Environment Information

::: tabs

== ARM

- Hardware environment information is as follows:

    - CPU：Kunpeng-920 HiSilicon

    - Memory: 381G

    - Network Card: 10GE
    - Hard Disk: NVMe SSD

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

    - CPU：Hygon C86-3G 7390 32-core Processor

    - Memory: 502G
    - Network Card: 10GE
    - Hard Disk: NVMe SSD

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

### Software Information

- Database: YashanDB v23.4.7.100

- Testing tool: Sysbench 1.1.0

The database instance creation parameters used in this test are as follows:

```sql
cluster = "yashandb"
create_simple_schema = false
mode = "YASHAN"
uuid = "bf60fcb174f637f4fca353b018759c98"
yas_type = "SE"
```

The configuration of the database system parameters is as follows:

```bash
DATA_BUFFER_SIZE = "180G"
_DATA_BUFFER_PARTS = 8
VM_BUFFER_SIZE = "30G"
VM_BUFFER_PARTS = 8
_REPLICATION_BUFFER_SIZE = "2G"
REDO_BUFFER_SIZE = "128M"
REDO_BUFFER_PARTS = 8
LARGE_POOL_SIZE = "2G"
WORK_AREA_POOL_SIZE = "2G"
UNDO_RETENTION = 30
UNDO_SHRINK_ENABLED = "FALSE"
_SESSION_RESERVED_CURSORS = 64
WORK_AREA_HEAP_SIZE = "2M"
CHECKPOINT_TIMEOUT = 1000000000
CHECKPOINT_INTERVAL = "128M"
SHARE_POOL_SIZE = "2G"
STATISTICS_LEVEL = "BASIC"
REDOFILE_IO_MODE = "DIRECT"
MAX_SESSIONS = 2048
DDL_LOCK_TIMEOUT = 3  
RECOVERY_PARALLELISM = 64
ARCH_CLEAN_IGNORE_MODE = "BACKUP"
ARCH_CLEAN_UPPER_THRESHOLD = "1K"
ARCH_CLEAN_LOWER_THRESHOLD = "0"
ARCHIVE_LOCAL_DEST = "/data03/jenkins/archive"
RECYCLEBIN_ENABLED = "OFF"
MAX_WORKERS = 550
STANDBY_RECOVER_ONLY = "true"
```

## Test Metrics Reference

The following test metrics are test data under the standard Sysbench OLTP scenario. The test configuration is 128 tables with 250,000 rows per table, and the test duration is 5 minutes.

### Standalone Single-Node Deployment Read-Only Scenario

| Chip Platform | Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| ---------------------------- | ------------------------ | ----------- | ----------- | --------------------------------------- | ---------------------------------- |
| ARM                          | 256                      | 83081.32    | 1329301.12  | 3.02                                    | 4.47                               |
| ARM                          | 512                      | 76413.34    | 1222613.44  | 6.43                                    | 8.25                               |
| Hygon                        | 256                      | 55194.04    | 883104.67   | 4.63                                    | 6.21                               |
| Hygon                        | 512                      | 57592.31    | 921476.96   | 8.87                                    | 26.20                              |

### Standalone Single-Node Deployment Read-Write Mixed Scenario

| Chip Platform | Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| ---------------------------- | ------------------------ | ----------- | ----------- | --------------------------------------- | ---------------------------------- |
| ARM                          | 256                      | 45678.88    | 913577.60   | 5.48                                    | 9.56                               |
| ARM                          | 512                      | 39896.32    | 797926.4    | 12.68                                   | 20.53                              |
| Hygon                        | 256                      | 36687.06    | 733741.28   | 6.97                                    | 10.46                              |
| Hygon                        | 512                      | 35035.09    | 700701.74   | 14.57                                   | 29.72                              |

### Standalone Primary-Standby Deployment Read-Only Scenario

| Chip Platform | Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| ---------------------------- | ------------------------ | ----------- | ----------- | --------------------------------------- | ---------------------------------- |
| ARM                          | 256                      | 82983.36    | 1327733.79  | 3.08                                    | 4.57                               |
| ARM                          | 512                      | 76313.55    | 1221016.83  | 6.70                                    | 8.43                               |
| Hygon                        | 256                      | 56189.23    | 899027.63   | 4.55                                    | 6.09                               |
| Hygon                        | 512                      | 58419.76    | 932716.13   | 8.75                                    | 23.74                              |

### Standalone Primary-Standby Deployment Read-Write Mixed Scenario

| Chip Platform | Concurrency |TPS |QPS |Average Latency (ms) |P95 Latency (ms) |
| ---------------------------- | ------------------------ | ----------- | ----------- | --------------------------------------- | ---------------------------------- |
| ARM                          | 256                      | 45478.91    | 909578.12   | 5.62                                    | 9.56                               |
| ARM                          | 512                      | 39697.22    | 793944.44   | 12.88                                   | 22.69                              |
| Hygon                        | 256                      | 36286.64    | 725732.90   | 7.04                                    | 10.46                              |
| Hygon                        | 512                      | 33644.54    | 672890.72   | 15.18                                   | 26.20                              |