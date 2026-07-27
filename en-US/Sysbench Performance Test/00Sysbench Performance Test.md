Sysbench is an open-source benchmark tool primarily used to evaluate the performance of operating systems and databases under various workloads. Sysbench supports multiple test modes, including CPU, memory, file I/O, thread, mutex, and database OLTP performance testing, making it one of the most commonly used tools in database performance testing.

The database test module of Sysbench simulates real OLTP transaction scenarios to test the database's concurrent processing capability, transaction throughput, and response time. The test results can directly reflect the database's performance under high-concurrency read/write scenarios.

## Common Sysbench Test Scenarios

This chapter will introduce Sysbench performance metrics, operation steps, and optimization strategies for YashanDB under common scenarios.

|Common Scenario | Description |Focus |
| --- | --- | --- |
| read_only | Pure read-only stress testing, consisting of primary key point queries, index point queries, range queries, aggregation and pagination sorting, with no DML write operations. | Index read efficiency, cache hit rate, query optimizer, and concurrent read performance |
| read_write | Reuses the full set of read-only SQL and adds four types of DML: index column updates, regular column updates, deletes, and inserts, forming a standard OLTP mixed workload with more reads than writes. | Index maintenance overhead, lock mechanisms, transaction rollback segments, log persistence, and concurrent read-write contention capability |

Other scenarios supported by Sysbench can be referred to in its official documentation and are not described in this document.

## Characteristics of Various Types of Operations in Sysbench Testing

### SELECT

| Operation Type| SQL Example| Description|
|---|---|---|
|Primary Key Point Query|SELECT c FROM sbtestX WHERE id = ?|Most basic and efficient query, TPS/QPS usually highest|
|Simple Range Query|SELECT c FROM ... WHERE id BETWEEN ? AND ?|Examine range scan capability|
|Aggregation Query|SELECT SUM(k) FROM ... WHERE id BETWEEN ? AND ?|Aggregation calculation capability|
|Sorted Query|SELECT c FROM ... WHERE id BETWEEN ? AND ? ORDER BY c|Sorting capability|
|Distinct Query|SELECT DISTINCT c FROM ... WHERE id BETWEEN ? AND ? ORDER BY c|Distinct sorting capability|
|Random Point Query|SELECT ... WHERE id IN (?, ?, ...)|Simulate unpredictable loads|
|Random Range Query|SELECT ... WHERE id BETWEEN ? AND ?|Random range scan|

### UPDATE

| Operation Type| SQL Example| Description|
|---|---|---|
|Index Update|UPDATE sbtestX SET k = k + 1 WHERE id = ?|Update k index column, need to maintain secondary index simultaneously, higher cost, used to examine lock contention, B+ tree adjustment and transaction log overhead when updating index column|
|Non-Index Update|UPDATE sbtestX SET c = ? WHERE id = ?|Update non-index column c, no index maintenance involved, lower cost, used to examine basic write performance of non-index updates|

### INSERT and DELETE

| Operation Type| SQL Example| Description|
|---|---|---|
|Insert|INSERT INTO sbtestX (id, k, c, pad) VALUES (?, ?, ?, ?)|Insert new record|
|Delete|DELETE FROM sbtestX WHERE id = ?|Delete record|

In read_write scenarios, usually paired as "delete-then-insert" atomic operations, simulating data changes and ensuring subsequent consistency.

## Sysbench Three-Phase Test Process

Sysbench follows a standard three-phase test process, with each phase corresponding to different command parameters:

|Phase|Description|Command Example|
|---|---|---|
|prepare|Prepare phase, create test tables and populate initial data| `sysbench ... prepare` |
|run|Run phase, execute actual performance stress test| `sysbench ... run` |
|cleanup|Cleanup phase, delete all tables and data created during testing| `sysbench ... cleanup` |

## Key Metrics in Test Results

After the Sysbench test is completed, detailed performance test results will be output. 

### Introduction to Test Results

The sample results are as follows:

```bash
SQL statistics:
    queries performed:
        read:                            854280
        write:                           121680
        other:                           60840
        total:                           1036800
    transactions:                         30400  (50.64 per sec.)
    queries:                              1036800 (1728.31 per sec.)
    ignored errors:                       0
    reconnects:                          0

General statistics:
    total time:                          600.0004s
    total number of events:              30400

Latency (ms):
         min:                                  1.52
         avg:                                  5.04
         max:                                123.45
         95th percentile:                      8.12
         99th percentile:                     12.34
         sum:                              153216.00

Threads fairness:
    events (avg/stddev):           30400.0000/0.00
    execution time (avg/stddev):        5.0438/0.00
```

The key metrics in the test results are as follows:

| Metric | Meaning |
| --- | --- |
| transactions | Total number of transactions completed during the test |
| transactions/sec | Transactions per Second (TPS), which measures the system throughput. The higher the value, the better the performance of the database |
| queries | Total number of queries executed during the test |
| queries/sec | Queries per Second (QPS). The higher the value, the better the database performance |
| latency | Average transaction response time (milliseconds) |
| latency min | Minimum response time |
| latency max | Maximum response time |
| latency avg | Average response time |
| latency percentile95 | 95th percentile response time |
| latency percentile99 | 99th percentile response time |
| threads_avg | Average active threads |
| threads | Number of concurrent threads |

> **Note**:
>
> P95/P99 percentile latency provides far greater insight than average values, revealing the true experience of the vast majority of requests and preventing extreme long-tail requests from masking issues.

### Core Indicators

In the Sysbench test results, the following four core indicators need to be focused on:

|Metric |Definition |Calculation |Significance |Optimization |
|---|---|---|---|---|
|TPS (Transactions Per Second)|The number of transactions completed per second|Total number of transactions / Total test time (seconds)|Reflects the comprehensive transaction processing capability of the database and is a core indicator of OLTP performance|The higher the value, the better|
|QPS (Queries Per Second) | The number of SQL statements executed per second|Total number of SQL statements / Total test time (seconds)|Reflects the pure SQL execution ability of the database without involving transaction commit overhead|The higher the value, the better|
| Average Latency (ms) | The arithmetic mean of the response times of all transactions | Sum of the response times of all transactions / Total number of transactions | A smooth metric reflecting the overall response performance of the system | The lower the value, the better |
| P95 Latency (ms) | The value below which 95% of the transaction response times fall | Sort all transaction response times and take the 95th percentile value | Reflects the actual experience of most users and can better illustrate the long - tail latency issue than average latency | The lower the value, the better |

For different scenarios, the emphasis/combination on the above-mentioned key indicators may vary. For example:

|Scenario |Key Metrics |Description |
|---|---|---|
|Performance comparison test|TPS|Throughput is the core comparison metric|
|User experience evaluation|P95 latency|Reflects impact of tail latency on users|
|System capacity planning|TPS + QPS|Evaluate system bearing capacity|
|Stability test|Average latency + P95 latency|Evaluate performance stability|