---
layout: post
title: "Apache Doris 数据导出"
date: 2021-10-13
description: "Apache Doris 数据导出"
tag: Apache Doris
---
#  Apache Doris 数据导出

## 1.概述

Apache Doris为了方便用将Doris的数据导出到其他系统， 提供了两种将数据导出的方式：

1. Export 方式：

   Export 是 Doris 提供的一种将数据导出的功能。该功能可以将用户指定的表或分区的数据，以文本的格式，通过 Broker 进程导出到远端存储上，如 HDFS/BOS 等。

2. 查询结果集导出方式：

   查询结果集的导出是使用 `SELECT INTO OUTFILE` 命令进行查询结果的导出操作。

下面我们分别介绍两种方式使用场景及使用方式

## 2. Export 导出方式

Export方式是用户提交一个 Export 作业后。Doris 会统计这个作业涉及的所有 Tablet。然后对这些 Tablet 进行分组，每组生成一个特殊的查询计划。该查询计划会读取所包含的 Tablet 上的数据，然后通过 Broker 将数据写到远端存储指定的路径中，也可以通过S3协议直接导出到支持S3协议的远端存储上，同时也可以将数据导出到本地文件系统（导出到本地文件系统在0.15版本支持）。

Export **只支持单表**数据导出，支持导出该表的指定字段（0.14之前的版本不支持导出指定字段功能）。

### 2.1 Export 任务调度流程

1. 用户提交一个 Export 作业到 FE。
2. FE 的 Export 调度器会通过两阶段来执行一个 Export 作业：
   1. PENDING：FE 生成 ExportPendingTask，向 BE 发送 snapshot 命令，对所有涉及到的 Tablet 做一个快照。并生成多个查询计划。
   2. EXPORTING：FE 生成 ExportExportingTask，开始执行查询计划。



### 2.2 Export 查询计划

#### 2.2.1 查询计划拆分

Export 作业会生成多个查询计划，每个查询计划负责扫描一部分 Tablet。每个查询计划扫描的 Tablet 个数由 FE 配置参数 `export_tablet_num_per_task` 指定，默认为 5。即假设一共 100 个 Tablet，则会生成 20 个查询计划。用户也可以在提交作业时，通过作业属性 `tablet_num_per_task` 指定这个数值。

#### 2.2.2 查询计划执行

一个查询计划扫描多个分片，将读取的数据以行的形式组织，每 1024 行为一个 batch，调用 Broker 写入到远端存储上。

查询计划遇到错误会整体自动重试 3 次。如果一个查询计划重试 3 次依然失败，则整个作业失败。

Doris 会首先在指定的远端存储的路径中，建立一个名为 `__doris_export_tmp_12345` 的临时目录（其中 `12345` 为作业 id）。导出的数据首先会写入这个临时目录。每个查询计划会生成一个文件，文件名示例：

```
export-data-c69fcf2b6db5420f-a96b94c1ff8bccef-1561453713822
```

其中 `c69fcf2b6db5420f-a96b94c1ff8bccef` 为查询计划的 query id。`1561453713822` 为文件生成的时间戳。

当所有数据都导出后，Doris 会将这些文件 rename 到用户指定的路径中。

### 2.3 使用实例

Export 的详细命令可以通过 `HELP EXPORT;` 。举例如下：

```sql
EXPORT TABLE db1.tbl1 
PARTITION (p1,p2)
[WHERE [expr]]
TO "hdfs://host/path/to/export/" 
PROPERTIES
(
	"label" = "mylabel",
    "column_separator"=",",
    "columns" = "col1,col2",
    "exec_mem_limit"="2147483648",
    "timeout" = "3600"
)
WITH BROKER "hdfs"
(
	"username" = "user",
	"password" = "passwd"
);
```

>注意：
>
>1. 在0.15版本支持将数据导出到本地，如果需要导出到本地需要在 fe.conf 中设置` enable_outfile_to_local=true`，否则是不支持到本地的

- `label`：本次导出作业的标识。后续可以使用这个标识查看作业状态。

- column: 指定待导出的列，使用英文逗号隔开，如果不填这个参数默认是导出表的所有列。

- `column_separator`：列分隔符。默认为 `\t`。支持不可见字符，比如 '\x07'。

- columns：要导出的列，使用英文状态逗号隔开，如果不填这个参数默认是导出表的所有列

- `line_delimiter`：行分隔符。默认为 `\n`。支持不可见字符，比如 '\x07'。

- `exec_mem_limit`： 表示 Export 作业中，一个查询计划在单个 BE 上的内存使用限制。默认 2GB。单位字节。

- `timeout`：作业超时时间。默认 2小时。单位秒。

- `tablet_num_per_task`：每个查询计划分配的最大分片数。默认为 5。

  

提交作业后，可以通过 `SHOW EXPORT` 命令查询导入作业状态。结果举例如下：

```text
     JobId: 14008
     Label: mylabel
     State: FINISHED
  Progress: 100%
  TaskInfo: {"partitions":["*"],"exec mem limit":2147483648,"column separator":",","line delimiter":"\n","tablet num":1,"broker":"hdfs","coord num":1,"db":"default_cluster:db1","tbl":"tbl3"}
      Path: bos://bj-test-cmy/export/
CreateTime: 2019-06-25 17:08:24
 StartTime: 2019-06-25 17:08:28
FinishTime: 2019-06-25 17:08:34
   Timeout: 3600
  ErrorMsg: N/A
```

- JobId：作业的唯一 ID
- Label：自定义作业标识
- State：作业状态：
  - PENDING：作业待调度
  - EXPORTING：数据导出中
  - FINISHED：作业成功
  - CANCELLED：作业失败
- Progress：作业进度。该进度以查询计划为单位。假设一共 10 个查询计划，当前已完成 3 个，则进度为 30%。
- TaskInfo：以 Json 格式展示的作业信息：
  - db：数据库名
  - tbl：表名
  - partitions：指定导出的分区。`*` 表示所有分区。
  - exec mem limit：查询计划内存使用限制。单位字节。
  - column separator：导出文件的列分隔符。
  - line delimiter：导出文件的行分隔符。
  - tablet num：涉及的总 Tablet 数量。
  - broker：使用的 broker 的名称。
  - coord num：查询计划的个数。
- Path：远端存储上的导出路径。
- CreateTime/StartTime/FinishTime：作业的创建时间、开始调度时间和结束时间。
- Timeout：作业超时时间。单位是秒。该时间从 CreateTime 开始计算。
- ErrorMsg：如果作业出现错误，这里会显示错误原因

### 2.4 最佳实践

#### 2.4.1 查询计划拆分

一个 Export 作业有多少查询计划需要执行，取决于总共有多少 Tablet，以及一个查询计划最多可以分配多少个 Tablet。因为多个查询计划是串行执行的，所以如果让一个查询计划处理更多的分片，则可以减少作业的执行时间。但如果查询计划出错（比如调用 Broker 的 RPC 失败，远端存储出现抖动等），过多的 Tablet 会导致一个查询计划的重试成本变高。所以需要合理安排查询计划的个数以及每个查询计划所需要扫描的分片数，在执行时间和执行成功率之间做出平衡。一般建议一个查询计划扫描的数据量在 3-5 GB内（一个表的 Tablet 的大小以及个数可以通过 `SHOW TABLET FROM tbl_name;` 语句查看。）。

#### 2.4.2 exec_mem_limit

通常一个 Export 作业的查询计划只有 `扫描`-`导出` 两部分，不涉及需要太多内存的计算逻辑。所以通常 2GB 的默认内存限制可以满足需求。但在某些场景下，比如一个查询计划，在同一个 BE 上需要扫描的 Tablet 过多，或者 Tablet 的数据版本过多时，可能会导致内存不足。此时需要通过这个参数设置更大的内存，比如 4GB、8GB 等

### 2.5 使用注意事项

- 不建议一次性导出大量数据。一个 Export 作业建议的导出数据量最大在几十 GB。过大的导出会导致更多的垃圾文件和更高的重试成本。
- 如果表数据量过大，建议按照分区导出。
- 在 Export 作业运行过程中，如果 FE 发生重启或切主，则 Export 作业会失败，需要用户重新提交。
- 如果 Export 作业运行失败，在远端存储中产生的 `__doris_export_tmp_xxx` 临时目录，以及已经生成的文件不会被删除，需要用户手动删除。
- 如果 Export 作业运行成功，在远端存储中产生的 `__doris_export_tmp_xxx` 目录，根据远端存储的文件系统语义，可能会保留，也可能会被清除。比如在百度对象存储（BOS）中，通过 rename 操作将一个目录中的最后一个文件移走后，该目录也会被删除。如果该目录没有被清除，用户可以手动清除。
- 当 Export 运行完成后（成功或失败），FE 发生重启或切主，则 `SHOW EXPORT` 展示的作业的部分信息会丢失，无法查看。
- Export 作业只会导出 Base 表的数据，不会导出 Rollup Index 的数据。
- Export 作业会扫描数据，占用 IO 资源，可能会影响系统的查询延迟

### 2.6 FE 相关参数配置

- `export_checker_interval_second`：Export 作业调度器的调度间隔，默认为 5 秒。设置该参数需重启 FE。
- `export_running_job_num_limit`：正在运行的 Export 作业数量限制。如果超过，则作业将等待并处于 PENDING 状态。默认为 5，可以运行时调整。
- `export_task_default_timeout_second`：Export 作业默认超时时间。默认为 2 小时。可以运行时调整。根据自己任务的预估运算实践进行调整
- `export_tablet_num_per_task`：一个查询计划负责的最大分片数。默认为 5。不建议每个拆分的查询计划执行太多的分片数据量。

## 3.查询结果导出

相对Export 导出，查询结果结果的导出方式更灵活，可以支持更复杂的SQL，Join，Union等方式。

`SELECT INTO OUTFILE` 语句可以将查询结果导出到文件中。目前支持通过 Broker 进程, 通过 S3 协议, 或直接通过 HDFS 协议，导出到远端存储，如 HDFS，S3，BOS，COS（腾讯云）

### 3.1 导出语法

```sql
query_stmt
INTO OUTFILE "file_path"
[format_as]
[properties]
```

- `file_path`

  `file_path` 指向文件存储的路径以及文件前缀。如 `hdfs://path/to/my_file_`。

  最终的文件名将由 `my_file_`，文件序号以及文件格式后缀组成。其中文件序号由0开始，数量为文件被分割的数量。如：

  ```text
  my_file_abcdefg_0.csv
  my_file_abcdefg_1.csv
  my_file_abcdegf_2.csv
  ```

- `[format_as]`

  ```text
  FORMAT AS CSV
  ```

  指定导出格式。默认为 CSV。

- `[properties]`

  指定相关属性。目前支持通过 Broker 进程, 或通过 S3 协议进行导出。

  - Broker 相关属性需加前缀 `broker.`。具体参阅 Broker 文档。
  - HDFS 相关属性需加前缀 `hdfs.`。
  - S3 协议则直接执行 S3 协议配置即可。

  ```text
  ("broker.prop_key" = "broker.prop_val", ...)
  or
  ("hdfs.fs.defaultFS" = "xxx", "hdfs.hdfs_user" = "xxx")
  or 
  ("AWS_ENDPOINT" = "xxx", ...)
  ```

  其他属性：

  ```text
  ("key1" = "val1", "key2" = "val2", ...)
  ```

  目前支持以下属性：

  - `column_separator`：列分隔符，仅对 CSV 格式适用。默认为 `\t`。
  - `line_delimiter`：行分隔符，仅对 CSV 格式适用。默认为 `\n`。
  - `max_file_size`：单个文件的最大大小。默认为 1GB。取值范围在 5MB 到 2GB 之间。超过这个大小的文件将会被切分。
  - `schema`：PARQUET 文件schema信息。仅对 PARQUET 格式适用。导出文件格式为PARQUET时，必须指定`schema`

#### 3.1.1 并发导出

  默认情况下，查询结果集的导出是非并发的，也就是单点导出。如果用户希望查询结果集可以并发导出，需要满足以下条件：

  1. session variable 'enable_parallel_outfile' 开启并发导出: `set enable_parallel_outfile = true;`
  2. 导出方式为 S3 , 或者 HDFS， 而不是使用 broker
  3. 查询可以满足并发导出的需求，比如顶层不包含 sort 等单点节点。（后面会举例说明，哪种属于不可并发导出结果集的查询）

  满足以上三个条件，就能触发并发导出查询结果集了。并发度 = `be_instacne_num * parallel_fragment_exec_instance_num`

#### 3.1.2 如何验证结果集被并发导出

  用户通过 session 变量设置开启并发导出后，如果想验证当前查询是否能进行并发导出，则可以通过下面这个方法。

  ```text
  explain select xxx from xxx where xxx  into outfile "s3://xxx" format as csv properties ("AWS_ENDPOINT" = "xxx", ...);
  ```

  对查询进行 explain 后，Doris 会返回该查询的规划，如果你发现 `RESULT FILE SINK` 出现在 `PLAN FRAGMENT 1` 中，就说明导出并发开启成功了。 如果 `RESULT FILE SINK` 出现在 `PLAN FRAGMENT 0` 中，则说明当前查询不能进行并发导出 (当前查询不同时满足并发导出的三个条件)。

  ```text
  并发导出的规划示例：
  +-----------------------------------------------------------------------------+
  | Explain String                                                              |
  +-----------------------------------------------------------------------------+
  | PLAN FRAGMENT 0                                                             |
  |  OUTPUT EXPRS:<slot 2> | <slot 3> | <slot 4> | <slot 5>                     |
  |   PARTITION: UNPARTITIONED                                                  |
  |                                                                             |
  |   RESULT SINK                                                               |
  |                                                                             |
  |   1:EXCHANGE                                                                |
  |                                                                             |
  | PLAN FRAGMENT 1                                                             |
  |  OUTPUT EXPRS:`k1` + `k2`                                                   |
  |   PARTITION: HASH_PARTITIONED: `default_cluster:test`.`multi_tablet`.`k1`   |
  |                                                                             |
  |   RESULT FILE SINK                                                          |
  |   FILE PATH: s3://ml-bd-repo/bpit_test/outfile_1951_                        |
  |   STORAGE TYPE: S3                                                          |
  |                                                                             |
  |   0:OlapScanNode                                                            |
  |      TABLE: multi_tablet                                                    |
  +-----------------------------------------------------------------------------+
  ```

### 3.2 使用示例

#### 3.2.1 使用Broker方式导出

##### 示例 1 ：

使用 broker 方式导出，将简单查询结果导出到文件 `hdfs:/path/to/result.txt`。指定导出格式为 CSV。使用 `my_broker` 并设置 kerberos 认证信息。指定列分隔符为 `,`，行分隔符为 `\n`。

```text
SELECT * FROM tbl
INTO OUTFILE "hdfs:/path/to/result_"
FORMAT AS CSV
PROPERTIES
(
    "broker.name" = "my_broker",
    "broker.hadoop.security.authentication" = "kerberos",
    "broker.kerberos_principal" = "doris@YOUR.COM",
    "broker.kerberos_keytab" = "/home/doris/my.keytab",
    "column_separator" = ",",
    "line_delimiter" = "\n",
    "max_file_size" = "100MB"
);
```

最终生成文件如如果不大于 100MB，则为：`result_0.csv`。

如果大于 100MB，则可能为 `result_0.csv, result_1.csv, ...`

##### **示例 2**  

将简单查询结果导出到文件 `hdfs:/path/to/result.parquet`。指定导出格式为 PARQUET。使用 `my_broker` 并设置 kerberos 认证信息。

```sql
SELECT c1, c2, c3 FROM tbl
INTO OUTFILE "hdfs:/path/to/result_"
FORMAT AS PARQUET
PROPERTIES
(
    "broker.name" = "my_broker",
    "broker.hadoop.security.authentication" = "kerberos",
    "broker.kerberos_principal" = "doris@YOUR.COM",
    "broker.kerberos_keytab" = "/home/doris/my.keytab",
    "schema"="required,int32,c1;required,byte_array,c2;required,byte_array,c2"
);
```

查询结果导出到parquet文件需要明确指定`schema`。

#### 3.2.2 复杂SQL语句导出

将 CTE 语句的查询结果导出到文件 `hdfs:/path/to/result.txt`。默认导出格式为 CSV。使用 `my_broker` 并设置 hdfs 高可用信息。使用默认的行列分隔符。

```sql
WITH
x1 AS
(SELECT k1, k2 FROM tbl1),
x2 AS
(SELECT k3 FROM tbl2)
SELEC k1 FROM x1 UNION SELECT k3 FROM x2
INTO OUTFILE "hdfs:/path/to/result_"
PROPERTIES
(
    "broker.name" = "my_broker",
    "broker.username"="user",
    "broker.password"="passwd",
    "broker.dfs.nameservices" = "my_ha",
    "broker.dfs.ha.namenodes.my_ha" = "my_namenode1, my_namenode2",
    "broker.dfs.namenode.rpc-address.my_ha.my_namenode1" = "nn1_host:rpc_port",
    "broker.dfs.namenode.rpc-address.my_ha.my_namenode2" = "nn2_host:rpc_port",
    "broker.dfs.client.failover.proxy.provider" = "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider"
);
```

最终生成文件如如果不大于 1GB，则为：`result_0.csv`。

如果大于 1GB，则可能为 `result_0.csv, result_1.csv, ...`。

#### 3.2.3 直接使用Union语句导出

将 UNION 语句的查询结果导出到文件 `bos://bucket/result.txt`。指定导出格式为 PARQUET。使用 `my_broker` 并设置 hdfs 高可用信息。PARQUET 格式无需指定列分割符。 导出完成后，生成一个标识文件。

```sql
SELECT k1 FROM tbl1 UNION SELECT k2 FROM tbl1
INTO OUTFILE "bos://bucket/result_"
FORMAT AS PARQUET
PROPERTIES
(
    "broker.name" = "my_broker",
    "broker.bos_endpoint" = "http://bj.bcebos.com",
    "broker.bos_accesskey" = "xxxxxxxxxxxxxxxxxxxxxxxxxx",
    "broker.bos_secret_accesskey" = "yyyyyyyyyyyyyyyyyyyyyyyyyy",
    "schema"="required,int32,k1;required,byte_array,k2"
);
```

### 3.3 查询返回结果

导出命令为同步命令。命令返回，即表示操作结束。同时会返回一行结果来展示导出的执行结果。

如果正常导出并返回，则结果如下：

```text
mysql> select * from tbl1 limit 10 into outfile "file:///home/work/path/result_";
+------------+-----------+----------+--------------------------------------------------------------------+
| FileNumber | TotalRows | FileSize | URL                                                                |
+------------+-----------+----------+--------------------------------------------------------------------+
|          1 |         2 |        8 | file:///192.168.1.10/home/work/path/result_{fragment_instance_id}_ |
+------------+-----------+----------+--------------------------------------------------------------------+
1 row in set (0.05 sec)
```

- FileNumber：最终生成的文件个数。
- TotalRows：结果集行数。
- FileSize：导出文件总大小。单位字节。
- URL：如果是导出到本地磁盘，则这里显示具体导出到哪个 Compute Node。

如果进行了并发导出，则会返回多行数据。

```text
+------------+-----------+----------+--------------------------------------------------------------------+
| FileNumber | TotalRows | FileSize | URL                                                                |
+------------+-----------+----------+--------------------------------------------------------------------+
|          1 |         3 |        7 | file:///192.168.1.10/home/work/path/result_{fragment_instance_id}_ |
|          1 |         2 |        4 | file:///192.168.1.11/home/work/path/result_{fragment_instance_id}_ |
+------------+-----------+----------+--------------------------------------------------------------------+
2 rows in set (2.218 sec)
```

如果执行错误，则会返回错误信息，如：

```text
mysql> SELECT * FROM tbl INTO OUTFILE ...
ERROR 1064 (HY000): errCode = 2, detailMessage = Open broker writer failed ...
```

### 3.4 注意事项

- 如果不开启并发导出，查询结果是由单个 BE 节点，单线程导出的。因此导出时间和导出结果集大小正相关。开启并发导出可以降低导出的时间。
- 导出命令不会检查文件及文件路径是否存在。是否会自动创建路径、或是否会覆盖已存在文件，完全由远端存储系统的语义决定。
- 如果在导出过程中出现错误，可能会有导出文件残留在远端存储系统上。Doris 不会清理这些文件。需要用户手动清理。
- 导出命令的超时时间同查询的超时时间。可以通过 `SET query_timeout=xxx` 进行设置。
- 对于结果集为空的查询，依然会产生一个大小为0的文件。
- 文件切分会保证一行数据完整的存储在单一文件中。因此文件的大小并不严格等于 `max_file_size`。
- 对于部分输出为非可见字符的函数，如 BITMAP、HLL 类型，输出为 `\N`，即 NULL。
- 目前部分地理信息函数，如 `ST_Point` 的输出类型为 VARCHAR，但实际输出值为经过编码的二进制字符。当前这些函数会输出乱码。对于地理函数，请使用 `ST_AsText` 进行输出。
