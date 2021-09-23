---
layout: post
title: "Apache Doris Broker 数据导入"
date: 2021-09-22
description: "Apache Doris Broker 数据导入"
tag: Apache Doris
---
# Apache Doris  Broker数据导入

## 1.概要

Broker load 是一个异步的导入方式，支持的数据源取决于 Broker 进程支持的数据源。

用户需要通过 MySQL 协议 创建 Broker load 导入，并通过查看导入命令检查导入结果

主要适用于以下场景：

- 外部数据源（如 HDFS等）读取数据，导入到Doris中。
- 数据量在 几十到百GB 级别。
- 主要用于数据迁移，或者定时批量导入

Broker load 支持文件类型：PARQUET、ORC、CSV格式

## 2. 原理

用户在提交导入任务后，FE 会生成对应的 Plan 并根据目前 BE 的个数和文件的大小，将 Plan 分给 多个 BE 执行，每个 BE 执行一部分导入数据。

BE 在执行的过程中会从 Broker 拉取数据，在对数据 transform 之后将数据导入系统。所有 BE 均完成导入，由 FE 最终决定导入是否成功

<img src="C:\Users\zhang\AppData\Roaming\Typora\typora-user-images\image-20210922153814143.png" style="zoom:50%;" />

## 3. 使用方式

Apache Doris Broker Load方式是通过 doris 提供的 Broker load SQL语句创建。

### 3.1 SQL 语法

下面是 SQL 语法，具体使用不清楚的地方也可以在Mysql Client 命令行下执行 `help broker load`查看具体使用方法

```sql
LOAD LABEL db_name.label_name 
(data_desc, ...)
WITH BROKER broker_name broker_properties
[PROPERTIES (key1=value1, ... )]

* data_desc:

    DATA INFILE ('file_path', ...)
    [NEGATIVE]
    INTO TABLE tbl_name
    [PARTITION (p1, p2)]
    [COLUMNS TERMINATED BY separator ]
    [(col1, ...)]
    [PRECEDING FILTER predicate]
    [SET (k1=f1(xx), k2=f2(xx))]
    [WHERE predicate]

* broker_properties: 

    (key1=value1, ...)
```

### 3.2 实例

这里我们使用Broker load方式，从hive 分区表中将数据导入到Doris指定的表中

下面是我的hive表结构，数据格式是：ORC，分区字段是：budat

```sql
CREATE TABLE purchase_hive_iostock (
  lgort string NULL,
  mblnr string NULL,
  mblpo string NULL,
  werks string NULL,
  ebeln string NULL,
  ebelp string NULL,
  aufnr string NULL,
  rsnum string NULL,
  rspos string NULL,
  kdauf string NULL,
  kdpos string NULL,
  bwart string NULL,
  menge decimal(18, 3) NULL,,
  meins string NULL ,
  matnr string NULL ,
  bukrs string NULL ,
  waers string NULL ,
  dmbtr decimal(18, 3) NULL ,
  shkzg string NULL    ,
  bstme string NULL    ,
  bstmg decimal(13, 2) NULL ,
  temp1 string NULL ,
  temp2 string NULL ,
  temp3 string NULL ,
  temp4 string NULL ,
  temp5 string NULL ,
  rq datetime NULL        
)
COMMENT '出入库记录'
PARTITIONED BY(budat STRING)
....
```

Doris 中对应的表：

```sql

CREATE TABLE `ods_purchase_hive_iostock_delta` (
  `budat` date NOT NULL    ,
  `lgort` varchar(100) NULL,
  `mblnr` varchar(100) NULL,
  `mblpo` varchar(100) NULL,
  `werks` varchar(100) NULL,
  `ebeln` varchar(100) NULL,
  `ebelp` varchar(100) NULL,
  `aufnr` varchar(100) NULL,
  `rsnum` varchar(100) NULL,
  `rspos` varchar(100) NULL,
  `kdauf` varchar(100) NULL,
  `kdpos` varchar(100) NULL,
  `bwart` varchar(100) NULL,
  `menge` decimal(18, 3) NULL ,
  `meins` varchar(100) NULL,
  `matnr` varchar(100) NULL,
  `bukrs` varchar(100) NULL,
  `waers` varchar(100) NULL,
  `dmbtr` decimal(18, 3) NULL,
  `shkzg` varchar(10) NULL   ,
  `bstme` varchar(20) NULL   ,
  `bstmg` decimal(13, 2) NULL,
  `temp1` varchar(100) NULL ,
  `temp2` varchar(100) NULL ,
  `temp3` varchar(100) NULL ,
  `temp4` varchar(100) NULL ,
  `temp5` varchar(100) NULL ,
  `rq` datetime NULL ,
) ENGINE=OLAP
UNIQUE KEY(`budat`, `lgort`, `mblnr`, `mblpo`, `werks`)
COMMENT "出入库记录数据表"
PARTITION BY RANGE(`budat`)
(PARTITION P_000000 VALUES [('0000-01-01'), ('2021-01-01')),
PARTITION P_202101 VALUES [('2021-01-01'), ('2021-02-01')),
PARTITION P_202102 VALUES [('2021-02-01'), ('2021-03-01')),
PARTITION P_202103 VALUES [('2021-03-01'), ('2021-04-01')),
PARTITION P_202104 VALUES [('2021-04-01'), ('2021-05-01')),
PARTITION P_202105 VALUES [('2021-05-01'), ('2021-06-01')),
PARTITION P_202106 VALUES [('2021-06-01'), ('2021-07-01')),
PARTITION P_202107 VALUES [('2021-07-01'), ('2021-08-01')),
PARTITION P_202108 VALUES [('2021-08-01'), ('2021-09-01')),
PARTITION P_202109 VALUES [('2021-09-01'), ('2021-10-01')),
PARTITION P_202110 VALUES [('2021-10-01'), ('2021-11-01')),
PARTITION P_202111 VALUES [('2021-11-01'), ('2021-12-01')))
DISTRIBUTED BY HASH(`werks`) BUCKETS 5
PROPERTIES (
"replication_num" = "3",
"dynamic_partition.enable" = "true",
"dynamic_partition.time_unit" = "MONTH",
"dynamic_partition.time_zone" = "Asia/Shanghai",
"dynamic_partition.start" = "-2147483648",
"dynamic_partition.end" = "2",
"dynamic_partition.prefix" = "P_",
"dynamic_partition.replication_num" = "3",
"dynamic_partition.buckets" = "5",
"dynamic_partition.start_day_of_month" = "1",
"in_memory" = "false",
"storage_format" = "V2"
);
```

下面的语句是将Hive 分区表中的数据导入到Doris 对应的表中

```sql
LOAD LABEL order_bill_2021_0915_3
(
    DATA INFILE("hdfs://namenodeservice1/user/data/hive_db/data_ods.db/purchase_hive_iostock/*/*")
    INTO TABLE ods_purchase_hive_iostock_delta
    COLUMNS TERMINATED BY "\\x01"
    FORMAT AS "orc"   (lgort,mblnr,mblpo,werks,ebeln,ebelp,aufnr,rsnum,rspos,kdauf,kdpos,bwart,menge,meins,matnr,bukrs,waers,dmbtr,shkzg,bstme,bstmg,temp1,temp2,temp3,temp4,temp5,rq)
    COLUMNS FROM PATH AS (budat)
    SET (budat=budat,lgort=lgort,mblnr=mblnr,mblpo=mblpo,werks=werks,ebeln=ebeln,ebelp=ebelp,aufnr=aufnr,rsnum=rsnum,rspos=rspos,kdauf=kdauf,kdpos=kdpos,bwart=bwart,menge=menge,meins=meins,matnr=matnr,bukrs=bukrs,waers=waers,dmbtr=dmbtr,shkzg=shkzg,bstme=bstme,bstmg=bstmg,temp1=temp1,temp2=temp2,temp3=temp3,temp4=temp4,temp5=temp5,rq=rq)
    where 1=1
)
WITH BROKER "hdfs_broker"
(
    "dfs.nameservices"="hadoop",
    "dfs.ha.namenodes.eadhadoop" = "nn1,nn2",
    "dfs.namenode.rpc-address.eadhadoop.nn1" = "222:8000",
    "dfs.namenode.rpc-address.eadhadoop.nn2" = "117:8000",
    "dfs.client.failover.proxy.provider" = "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider",
    "hadoop.security.authentication" = "kerberos",
    "kerberos_principal" = "ddd.COM",
    "kerberos_keytab_content" = "BQHIININ1111URAAE="
)
PROPERTIES
(
    "timeout"="1200",
    "max_filter_ratio"="0.1"
);
```

然后提交任务就行了

### 3.3 认证方式

- 支持简单认证访问
- 支持通过 kerberos 认证访问
- 支持 HDFS HA 模式访问

#### 3.3.1 简单认证方式

简单认证即 Hadoop 配置 `hadoop.security.authentication` 为 `simple`。

使用系统用户访问 HDFS。或者在 Broker 启动的环境变量中添加：`HADOOP_USER_NAME`。密码置空即可

```
(
    "username" = "user",
    "password" = ""
);
```

#### 3.3.2 Kerberos 认证

上面示例使用的就是Kerberos认证方式。

该认证方式需提供以下信息：

- `hadoop.security.authentication`：指定认证方式为 kerberos。
- `kerberos_principal`：指定 kerberos 的 principal。
- `kerberos_keytab`：指定 kerberos 的 keytab 文件路径。该文件必须为 Broker 进程所在服务器上的文件的绝对路径。并且可以被 Broker 进程访问。
- `kerberos_keytab_content`：指定 kerberos 中 keytab 文件内容经过 base64 编码之后的内容。这个跟 `kerberos_keytab` 配置二选一即可

示例：

```
(
    "hadoop.security.authentication" = "kerberos",
    "kerberos_principal" = "doris@YOUR.COM",
    "kerberos_keytab" = "/home/doris/my.keytab"
)
```

或者

```
(
    "hadoop.security.authentication" = "kerberos",
    "kerberos_principal" = "doris@YOUR.COM",
    "kerberos_keytab_content" = "ASDOWHDLAWIDJHWLDKSALDJSDIWALD"
)
```

如果采用Kerberos认证方式，则部署Broker进程的时候需要`krb5.conf`文件， `krb5.conf`文件包含Kerberos的配置信息，通常，您应该将`krb5.conf`文件安装在目录/etc中。您可以通过设置环境变量KRB5_CONFIG覆盖默认位置。 `krb5.conf`文件的内容示例如下：

```
[libdefaults]
    default_realm = DORIS.HADOOP
    default_tkt_enctypes = des3-hmac-sha1 des-cbc-crc
    default_tgs_enctypes = des3-hmac-sha1 des-cbc-crc
    dns_lookup_kdc = true
    dns_lookup_realm = false

[realms]
    DORIS.HADOOP = {
        kdc = kerberos-doris.hadoop.service:7005
    }
```

#### 3.3.3 HDFS HA 模式

Doris 只是HDFS HA模式，下面这个配置用于访问以 HA 模式部署的 HDFS 集群。

- `dfs.nameservices`：指定 hdfs 服务的名字，自定义，如："dfs.nameservices" = "my_ha"。
- `dfs.ha.namenodes.xxx`：自定义 namenode 的名字,多个名字以逗号分隔。其中 xxx 为 `dfs.nameservices` 中自定义的名字，如： "dfs.ha.namenodes.my_ha" = "my_nn"。
- `dfs.namenode.rpc-address.xxx.nn`：指定 namenode 的rpc地址信息。其中 nn 表示 `dfs.ha.namenodes.xxx` 中配置的 namenode 的名字，如："dfs.namenode.rpc-address.my_ha.my_nn" = "host:port"。
- `dfs.client.failover.proxy.provider`：指定 client 连接 namenode 的 provider，默认为：org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider。

示例如下：

```
(
    "dfs.nameservices" = "my_ha",
    "dfs.ha.namenodes.my_ha" = "my_namenode1, my_namenode2",
    "dfs.namenode.rpc-address.my_ha.my_namenode1" = "nn1_host:rpc_port",
    "dfs.namenode.rpc-address.my_ha.my_namenode2" = "nn2_host:rpc_port",
    "dfs.client.failover.proxy.provider" = "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider"
)
```

如果HA模式和认证结合使用，示例如下：

```
(
    "username"="user",
    "password"="passwd",
    "dfs.nameservices" = "my_ha",
    "dfs.ha.namenodes.my_ha" = "my_namenode1, my_namenode2",
    "dfs.namenode.rpc-address.my_ha.my_namenode1" = "nn1_host:rpc_port",
    "dfs.namenode.rpc-address.my_ha.my_namenode2" = "nn2_host:rpc_port",
    "dfs.client.failover.proxy.provider" = "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider"
)
```

关于HDFS集群的配置可以写入hdfs-site.xml文件中，用户使用Broker进程读取HDFS集群的信息时，只需要填写集群的文件路径名和认证信息即可。

### 3.4 主要参数介绍

创建导入的详细语法执行 `HELP BROKER LOAD` 查看语法帮助。这里主要介绍 Broker load 的创建导入语法中参数意义和注意事项：

#### 3.4.1 Label

导入任务的标识。每个导入任务，都有一个在单 database 内部唯一的 Label。Label 是用户在导入命令中自定义的名称。通过这个 Label，用户可以查看对应导入任务的执行情况。

Label 的另一个作用，是防止用户重复导入相同的数据。**强烈推荐用户同一批次数据使用相同的label。这样同一批次数据的重复请求只会被接受一次，保证了 At-Most-Once 语义**

当 Label 对应的导入作业状态为 CANCELLED 时，可以再次使用该 Label 提交导入作业

#### 3.4.2 数据描述类参数

数据描述类参数主要指的是 Broker load 创建导入语句中的属于 `data_desc` 部分的参数。每组 `data_desc` 主要表述了本次导入涉及到的数据源地址，ETL 函数，目标表及分区等信息

- 多表导入

  Broker load 支持一次导入任务涉及多张表，每个 Broker load 导入任务可在多个 `data_desc` 声明多张表来实现多表导入。每个单独的 `data_desc` 还可以指定属于该表的数据源地址。Broker load 保证了单次导入的多张表之间原子性成功或失败。

- negative

  `data_desc`中还可以设置数据取反导入。这个功能主要用于，当数据表中聚合列的类型都为 SUM 类型时。如果希望撤销某一批导入的数据。则可以通过 `negative` 参数导入同一批数据。Doris 会自动为这一批数据在聚合列上数据取反，以达到消除同一批数据的功能。

- partition

  在 `data_desc` 中可以指定待导入表的 partition 信息，如果待导入数据不属于指定的 partition 则不会被导入。同时，不在指定 Partition 的数据会被认为是错误数据。

- set column mapping

  在 `data_desc` 中的 SET 语句负责设置列函数变换，这里的列函数变换支持所有查询的等值表达式变换。如果原始数据的列和表中的列不一一对应，就需要用到这个属性。

- preceding filter predicate

  用于过滤原始数据。原始数据是未经列映射、转换的数据。用户可以在对转换前的数据前进行一次过滤，选取期望的数据，再进行转换。

- where predicate

  在 `data_desc` 中的 WHERE 语句中负责过滤已经完成 transform 的数据，被 filter 的数据不会进入容忍率的统计中。如果多个 data_desc 中声明了同一张表的多个条件的话，则会 merge 同一张表的多个条件，merge 策略是 AND 

- COLUMNS FROM PATH AS

  因为Hive的分区是体现在HDFS的路径上，我们在导入的时候，可以通过这个参数动态的通过HDFS路径获取hive分区字段，
  
- FORMAT

  数据源的数据文件格式，如果是PARQUET或者ORC格式的数据,需要再文件头的列名与doris表中的列名一致
  
  示例：
  
  ```text
  (tmp_c1,tmp_c2)
  SET
  (
      id=tmp_c2,
      name=tmp_c1
  )
  ```

#### 3.4.3 导入作业参数

导入作业参数主要指的是 Broker load 创建导入语句中的属于 `opt_properties`部分的参数。导入作业参数是作用于整个导入作业的。

- timeout

  导入作业的超时时间(以秒为单位)，用户可以在 `opt_properties` 中自行设置每个导入的超时时间。导入任务在设定的 timeout 时间内未完成则会被系统取消，变成 CANCELLED。Broker load 的默认导入超时时间为4小时。

  通常情况下，用户不需要手动设置导入任务的超时时间。当在默认超时时间内无法完成导入时，可以手动设置任务的超时时间。

  > 推荐超时时间
  >
  > 总文件大小（MB） / 用户 Doris 集群最慢导入速度(MB/s) > timeout > （（总文件大小(MB) * 待导入的表及相关 Roll up 表的个数） / (10 * 导入并发数） ）

  > 导入并发数见文档最后的导入系统配置说明，公式中的 10 为目前的导入限速 10MB/s。

  > 例如一个 1G 的待导入数据，待导入表包含3个 Rollup 表，当前的导入并发数为 3。则 timeout 的 最小值为 `(1 * 1024 * 3 ) / (10 * 3) = 102 秒`

  由于每个 Doris 集群的机器环境不同且集群并发的查询任务也不同，所以用户 Doris 集群的最慢导入速度需要用户自己根据历史的导入任务速度进行推测。

- max_filter_ratio

  导入任务的最大容忍率，默认为0容忍，取值范围是0~1。当导入的错误率超过该值，则导入失败。

  如果用户希望忽略错误的行，可以通过设置这个参数大于 0，来保证导入可以成功。

  计算公式为：

  `max_filter_ratio = (dpp.abnorm.ALL / (dpp.abnorm.ALL + dpp.norm.ALL ) )`

  `dpp.abnorm.ALL` 表示数据质量不合格的行数。如类型不匹配，列数不匹配，长度不匹配等等。

  `dpp.norm.ALL` 指的是导入过程中正确数据的条数。可以通过 `SHOW LOAD` 命令查询导入任务的正确数据量。

  原始文件的行数 = `dpp.abnorm.ALL + dpp.norm.ALL`

- exec_mem_limit

  导入内存限制。默认是 2GB。单位为字节。

- strict_mode

  Broker load 导入可以开启 strict mode 模式。开启方式为 `properties ("strict_mode" = "true")` 。默认的 strict mode 为关闭。

  strict mode 模式的意思是：对于导入过程中的列类型转换进行严格过滤。严格过滤的策略如下：

  1. 对于列类型转换来说，如果 strict mode 为true，则错误的数据将被 filter。这里的错误数据是指：原始数据并不为空值，在参与列类型转换后结果为空值的这一类数据。
  2. 对于导入的某列由函数变换生成时，strict mode 对其不产生影响。
  3. 对于导入的某列类型包含范围限制的，如果原始数据能正常通过类型转换，但无法通过范围限制的，strict mode 对其也不产生影响。例如：如果类型是 decimal(1,0), 原始数据为 10，则属于可以通过类型转换但不在列声明的范围内。这种数据 strict 对其不产生影响。

- merge_type 数据的合并类型，一共支持三种类型APPEND、DELETE、MERGE 其中，APPEND是默认值，表示这批数据全部需要追加到现有数据中，DELETE 表示删除与这批数据key相同的所有行，MERGE 语义 需要与delete 条件联合使用，表示满足delete 条件的数据按照DELETE 语义处理其余的按照APPEND 语义处理

#### 3.4.4 strict mode 与 source data 的导入关系

这里以列类型为 TinyInt 来举例

> 注：当表中的列允许导入空值时

| 源数据   | 源数据示例  | string to int | strict_mode   | result           |
| -------- | ----------- | ------------- | ------------- | ---------------- |
| 空值     | \N          | N/A           | true or false | NULL             |
| not null | aaa or 2000 | NULL          | true          | 无效数据(被过滤) |
| not null | aaa         | NULL          | false         | NULL             |
| not null | 1           | 1             | true or false | 正确数据         |

这里以列类型为 Decimal(1,0) 举例

> 注：当表中的列允许导入空值时

| 源数据   | 源数据示例 | string to int | strict_mode   | result           |
| -------- | ---------- | ------------- | ------------- | ---------------- |
| 空值     | \N         | N/A           | true or false | NULL             |
| not null | aaa        | NULL          | true          | 无效数据(被过滤) |
| not null | aaa        | NULL          | false         | NULL             |
| not null | 1 or 10    | 1             | true or false | 正确数据         |

> 注意：10 虽然是一个超过范围的值，但是因为其类型符合 decimal的要求，所以 strict mode对其不产生影响。10 最后会在其他 ETL 处理流程中被过滤。但不会被 strict mode 过滤。

### 3.5 查看结果

Broker load 导入方式由于是异步的，所以用户必须将创建导入的 Label 记录，并且在**查看导入命令中使用 Label 来查看导入结果**。查看导入命令在所有导入方式中是通用的，具体语法可执行 `HELP SHOW LOAD` 查看。

```sql
mysql> show load order by createtime desc limit 1\G
*************************** 1. row ***************************
         JobId: 76391
         Label: label1
         State: FINISHED
      Progress: ETL:N/A; LOAD:100%
          Type: BROKER
       EtlInfo: unselected.rows=4; dpp.abnorm.ALL=15; dpp.norm.ALL=28133376
      TaskInfo: cluster:N/A; timeout(s):10800; max_filter_ratio:5.0E-5
      ErrorMsg: N/A
    CreateTime: 2019-07-27 11:46:42
  EtlStartTime: 2019-07-27 11:46:44
 EtlFinishTime: 2019-07-27 11:46:44
 LoadStartTime: 2019-07-27 11:46:44
LoadFinishTime: 2019-07-27 11:50:16
           URL: http://192.168.1.10:8040/api/_load_error_log?file=__shard_4/error_log_insert_stmt_4bb00753932c491a-a6da6e2725415317_4bb00753932c491a_a6da6e2725415317
    JobDetails: {"Unfinished backends":{"9c3441027ff948a0-8287923329a2b6a7":[10002]},"ScannedRows":2390016,"TaskNumber":1,"All backends":{"9c3441027ff948a0-8287923329a2b6a7":[10002]},"FileNumber":1,"FileSize":1073741824}

```

下面主要介绍了查看导入命令返回结果集中参数意义：

- JobId

  导入任务的唯一ID，每个导入任务的 JobId 都不同，由系统自动生成。与 Label 不同的是，JobId永远不会相同，而 Label 则可以在导入任务失败后被复用。

- Label

  导入任务的标识。

- State

  导入任务当前所处的阶段。在 Broker load 导入过程中主要会出现 PENDING 和 LOADING 这两个导入中的状态。如果 Broker load 处于 PENDING 状态，则说明当前导入任务正在等待被执行；LOADING 状态则表示正在执行中。

  导入任务的最终阶段有两个：CANCELLED 和 FINISHED，当 Load job 处于这两个阶段时，导入完成。其中 CANCELLED 为导入失败，FINISHED 为导入成功。

- Progress

  导入任务的进度描述。分为两种进度：ETL 和 LOAD，对应了导入流程的两个阶段 ETL 和 LOADING。目前 Broker load 由于只有 LOADING 阶段，所以 ETL 则会永远显示为 `N/A`

  LOAD 的进度范围为：0~100%。

  `LOAD 进度 = 当前完成导入的表个数 / 本次导入任务设计的总表个数 * 100%`

  **如果所有导入表均完成导入，此时 LOAD 的进度为 99%** 导入进入到最后生效阶段，整个导入完成后，LOAD 的进度才会改为 100%。

  导入进度并不是线性的。所以如果一段时间内进度没有变化，并不代表导入没有在执行。

- Type

  导入任务的类型。Broker load 的 type 取值只有 BROKER。

- EtlInfo

  主要显示了导入的数据量指标 `unselected.rows` , `dpp.norm.ALL` 和 `dpp.abnorm.ALL`。用户可以根据第一个数值判断 where 条件过滤了多少行，后两个指标验证当前导入任务的错误率是否超过 `max_filter_ratio`。

  三个指标之和就是原始数据量的总行数。

- TaskInfo

  主要显示了当前导入任务参数，也就是创建 Broker load 导入任务时用户指定的导入任务参数，包括：`cluster`，`timeout` 和`max_filter_ratio`。

- ErrorMsg

  在导入任务状态为CANCELLED，会显示失败的原因，显示分两部分：type 和 msg，如果导入任务成功则显示 `N/A`。

  type的取值意义：

  ```text
  USER_CANCEL: 用户取消的任务
  ETL_RUN_FAIL：在ETL阶段失败的导入任务
  ETL_QUALITY_UNSATISFIED：数据质量不合格，也就是错误数据率超过了 max_filter_ratio
  LOAD_RUN_FAIL：在LOADING阶段失败的导入任务
  TIMEOUT：导入任务没在超时时间内完成
  UNKNOWN：未知的导入错误
  ```

- CreateTime/EtlStartTime/EtlFinishTime/LoadStartTime/LoadFinishTime

  这几个值分别代表导入创建的时间，ETL阶段开始的时间，ETL阶段完成的时间，Loading阶段开始的时间和整个导入任务完成的时间。

  Broker load 导入由于没有 ETL 阶段，所以其 EtlStartTime, EtlFinishTime, LoadStartTime 被设置为同一个值。

  导入任务长时间停留在 CreateTime，而 LoadStartTime 为 N/A 则说明目前导入任务堆积严重。用户可减少导入提交的频率。

  ```text
  LoadFinishTime - CreateTime = 整个导入任务所消耗时间
  LoadFinishTime - LoadStartTime = 整个 Broker load 导入任务执行时间 = 整个导入任务所消耗时间 - 导入任务等待的时间
  ```

- URL

  导入任务的错误数据样例，访问 URL 地址既可获取本次导入的错误数据样例。当本次导入不存在错误数据时，URL 字段则为 N/A。

- JobDetails

  显示一些作业的详细运行状态。包括导入文件的个数、总大小（字节）、子任务个数、已处理的原始行数，运行子任务的 BE 节点 Id，未完成的 BE 节点 Id。

  ```text
  {"Unfinished backends":{"9c3441027ff948a0-8287923329a2b6a7":[10002]},"ScannedRows":2390016,"TaskNumber":1,"All backends":{"9c3441027ff948a0-8287923329a2b6a7":[10002]},"FileNumber":1,"FileSize":1073741824}
  ```

  其中已处理的原始行数，每 5 秒更新一次。该行数仅用于展示当前的进度，不代表最终实际的处理行数。实际处理行数以 EtlInfo 中显示的为准。

### 3.6 取消导入

当 Broker load 作业状态不为 CANCELLED 或 FINISHED 时，可以被用户手动取消。取消时需要指定待取消导入任务的 Label 。取消导入命令语法可执行 `HELP CANCEL LOAD`查看

### 3.7 Broker 导入相关参数

以下三个参数主要是为了控制导入的速度

- min_bytes_per_broker_scanner/max_bytes_per_broker_scanner/max_broker_concurrency

  前两个配置限制了单个 BE 处理的数据量的最小和最大值。第三个配置限制了一个作业的最大的导入并发数。最小处理的数据量，最大并发数，源文件的大小和当前集群 BE 的个数 **共同决定了本次导入的并发数**。

  ```text
  本次导入并发数 = Math.min(源文件大小/最小处理量，最大并发数，当前BE节点个数)
  本次导入单个BE的处理量 = 源文件大小/本次导入的并发数
  ```

  通常一个导入作业支持的最大数据量为 `max_bytes_per_broker_scanner * BE 节点数`。如果需要导入更大数据量，则需要适当调整 `max_bytes_per_broker_scanner` 参数的大小。

  默认配置：

  ```text
  参数名：min_bytes_per_broker_scanner， 默认 64MB，单位bytes。
  参数名：max_broker_concurrency， 默认 10。
  参数名：max_bytes_per_broker_scanner，默认 3G，单位bytes。
  ```

## 4. 最佳实践

使用 Broker load 最适合的场景就是原始数据在文件系统（HDFS等）中的场景。其次，由于 Broker load 是单次导入中唯一的一种异步导入的方式，所以如果用户在导入大文件中，需要使用异步接入，也可以考虑使用 Broker load

下面以单个BE使用Broker方式导入，在导入的数据量上的建议，如果你有多个BE，数据量应该乘以BE的数量来进行计算

1. 3G数据

   用户可以直接提交 Broker load 创建导入请求。

2. 3G以上

   由于单个导入 BE 最大的处理量为 3G，超过 3G 的待导入文件就需要通过调整 Broker load 的导入参数来实现大文件的导入

   - 最大扫描量和最大并发数

     根据当前 BE 的个数和原始文件的大小修改单个 BE 的最大扫描量和最大并发数

     ```
     修改 fe.conf 中配置
     
     max_broker_concurrency = BE 个数
     当前导入任务单个 BE 处理的数据量 = 原始文件大小 / max_broker_concurrency
     max_bytes_per_broker_scanner >= 当前导入任务单个 BE 处理的数据量
     
     比如一个 100G 的文件，集群的 BE 个数为 10 个
     max_broker_concurrency = 10
     max_bytes_per_broker_scanner >= 10G = 100G / 10
     ```

     修改后，所有的 BE 会并发的处理导入任务，每个 BE 处理原始文件的一部分。

     **注意：上述两个 FE 中的配置均为系统配置，也就是说其修改是作用于所有的 Broker load的任务的。**

   - 超时时间（timeout）

     在创建导入的时候自定义当前导入任务的 timeout 时间

     ```text
     当前导入任务单个 BE 处理的数据量 / 用户 Doris 集群最慢导入速度(MB/s) >= 当前导入任务的 timeout 时间 >= 当前导入任务单个 BE 处理的数据量 / 10M/s
     
     比如一个 100G 的文件，集群的 BE 个数为 10个
     timeout >= 1000s = 10G / 10M/s
     ```

   - 默认的导入最大超时时间

     当用户发现第二步计算出的 timeout 时间超过系统默认的导入最大超时时间 4小时

     这时候不推荐用户将导入最大超时时间直接改大来解决问题。单个导入时间如果超过默认的导入最大超时时间4小时，最好是通过切分待导入文件并且分多次导入来解决问题。主要原因是：单次导入超过4小时的话，导入失败后重试的时间成本很高。

     可以通过如下公式计算出 Doris 集群期望最大导入文件数据量：

     ```text
     期望最大导入文件数据量 = 14400s * 10M/s * BE 个数
     比如：集群的 BE 个数为 10个
     期望最大导入文件数据量 = 14400s * 10M/s * 10 = 1440000M ≈ 1440G
     
     注意：一般用户的环境可能达不到 10M/s 的速度，所以建议超过 500G 的文件都进行文件切分，再导入。
     ```

## 5. 性能分析

   可以在提交 LOAD 作业前，先执行 `set is_report_success=true` 打开会话变量。然后提交导入作业。待导入作业完成后，可以在 FE 的 web 页面的 `Queris` 标签中查看到导入作业的 Profile。

   这个 Profile 可以帮助分析导入作业的运行状态。

   当前只有作业成功执行后，才能查看 Profile。

   **这个同样适用去其他作业，包括查询**