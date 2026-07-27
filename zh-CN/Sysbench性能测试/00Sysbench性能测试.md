Sysbench是一款开源的基准测试工具，主要用于评估操作系统和数据库在各种工作负载下的性能表现。Sysbench支持多种测试模式，包括CPU、内存、文件I/O、线程、Mutex以及数据库OLTP性能测试，是数据库性能测试领域最常用的工具之一。

Sysbench的数据库测试模块通过模拟真实的OLTP事务场景，测试数据库的并发处理能力、事务吞吐量和响应时间。测试结果能够直观反映数据库在高并发读写场景下的性能表现。

## 常见Sysbench测试场景

本章节将介绍YashanDB在常见场景中的Sysbench性能指标、操作步骤和调优思路。

| 常见场景| 描述 | 考察重点|
| --- | --- | --- |
| 只读（read_only） | 纯只读压测，由主键点查、索引点查、区间查询、聚合与分页排序组成，无任何DML写入操作。 | 索引读取效率、缓存命中率、查询优化器与并发读性能 |
| 读写混合（read_write） | 复用全套只读SQL，并新增了索引列更新、普通列更新、删除、插入四类DML，构成读多写少的标准OLTP混合负载。 | 索引维护开销、锁机制、事务回滚段、日志落盘与读写并发争抢能力 |

Sysbench支持的其他场景可参考其官方介绍，本文不作赘述。

## Sysbench测试中各类操作的特点

### SELECT

|操作类型 |SQL示例 |说明 |
|---|---|---|
|主键点查询|SELECT c FROM sbtestX WHERE id = ?|最基础高效的查询，TPS/QPS通常最高|
|简单范围查询|SELECT c FROM ... WHERE id BETWEEN ? AND ?  |考察范围扫描能力|
|汇总查询|SELECT SUM(k) FROM ... WHERE id BETWEEN ? AND ?|聚合计算能力|
|排序查询|SELECT c FROM ... WHERE id BETWEEN ? AND ? ORDER BY c|排序能力|
|去重查询|SELECT DISTINCT c FROM ... WHERE id BETWEEN ? AND ? ORDER BY c|去重排序能力|
|随机点查询|SELECT ... WHERE id IN (?, ?, ...)|模拟不可预测的负载|
|随机范围查询|SELECT ... WHERE id BETWEEN ? AND ?|随机范围扫描|

### UPDATE

|操作类型 |SQL示例 |说明 |
|---|---|---|
|索引更新|UPDATE sbtestX SET k = k + 1 WHERE id = ?|更新k索引列，需同时维护二级索引，成本较高，用于考察更新索引列时的锁竞争、B+树调整和事务日志开销|
|非索引更新|UPDATE sbtestX SET c = ? WHERE id = ?|更新非索引列c，不涉及索引维护，成本较低，用于考察非索引更新的基础写入性能|

### INSERT与DELETE

|操作类型 |SQL示例 |说明 |
|---|---|---|
|插入|INSERT INTO sbtestX (id, k, c, pad) VALUES (?, ?, ?, ?)|插入新记录|
|删除|DELETE FROM sbtestX WHERE id = ?|删除记录|

在读写混合（read_write）等场景中，通常配对为“先删后插”的原子操作，模拟数据变更并确保后续一致性。

## Sysbench三段式测试流程

Sysbench遵循标准的三段式测试流程，每个阶段对应不同的command参数：

|阶段|说明|命令示例|
|---|---|---|
|prepare|准备阶段，创建测试表并填充初始数据| `sysbench ... prepare` |
|run|运行阶段，执行实际的性能压测| `sysbench ... run` |
|cleanup|清理阶段，删除测试过程中创建的所有表和数据| `sysbench ... cleanup` |

## 测试结果解读

Sysbench测试完成后，会输出详细的性能测试结果。

### 测试结果简介

示例结果如下：

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

测试结果中的关键指标含义如下：

| 指标 | 含义 |
| --- | --- |
| transactions | 事务总数，测试期间完成的事务数量 |
| transactions/sec | 每秒事务数（TPS），衡量系统吞吐量，数值越高表明数据库性能越佳 |
| queries | 查询总数，测试期间执行的查询数量 |
| queries/sec | 每秒查询数（QPS），数值越高表明数据库性能越佳 |
| latency | 事务平均响应时间（毫秒） |
| latency min | 最小响应时间 |
| latency max | 最大响应时间 |
| latency avg | 平均响应时间 |
| latency percentile95 | 95%分位响应时间 |
| latency percentile99 | 99%分位响应时间 |
| threads_avg | 平均活动线程数 |
| threads | 并发线程数 |

> **Note**:
>
> P95/P99百分位延迟的参考价值远超平均值，其能够揭示绝大多数请求的真实体验，防止少数极端长尾请求掩盖问题。

### 核心指标

Sysbench测试结果中需要重点关注以下四个核心指标：

|指标| 定义| 计算方式| 意义| 优化方向|
|---|---|---|---|---|
|TPS（Transactions Per Second）| 每秒完成的事务数量 |总事务数 / 测试总时间（秒）|反映数据库综合事务处理能力，是OLTP性能的核心指标|数值越高越好|
|QPS（Queries Per Second）| 每秒执行的SQL语句数量|总SQL语句数 / 测试总时间（秒）|反映数据库纯SQL执行能力，不涉及事务提交开销|数值越高越好|
|平均延迟（ms）| 所有事务响应时间的算术平均值 | 所有事务响应时间之和 / 事务总数 | 反映系统整体响应性能的平滑指标 | 数值越低越好 |
|P95延迟（ms）| 95%的事务响应时间低于此数值 | 将所有事务响应时间排序，取第95%分位值|反映大部分用户的实际体验，比平均延迟更能体现长尾延迟问题 | 数值越低越好|

针对不同场景，上述关键指标的侧重点/组合可能有所不同，例如：

| 场景| 重点指标| 说明|
|---|---|---|
|性能对比测试| TPS | 吞吐量是核心对比指标 |
|用户体验评估| P95延迟 | 反映长尾延迟对用户的影响 |
|系统容量规划| TPS + QPS | 评估系统承载能力 |
|稳定性测试| 平均延迟 + P95延迟 | 评估性能稳定性 |
