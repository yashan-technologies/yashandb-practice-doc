This article will introduce the key steps for tuning the TPC-C test applicable to the Kunpeng environment, including tuning at the database layer and the database operating environment.

## Environment Preparation

- Hardware environment information is as follows:
    - Server: 2 Kunpeng 920, 128-core CPU, 4 NUMA nodes

    - Network card: Kunpeng matching network card, 10GE

    - Hard disk: SAS SSD (NVME SSD preferred)

- Software environment information is as follows:

    - Database: YashanDB v23.3 and above

    - TPC-C client: BenchMarkSQL5.0

## Tuning Operations

### Operating System Tuning

In a NUMA architecture, if tuning needs to be done at the operating system level, finer control over the interrupt load can be achieved by disabling irqbalance to avoid unnecessary resource waste and delays.

```bash
systemctl stop irqbalance
```

### Network Configuration Tuning

For TPC-C stress testing, it is generally recommended to deploy the client and server on different servers to achieve better results. In this testing scenario, the network transmission between the client and the server will be a significant bottleneck. At this point, the server's network card queue interrupt handling can be bound to the CPU on the NUMA node where the network card is located to optimize network performance.

Common commands related to network configuration are as follows:

-  `ifconfig` is used to view the network card device name.
- `ethtool -i %NETWORK_CARD_NAME%` is used to view the network card bus-info.
- `lspci -vvvs %NETWORK_CARD_NAME%bus-info` is used to view the NUMA node where the network card is located.
- `ethtool -l %NETWORK_CARD_NAME%` is used to view the interrupt queue information of the network card.
- `ethtool -L %NETWORK_CARD_NAME% combined %INTERRUPT_QUEUES_NUMBER%` is used to set the number of interrupt queues for the network card.

```bash
// View the interrupt queue messages of the network card in the test environment
$ ethtool -l enp1s0f0np0
Channel parameters for enp1s0f0np0:
Pre-set maximums:
RX:             0
TX:             0
Other:          0
Combined:       63
Current hardware settings:
RX:             0
TX:             0
Other:          0
Combined:       16
// According to the echo information, the maximum number of interrupt queues in the current environment is 63, and the set number of interrupt queues is 16
```

There are 4 NUMA nodes in the test environment, and each NUMA node has 32 cores. The optimal number of interrupt queues for the network card is set to 32, and the network card interrupt queues are bound to the CPU on NUMA node0. The script is as follows; sh irqbind.sh %NETWORK CARD%bus-info binds the network card interrupt queues to CPUs 0-31.
```bash
#!/bin/bash
# irqbind.sh $1 is the network card device buf-info
irq_list=(`cat /proc/interrupts | grep $1 | awk -F: '{print $1}'`)

cpunum=0
for irq in ${irq_list[@]}
do
echo $cpunum &gt; /proc/irq/$irq/smp_affinity_list
echo `cat /proc/irq/$irq/smp_affinity_list`
(( cpunum+=1 ))
(( cpunum %= 32))
done
```

### Database Tuning

#### Database Server Core Binding

To improve the processing capability of the network module, the database server should avoid occupying the CPUs bound to the network card as much as possible; the database process can be bound to other NUMA nodes.

After binding the database process to a NUMA node, it will only be able to use the corresponding CPU and memory resources, so ensure that the corresponding resources are sufficient; otherwise, performance may decrease.

In the test environment, according to TPC-C best practices, the CPU usage of the database process is about 5000%, and the database process can be bound to the corresponding NUMA node using the following command.

```bash
numactl --cpunodebind=1,2,3 --membind=1,2,3 yasdb open
```

#### Database Configuration Parameter Tuning

The following lists the recommended database configuration parameters for TPC-C testing. Specific parameter values can be further adjusted according to the actual environment.

The following configurations are not limited to optimization on domestic chip architecture platforms; they are also applicable on general-purpose chip architecture platforms.

```bash
DATA_BUFFER_SIZE = "200G"
_DATA_BUFFER_PARTS=8
VM_BUFFER_SIZE="50G"
VM_BUFFER_PARTS=8
REDO_BUFFER_SIZE="64M"
REDO_BUFFER_PARTS = 8
LARGE_POOL_SIZE="2G"
UNDO_RETENTION=10
UNDO_SHRINK_ENABLED="FALSE"
_SESSION_RESERVED_CURSORS=64
LOCK_POOL_SIZE="3G"
SHARE_POOL_SIZE="30G"
SQL_POOL_PARTS = 8
COMMIT_LOGGING = "BATCH"
DBWR_COUNT=8
DBWR_BUFFER_SIZE="16M"
CHECKPOINT_TIMEOUT=1000000000
CHECKPOINT_INTERVAL="200G"
```

#### Benchmark Tuning

Combine the data characteristics to reasonably use table partitioning.

The following configurations are not limited to optimization on domestic chip architecture platforms; they are also applicable on general-purpose chip architecture platforms.

```sql
create table bmsql_config
(
    cfg_name  varchar2(30) primary key,
    cfg_value varchar2(50)
) tablespace users pctfree 50
partition by hash (cfg_name) partitions 64;

create table bmsql_warehouse
(
    w_id       integer not null,
    w_ytd      number(12,2),
    w_tax      number(4,4),
    w_name     varchar2(10),
    w_street_1 varchar2(20),
    w_street_2 varchar2(20),
    w_city     varchar2(20),
    w_state    char(2),
    w_zip      char(9)
) tablespace users pctfree 50
partition by range (w_id) interval(1) (
partition values less than (1)
);

create table bmsql_district
(
    d_w_id      integer not null,
    d_id        integer not null,
    d_ytd       number(12,2),
    d_tax       number(4,4),
    d_next_o_id integer,
    d_name      varchar2(10),
    d_street_1  varchar2(20),
    d_street_2  varchar2(20),
    d_city      varchar2(20),
    d_state     char(2),
    d_zip       char(9)
) tablespace users pctfree 50
partition by range (d_w_id) interval(1) (
partition values less than (1)
);

create table bmsql_customer
(
    c_w_id         integer not null,
    c_d_id         integer not null,
    c_id           integer not null,
    c_discount     number(4,4),
    c_credit       char(2),
    c_last         varchar2(16),
    c_first        varchar2(16),
    c_credit_lim   number(12,2),
    c_balance      number(12,2),
    c_ytd_payment  number(12,2),
    c_payment_cnt  integer,
    c_delivery_cnt integer,
    c_street_1     varchar2(20),
    c_street_2     varchar2(20),
    c_city         varchar2(20),
    c_state        char(2),
    c_zip          char(9),
    c_phone        char(16),
    c_since        timestamp,
    c_middle       char(2),
    c_data         varchar2(500)
) tablespace users pctfree 50
partition by range (c_w_id) interval(1) (
partition values less than (1)
);

create table bmsql_history
(
    hist_id  integer,
    h_c_id   integer,
    h_c_d_id integer,
    h_c_w_id integer,
    h_d_id   integer,
    h_w_id   integer,
    h_date   timestamp,
    h_amount number(6,2),
    h_data   varchar2(24)
) tablespace users pctfree 50
partition by range (h_w_id) interval(1) (
partition values less than (1)
);

create table bmsql_new_order
(
    no_w_id integer not null,
    no_d_id integer not null,
    no_o_id integer not null
) tablespace users pctfree 50
partition by range (no_w_id) interval(1) (
partition values less than (1)
);

create table bmsql_oorder
(
    o_w_id       integer not null,
    o_d_id       integer not null,
    o_id         integer not null,
    o_c_id       integer,
    o_carrier_id integer,
    o_ol_cnt     integer,
    o_all_local  integer,
    o_entry_d    timestamp
) tablespace users pctfree 50
partition by range (o_w_id) interval(1) (
partition values less than (1)
);

create table bmsql_order_line
(
    ol_w_id        integer not null,
    ol_d_id        integer not null,
    ol_o_id        integer not null,
    ol_number      integer not null,
    ol_i_id        integer not null,
    ol_delivery_d  timestamp,
    ol_amount      number(6,2),
    ol_supply_w_id integer,
    ol_quantity    integer,
    ol_dist_info   char(24)
) tablespace users pctfree 50
partition by range (ol_w_id) interval(1) (
partition values less than (1)
);

create table bmsql_item
(
    i_id    integer not null,
    i_name  varchar2(24),
    i_price number(5,2),
    i_data  varchar2(50),
    i_im_id integer
) tablespace users pctfree 50
partition by hash (i_id) partitions 64;

create table bmsql_stock
(
    s_w_id       integer not null,
    s_i_id       integer not null,
    s_quantity   integer,
    s_ytd        integer,
    s_order_cnt  integer,
    s_remote_cnt integer,
    s_data       varchar2(50),
    s_dist_01    char(24),
    s_dist_02    char(24),
    s_dist_03    char(24),
    s_dist_04    char(24),
    s_dist_05    char(24),
    s_dist_06    char(24),
    s_dist_07    char(24),
    s_dist_08    char(24),
    s_dist_09    char(24),
    s_dist_10    char(24)
) tablespace users pctfree 50
partition by range (s_w_id) interval(1) (
partition values less than (1)
);

alter table bmsql_warehouse add constraint bmsql_warehouse_pkey
    primary key (w_id) using index local;

alter table bmsql_district add constraint bmsql_district_pkey
    primary key (d_w_id, d_id) using index local;

alter table bmsql_customer add constraint bmsql_customer_pkey
    primary key (c_w_id, c_d_id, c_id) using index local;

create index bmsql_customer_idx1
  on  bmsql_customer (c_w_id, c_d_id, c_last, c_first) local tablespace users pctfree 50;

alter table bmsql_oorder add constraint bmsql_oorder_pkey
    primary key (o_w_id, o_d_id, o_id) using index local;

alter table bmsql_new_order add constraint bmsql_new_order_pkey
    primary key (no_w_id, no_d_id, no_o_id) using index local;

alter table bmsql_order_line add constraint bmsql_order_line_pkey
    primary key (ol_w_id, ol_d_id, ol_o_id, ol_number) using index local;

create unique index bmsql_stock_idx1 on bmsql_stock(s_w_id, s_i_id) local tablespace users pctfree 50;

alter table bmsql_stock add constraint bmsql_stock_pkey
    primary key (s_w_id, s_i_id) using index local;

alter table bmsql_item add constraint bmsql_item_pkey
    primary key (i_id) using index local;
```

#### Database File Partition Storage

Based on the read and write frequency characteristics of various data files, rationally plan their storage locations. For example, store the database redo files separately from other files on different SSDs to ensure that the disk IO of the redo files is not contended by other files.