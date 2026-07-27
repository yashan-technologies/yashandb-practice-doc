This chapter will introduce the specific operations and related examples for performing Sysbench performance tests on a YashanDB standalone database.

> **Caution**：
>
> Stress test client should be directly connected to the database instance to ensure network latency < 0.2ms, avoiding network becoming a performance bottleneck.

## Install Sysbench

It is recommended to use the official pre-compiled package for installation:

```bash
## Fpr root user
# curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | bash
# yum -y install sysbench

# For non-root user (must have sudo permission)
$ curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
$ sudo yum -y install sysbench
```

Verify the installation:

```bash
$ sysbench --version
sysbench 1.1.0
```

## Install YashanDB Client

Sysbench connects to YashanDB via the YashanDB C driver. The YashanDB C driver installation files are integrated in the YashanDB client installation package, so you need to install the YashanDB client to provide the necessary library files.

1. From the YashanDB Official Website Download Center ([https://download.yashandb.com/download](https://download.yashandb.com/download)) or contact our technical support to obtain the corresponding software package.

2. Extract the installation package:

   ```bash
   tar -zxvf yashandb-client-xx.xx-linux-x86_64.tar.gz -C /home/
   ```

3. Configure environment variables:

   ```bash
   # Edit bashrc file
   $ vi ~/.bashrc
   
   # Add library path
   $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yashandb-client-xx.xx-linux-x86_64/lib
   
   # Refresh configuration
   $ source ~/.bashrc
   ```

## Introduction to Sysbench-related configurations

### Database Connection Parameter

Sysbench uses the following parameters to connect to YashanDB:

```bash
--yashan-db="192.168.1.2:1688" \
--yashan-user="sysbench" \
--yashan-password="$YASHAN_PASSWORD"
```

| Parameter | Description |
| --- | --- |
| --yashan-db | Database connection address in the format `host:port` |
| --yashan-user | Database username, requires CONNECT and RESOURCE permissions |
| --yashan-password | Database user password |

### General Parameters

| Parameter | Description | Default Value |
| --- | --- | --- |
| --threads=N / --num-threads=N | Concurrent test thread count | 1 |
| --time=N / --max-time=N | Total execution time limit in seconds (0 means unlimited) | 0 / 10 |
| --report-interval=N | Output intermediate statistics report every N seconds (0 means disabled) | 0 |
| --percentile=N | Percentile value for latency statistics | 95 |
| --thread-init-timeout=N | Thread initialization timeout | 60 |
| --tables=N | Number of test tables | / |
| --table-size=N | Number of rows per table | / |

## Run Sysbench Test

### Deploy YashanDB Database

For the specific installation and deployment operations of YashanDB database, please refer to [Installation and Deployment](https://doc.yashandb.com/yashandb/23.4/zh/Install-Guide/00Install-Guide.html) in the product documentation.

Performance data may vary due to differences in CPU, memory, IO, and network conditions in the test environment. To achieve optimal performance of YashanDB in the test environment, performance tuning needs to be performed based on the test environment configuration. For detailed information on database performance tuning, please refer to [Optimization Strategy](./Optimization Strategy).

Sysbench test optimization mainly focuses on the following aspects:

**Database Creation Configuration Tuning**

If storage conditions permit, deploy the redo file and data file on separate disks to reduce I/O contention between the two. 

| Configuration Item | Recommended Value |
| --- | --- |
| Redo separate disk deployment | Size 50G * 10, BLOCKSIZE 512 |
| Undo size | Increase to 40G |
| tablespace  | Create according to actual data volume, need to create in advance |

It is recommended to directly build a test database by configuring database creation parameters during installation.

| Configuration Item | Parameter Description  | Recommended Value |
| --- | --- | --- |
| REDO_FILE_NUM| Number of redo files |10|
| REDO_FILE_PATH | Path of redo files, which should be stored on separate disks | / |
| REDO_FILE_SIZE | Size of redo files | 50G |
| UNDO_FILE_INIT_SIZE | Initial size of the UNDO tablespace | 40G |
| SWAP_FILE_INIT_SIZE | Initial size of the SWAP tablespace | 10G |
| SYSTEM_FILE_INIT_SIZE | Initial size of the SYSTEM tablespace | 5G |
| SYSAUX_FILE_INIT_SIZE | Initial size of the SYSAUX tablespace | 5G |
| DATA_FILE_INIT_SIZE | Initial size of the default data tablespace |300G|


**Database [Configuration Parameters](https://doc.yashandb.com/yashandb-en/23.4/en/All-Manuals/Reference-Manual/Configuration-Parameters.html) Tuning**

In Sysbench test scenarios, focus mainly on cache size and concurrency parameter configuration:

```sql
-- Data buffer size for caching data blocks, recommended to configure 80% of planned memory
DATA_BUFFER_SIZE=200G

-- Data buffer partitions, can reduce buffer lock conflicts in high concurrency scenarios
_DATA_BUFFER_PARTS=8

-- VM buffer size for storing intermediate results of operations like sorting
VM_BUFFER_SIZE=25G

-- VM buffer partitions
VM_BUFFER_PARTS=8

-- Undo retention time, lowering it can improve performance in small transaction scenarios
UNDO_RETENTION=15

-- Checkpoint timeout
CHECKPOINT_TIMEOUT=900

-- Share pool size
SHARE_POOL_SIZE=2G

-- Statement-level consistent writes. After the test is completed, it is recommended to restore the relevant configurations. (If the environment is not specifically dedicated to Sysbench testing, it is recommended to add the row movement attribute to the test tables to achieve statement-level consistent writes.)
_CONSISTENT_WRITE=TRUE
```

### Prepare Test Data (prepare Phase)

> **Note**:
>
> - To avoid previous test data affecting results, it is recommended to run cleanup before prepare.
>
> - When repeating tests, it is recommended to manually execute checkpoint and switch redo.
>
> - It is recommended that the number of test tables be no less than 10, and the number of rows of data in a single table be no less than 5 million (when using SSD, it can be increased to more than 100 million rows).

Enter Sysbench directory and prepare test data (create tables and populate data). 

```bash
$ cd /home/yashan/sysbench

$ /home/yashan/sysbench/src/sysbench /home/yashan/sysbench/src/lua/oltp_read_write.lua \
--report-interval=5 \
--tables=10 \
--table_size=100000 \
--yashan-db="192.168.1.2:1688" \
--yashan-user="sysbench" \
--yashan-password="$YASHAN_PASSWORD" \
--threads=128 \
prepare
```

After executing the above command, Sysbench will automatically create tables with a fixed structure similar to the following and conduct related tests around the target table, simulating online transaction processing workloads by combining multiple SQL statements.

The table creation statement is as follows:

```sql
CREATE TABLE sbtest1 (
  id int(10) unsigned NOT NULL AUTO_INCREMENT,
  k int(10) unsigned NOT NULL DEFAULT '0',
  c char(120) NOT NULL DEFAULT '',
  pad char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (id),
  KEY k_1 (k)
);
```

The table structure is analyzed as follows:

| Column Name| Data Type | Index | Characteristics|
|---|---|---|---|
|id|int(10) unsigned|Primary key index (PRIMARY KEY)|Default auto-increment, can be disabled with --auto_inc=off|
|k|int(10) unsigned| Secondary index (KEY k_1)       |Core index column, used to simulate index query and update, can be disabled with --create_secondary=off|
|c|char(120)|None|Fixed-width character column, simulating non-indexed data, increasing row width and data volume|
|pad|char(60)|None|Fixed-width character column, simulating non-indexed data|

The table effectively examines database behavior under different IO modes and lock contention through the combination of **index column k** and **non-index columns c and pad**, simulating typical business table structures.


### Run Test (run Phase)

During the testing process, try to avoid restarting the database. Restarting will cause dirty pages to be written to the disk, triggering a large number of disk reading operations, which will have an adverse impact on performance. If a restart is truly necessary, after the restart, you can first run a read - only scenario test for a period of time to load data into the cache.

#### Read-Only Test

Execute the following command in the Sysbench directory for read-only testing.

```bash
$ /home/yashan/sysbench/src/sysbench /home/yashan/sysbench/src/lua/oltp_read_only.lua \
--report-interval=5 \
--tables=10 \
--table_size=100000 \
--yashan-db="192.168.1.2:1688" \
--yashan-user="sysbench" \
--yashan-password="$YASHAN_PASSWORD" \
--time=600 \
--threads=256 \
--thread-init-timeout=1000 \
run \
--percentile=99
```

The sample results are as follows:

```bash
SQL statistics:
queries performed:
    read:                            117330135
    write:                           31288036
    other:                           7822009
    total:                           156440180
transactions:                        7822009 (24254.18 per sec.)
queries:                             156440180 (520710.56 per sec.)
ignored errors:                      0      (0.00 per sec.)
reconnects:                          0      (0.00 per sec.)

Throughput:
events/s (eps):                      26035.5279
time elapsed:                        300.4360s
total number of events:              7822009

Latency (ms):
         min:                                    2.10
         avg:                                   19.64
         max:                                  850.86
         95th percentile:                       47.47
         sum:                            153597621.64

Threads fairness:
events (avg/stddev):           15277.3613/3574.44
execution time (avg/stddev):   299.9954/0.01
```

#### Read-Write Test

Execute the following command in the Sysbench directory for read-write mixed testing.

```bash
$ /home/yashan/sysbench/src/sysbench /home/yashan/sysbench/src/lua/oltp_read_write.lua \
--report-interval=5 \
--tables=10 \
--table_size=100000 \
--yashan-db="192.168.1.2:1688" \
--yashan-user="sysbench" \
--yashan-password="$YASHAN_PASSWORD" \
--time=600 \
--threads=256 \
--thread-init-timeout=1000 \
run \
--percentile=99
```

The sample results are as follows:

```bash
SQL statistics:
queries performed:
    read:                            109336335
    write:                           29156356
    other:                           7289089
    total:                           145781780
transactions:                        7289089 (24246.96 per sec.)
queries:                             145781780 (484939.16 per sec.)
ignored errors:                      0      (0.00 per sec.)
reconnects:                          0      (0.00 per sec.)

Throughput:
events/s (eps):                      24246.9580
time elapsed:                        300.6187s
total number of events:              7289089

Latency (ms):
         min:                                    2.90
         avg:                                   21.07
         max:                                  5016.59
         95th percentile:                       48.34
         sum:                            153598599.07

Threads fairness:
events (avg/stddev):           14236.5020/5975.94
execution time (avg/stddev):   299.9973/0.01
```

### Clean Up Test Data (cleanup Phase)

Execute the following command in the Sysbench directory to clean up test data.

```bash
$ /home/yashan/sysbench/src/sysbench /home/yashan/sysbench/src/lua/oltp_read_write.lua \
--yashan-db="192.168.1.2:1688" \
--yashan-user="sysbench" \
--yashan-password="$YASHAN_PASSWORD" \
cleanup
```
