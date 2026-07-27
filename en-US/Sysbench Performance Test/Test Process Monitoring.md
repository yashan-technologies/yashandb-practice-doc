When performing Sysbench performance tests, monitoring the running status of the database and operating system can help analyze performance bottleneck points and guide further optimization directions. This chapter will introduce the monitoring metrics and methods that need to be paid attention to during Sysbench testing.

## Operating System Monitoring

### CPU Monitoring

Use top or mpstat commands to monitor CPU usage:

```bash
# View CPU usage in real-time
$ top -bn1 | head -20

# View usage per CPU core
$ mpstat -P ALL 1

# View load of each CPU core
$ sar -P ALL 1
```

Key monitoring metrics:

- **%usr**: User space CPU usage

- **%sys**: Kernel space CPU usage

- **%iowait**: CPU time ratio waiting for IO, high value indicates IO bottleneck

- **%idle**: Idle CPU ratio

### Memory Monitoring

```bash
# View memory usage
$ free -h

# Monitor memory changes in real-time
$ vmstat 1

# View detailed memory information
$ cat /proc/meminfo
```

Key monitoring metrics:

- **used**: Used memory

- **free**: Free memory

- **buff/cache**: Buffer/cache memory

- **available**: Available memory

- **si/so**: Swap in/out memory, high value indicates insufficient memory

### IO Monitoring

```bash
# View IO usage
$ iostat -x 1

# View detailed IO device information
$ iostat -dx 1

# Monitor IO request queue length
$ sar -d 1
```

Key monitoring metrics:

- **%util**: IO device utilization, high value indicates IO bottleneck

- **r/s, w/s**: Read/write requests per second

- **rMB/s, wMB/s**: Bytes read/written per second

- **await**: Average IO request wait time

- **svctm**: Average IO request service time

### Disk Performance Testing

Use fio testing tool:

| Test Scenario | Description |
| --- | --- |
| Redo Disk | Synchronous write, sequential write, blocksize=512 |
| Data Disk | Synchronous write, random write/read, blocksize=8k |
| Cluster Scenario | Use direct io |

```bash
#!/bin/bash
DEVICE=/home/wangshaohua/fio_test
RUNTIME=30

echo "=== Synchronous I/O Sequential Write ==="
sudo fio --name=sync_seq --filename=$DEVICE --direct=0 --rw=write \
  --bs=4k --numjobs=1 --iodepth=1 --runtime=$RUNTIME --time_based \
  --group_reporting --ioengine=sync --size=1G

echo "=== Synchronous I/O Random Write ==="
sudo fio --name=sync_rand --filename=$DEVICE --direct=0 --rw=randwrite \
  --bs=4k --numjobs=16 --iodepth=1 --runtime=$RUNTIME --time_based \
  --group_reporting --ioengine=sync --size=1G
```

### Network Monitoring

```bash
# View network interface statistics
$ sar -n DEV 1

# View network connection status
$ netstat -an | grep -E "Establ|Wait"

# View detailed network statistics
$ ss -s
```

Key monitoring metrics:

- **rxpck/s, txpck/s**: Packets received/sent per second

- **rxKB/s, txKB/s**: Bytes received/sent per second

- **rxerrs, txerrs**: Receive/send error counts

## Database Monitoring

### Read Scenario Analysis

- Focus on whether all data can be cached

  ```sql
  -- 1. View DATA_BUFFER_SIZE parameter
  show parameter data_buffer_size

  -- 2. View data file size for the corresponding sysbench user
  SELECT SUM(BYTES) AS TOTAL_USER_SPACE FROM DBA_SEGMENTS WHERE OWNER = upper('username');

  -- 3. View DATA_BUFFER_SIZE overall statistics
  SELECT * FROM V$BUFFER_POOL_STATISTICS;
  ```

- Pay attention to the disk random read performance

  ```bash
  # 1. Confirm whether the disk bandwidth upper limit of rMB/s is reached, whether the disk utilization rate %util reaches 100%, check the IO latency situation and the r_wait time.
  $ iostat -xm 1
  
  # 2. Check the IO usage of each thread in the database. This can be compared with the IO under a single - thread scenario to confirm whether there is a situation where thread IO is preempted, leading to performance degradation
  $ sudo iotop
  ```

### Write Scenario Analysis

- Pay attention to whether the redo is deployed on separate disks.

  In write scenarios, a large amount of redo logs will be generated (approximately 8K is generated per transaction. This data is for reference only, and in practice, it needs to be judged by combining time - based observations). By configuring redo to be on separate disks to make full use of disk performance, the actual test results can be significantly improved. Redo operations are sequential writes and are pure write - I/O; while data operations involve both reading and writing and are random reads and writes. The characteristics of both can be comprehensively considered to decide which disk to place the redo on. 

- Pay attention to whether the size of the redo block size is appropriate.

  The recommended value is 512, which can be adjusted as appropriate based on the test results. When disk I/O becomes a bottleneck, this value will have a certain degree of impact.


- Pay attention to the DBWR_COUNT configuration

  DBWR_COUNT refers to the number of threads that actually perform the dirty page flushing operation. Reasonable configuration of this parameter can give full play to the disk IO performance and complete the dirty page flushing work in a timely manner. However, it's not that the faster the dirty pages are flushed, the better. Flushing dirty pages too quickly will preempt the IO resources in the write scenario. In addition, if the dbwr threads are mostly performing the checkpoint operation, they cannot clean the buffers in the Data Buffer in a timely manner. This may lead to a large number of read IOs occurring simultaneously, and thus make read IOs become a performance bottleneck in the write scenario.

  The configurations recommended from the perspective of stability are as follows:

  - Priority on stability: DBWR_COUNT = 1

  - Pursuing extreme performance (but there may be significant performance fluctuations, and in high - concurrency scenarios, performance degradation is likely to occur due to read IO preemption): DBWR_COUNT = 8

- Pay attention to whether the checkpoint strategy is appropriate.

  If there is no need for high-availability switching and other fault - recovery requirements, the following configurations are recommended:

  |Related Parameter |Configuration Suggestion |Configuration Explanation |
  | ----------- | -------- | -------- |
  | CHECKPOINT_INTERVAL | Half of the total size of the redo file | Allocate IO resources to the buffer cleaning operation to release a larger cache space as a priority. |
  | CHECKPOINT_TIMEOUT |  120  | Ensure the timely update of checkpoint to prevent redo from catching up. |

### Wait Event Monitoring

In YashanDB, V$SYSTEM_EVENT/GV$SYSTEM_EVENT views display current system wait event information. Analyzing wait events can locate database performance bottlenecks.

Query current wait events:

```sql
-- View current system wait events
SELECT event, wait_time, wait_count FROM V$SYSTEM_EVENT ORDER BY wait_time DESC;

-- View global wait events
SELECT inst_id, event, wait_time, wait_count FROM GV$SYSTEM_EVENT ORDER BY wait_time DESC;
```

Common wait events and their handling methods are as follows:

| Wait Event | Possible Cause | Solutions |
| --- | --- | --- |
| extending data file | Triggered tablespace auto-extension | According to actual needs, pre-create data file size |
| undo segment extending | Triggered undo file auto-extension | alter database datafile 'undo' resize 10G; |
| swap extending data file | Triggered swap file auto-extension | ALTER DATABASE TEMPFILE 'swap' RESIZE 6G; |
| xslot busy wait | initrans set too small during table creation, no available xslot on block for transaction, need to apply for wait | Actual verification shows initrans=2 performs better than initrans=128 |
| log file parallel write | Redo flush wait | Reduce redo BLOCKSIZE, mostly IO bottleneck, can separate redo and data files |
| log file sync | Waiting for LGWR process to write REDO logs to disk when transaction commits | Increase REDO_BUFFER_SIZE, increase REDO_BUFFER_PARTS partitions |
| db file scattered read | Data block physical read wait event | Verify disk IO performance, optimize SQL statements to add indexes |
| db file sequential read | Data block sequential read wait event, usually occurs during index scans | Increase data buffer size, optimize indexes |
| free buffer wait | Insufficient data buffer, waiting for free buffer | Increase DATA_BUFFER_SIZE, increase DBWR_COUNT |
| buffer busy wait | Hotspot block access wait | Adjust PCTFREE parameter, use partitioned tables |
| Redo tailgating | redo status are all active and current | Increase redo number |
| redo remote send | Redo send wait | Increase the value of the `_REPLICATION_BUFFER_SIZE` parameter |
| redo remote sync complete | Redo waiting for standby sync | Increase the value of the `RECOVERY_PARALLELISM` parameter |

### AWR Analysis

Use AWR for performance analysis:

```sql
-- Create snapshot
EXEC DBMS_AWR.CREATE_SNAPSHOT();

-- Execute target scenario

-- Create snapshot again
EXEC DBMS_AWR.CREATE_SNAPSHOT();

-- Query snapshot ID
SELECT * from sys.wrm$_snapshot ORDER BY snap_id DESC limit 2;

-- Generate AWR report
set serveroutput on;
EXEC DBMS_AWR.AWR_REPORT(DBID,SNAP_LEVEL,SNAP_ID1,SNAP_ID2);
```

### Redo Log Monitoring and Switching

```sql
-- View REDO log generation
SELECT name, value FROM V$SYSSTAT WHERE name LIKE '%REDO%';

-- View log write waits
SELECT event, time_waited, total_waits FROM V$SYSTEM_EVENT WHERE event LIKE '%log%';
```

When repeating tests, it is recommended to manually execute checkpoint and switch redo:

```sql
-- Write the dirty data in memory to the disk
ALTER SYSTEM CHECKPOINT;

-- Archive the currently writing redo log and switch to the next available redo log file at the same time
ALTER SYSTEM ARCHIVE LOG CURRENT; 

-- Force a switch of the currently writing redo log
ALTER SYSTEM SWITCH LOGFILE; 
```

### Session Monitoring

```sql
-- View current active sessions
SELECT sid, serial#, username, status, sql_id FROM V$SESSION WHERE status = 'ACTIVE';

-- View session wait events
SELECT s.sid, s.username, e.wait_event 
FROM V$SESSION s, V$SESSION_WAIT e 
WHERE s.sid = e.sid AND s.status = 'ACTIVE';

-- View SQL execution statistics
SELECT sql_id, executions, rows_processed, cpu_time, elapsed_time 
FROM V$SQLAREA
ORDER BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;
```

### Cache Hit Rate Monitoring

```sql
-- View data buffer hit rate
SELECT name, value FROM V$SYSSTAT WHERE name LIKE '%buffer%';

-- View specific hit rate calculation
SELECT
    CASE WHEN (a.value + b.value) = 0 THEN 0
    ELSE ROUND(a.value / (a.value + b.value) * 100, 2)
    END AS hit_rate
FROM
    (SELECT value FROM V$SYSSTAT WHERE name = 'consistent gets') a,
    (SELECT value FROM V$SYSSTAT WHERE name = 'physical reads') b;
```

Key metrics:

- **consistent gets**: Consistent read count

- **physical reads**: Physical read count

- **buffer hit rate**: Buffer hit rate, should be kept above 95%

## Sysbench Built-in Statistics

Sysbench outputs statistics in real-time during testing. You can control the output interval using the --report-interval parameter:

```bash
# Output statistics every 10 seconds
/home/yashan/sysbench/src/sysbench /home/yashan/sysbench/src/lua/oltp_read_write.lua \
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

Key output metrics:

- **transactions**: Total number of executed transactions

- **transactions/sec**: Transactions per second (TPS)

- **queries**: Total number of executed queries

- **queries/sec**: Queries per second (QPS)

- **avg latency**: Average response time

- **95th/99th percentile**: Percentile response time

## Common Problems and Solutions

### 1. Unique Index Conflict

- **Problem**: Unique index conflict causes test failure

- **Solution**: Adjust configuration parameter `_CONSISTENT_WRITE`, modify row movement.

### 2. Sysbench Startup Timeout Under High Concurrency

- **Problem**: Sysbench startup timeout under high concurrency scenarios

- **Solution**: Consider setting timeout parameter.

### 3. Performance Degradation Under High Concurrency

- **Problem**: Performance degradation under high concurrency in sysbench

- **Solution**: Check `max_reactor_channels` and `max_workers` parameter configuration, use these two parameters to limit the actual database concurrency.

### 4. High Read in write_only Scenario

- **Problem**: Read is particularly high in write_only scenario

- **Problem Analysis**: This situation may be somewhat reasonable. Through calculation, in a random scenario, the amount of read I/O may reach 20 times that of write I/O.