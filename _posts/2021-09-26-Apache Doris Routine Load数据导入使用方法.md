---
layout: post
title: "Apache Doris Routine Load数据导入使用方法"
date: 2021-09-26
description: "Apache Doris Routine Load数据导入使用方法"
tag: Apache Doris
---
# Apache Doris Routine Load数据导入使用方法

## 1.概要

Routine load 功能为用户提供了一种自动从指定数据源进行数据导入的功能。


Routine Load 是支持用户提交一个常驻的导入任务，通过不断的从指定的数据源读取数据，将数据导入到 Doris 中。目前仅支持通过无认证或者 SSL 认证方式，从 Kakfa 导入的数据。


Routine load是一种同步的数据导入方式。

Routine load 支持导入的数据类型： 文本 和 JSON两种格式

## 2. 原理

<img src="/images/load/image-20210926092117050.png" style="zoom:50%;" />

FE 通过 JobScheduler 将一个导入作业拆分成若干个 Task（一般是和Kafka的Partition数量一致）。每个 Task 负责导入指定的一部分数据。Task 被 TaskScheduler 分配到指定的 BE 上执行。

在 BE 上，一个 Task 被视为一个普通的导入任务，通过 Stream Load 的导入机制进行导入。导入完成后，向 FE 汇报。

FE 中的 JobScheduler 根据汇报结果，继续生成后续新的 Task，或者对失败的 Task 进行重试。

整个例行导入作业通过不断的产生新的 Task，来完成数据不间断的导入

## 3. 使用方式

### 3.1 使用限制

1. 支持无认证的 Kafka 访问，以及通过 SSL 方式认证的 Kafka 集群。
2. 支持的消息格式为 csv, json 文本格式。csv 每一个 message 为一行，且行尾**不包含**换行符。
3. 仅支持 Kafka 0.10.0.0(含) 以上版本

### 3.2 Routine Load SQL语法

```sql
CREATE ROUTINE LOAD [db.]job_name ON tbl_name
[merge_type]
[load_properties]
[job_properties]
FROM data_source
[data_source_properties]
```

#### 3.2.1 Routine load 作业参数说明

1. [db.]job_name

    导入作业的名称，在同一个 database 内，相同名称只能有一个 job 在运行。

2. tbl_name

    指定需要导入的表的名称。
    
3. merge_type
    数据的合并类型，一共支持三种类型APPEND、DELETE、MERGE 其中，APPEND是默认值，表示这批数据全部需要追加到现有数据中，DELETE 表示删除与这批数据key相同的所有行，MERGE 语义 需要与delete on条件联合使用，表示满足delete 条件的数据按照DELETE 语义处理其余的按照APPEND 语义处理, 语法为[WITH MERGE|APPEND|DELETE]

#### 3.2.2 load_properties参数说明

这部分参数用于描述导入数据。语法：
    [column_separator],
    [columns_mapping],
    [where_predicates],
    [delete_on_predicates],
    [source_sequence],
    [partitions],
    [preceding_predicates]

1. column_separator:

指定列分隔符，如：      
```sql
COLUMNS TERMINATED BY ","
```

这个只在文本数据导入的时候需要指定，JSON格式的数据导入不需要指定这个参数。

默认为：\t

2. columns_mapping:

指定源数据中列的映射关系，以及定义衍生列的生成方式。
        
- 映射列：

按顺序指定，源数据中各个列，对应目的表中的哪些列。对于希望跳过的列，可以指定一个不存在的列名。假设目的表有三列 k1, k2, v1。源数据有4列，其中第1、2、4列分别对应 k2, k1, v1。则书写如下： 

```sql
COLUMNS (k2, k1, xxx, v1)
```

其中 xxx 为不存在的一列，用于跳过源数据中的第三列。
        

- 衍生列：

以 col_name = expr 的形式表示的列，我们称为衍生列。即支持通过 expr 计算得出目的表中对应列的值。 衍生列通常排列在映射列之后，虽然这不是强制的规定，但是 Doris 总是先解析映射列，再解析衍生列。  接上一个示例，假设目的表还有第4列 v2，v2 由 k1 和 k2 的和产生。则可以书写如下：        
```sql
COLUMNS (k2, k1, xxx, v1, v2 = k1 + k2);
```

再举例，假设用户需要导入只包含 `k1` 一列的表，列类型为 `int`。并且需要将源文件中的对应列进行处理：将负数转换为正数，而将正数乘以 100。这个功能可以通过 `case when` 函数实现，正确写法应如下：

```sql
COLUMNS (xx, k1 = case when xx < 0 then cast(-xx as varchar) else cast((xx + '100') as varchar) end)
```

1. where_predicates

用于指定过滤条件，以过滤掉不需要的列。过滤列可以是映射列或衍生列。 例如我们只希望导入 k1 大于 100 并且 k2 等于 1000 的列，则书写如下：        
```sql
WHERE k1 > 100 and k2 = 1000
```

4. partitions

指定导入目的表的哪些 partition 中。如果不指定，则会自动导入到对应的 partition 中。

示例：        

```sql
PARTITION(p1, p2, p3)
```

5. delete_on_predicates

  表示删除条件，仅在 merge type 为MERGE 时有意义，语法与where 相同

6. source_sequence:


只适用于UNIQUE_KEYS,相同key列下，保证value列按照source_sequence列进行REPLACE, source_sequence可以是数据源中的列，也可以是表结构中的一列。

7. preceding_predicates

```
PRECEDING FILTER predicate
```

用于过滤原始数据。原始数据是未经列映射、转换的数据。用户可以在对转换前的数据前进行一次过滤，选取期望的数据，再进行转换。

#### 3.3.3 job_properties参数说明

用于指定例行导入作业的通用参数。
语法：

    PROPERTIES (
        "key1" = "val1",
        "key2" = "val2"
    )

目前支持以下参数：

1. desired_concurrent_number

   期望的并发度。一个例行导入作业会被分成多个子任务执行。这个参数指定一个作业最多有多少任务可以同时执行。必须大于0。默认为3。 这个并发度并不是实际的并发度，实际的并发度，会通过集群的节点数、负载情况，以及数据源的情况综合考虑。
   例如：

       "desired_concurrent_number" = "3"

   一个作业，最多有多少 task 同时在执行。对于 Kafka 导入而言，当前的实际并发度计算如下：

   ```text
   Min(partition num, desired_concurrent_number, alive_backend_num, Config.max_routine_load_task_concurrrent_num)
   ```

   其中 `Config.max_routine_load_task_concurrrent_num` 是系统的一个默认的最大并发数限制。这是一个 FE 配置，可以通过改配置调整。默认为 5。

   其中 partition num 指订阅的 Kafka topic 的 partition 数量。`alive_backend_num` 是当前正常的 BE 节点数。

2. max_batch_interval/max_batch_rows/max_batch_size

   这三个参数分别表示：
           1）每个子任务最大执行时间，单位是秒。范围为 5 到 60。默认为10。
           2）每个子任务最多读取的行数。必须大于等于200000。默认是200000。
           3）每个子任务最多读取的字节数。单位是字节，范围是 100MB 到 1GB。默认是 100MB。

   这三个参数，用于控制一个子任务的执行时间和处理量。当任意一个达到阈值，则任务结束。
   例如：

       "max_batch_interval" = "20",
       "max_batch_rows" = "300000",
       "max_batch_size" = "209715200"

3. max_error_number

   采样窗口内，允许的最大错误行数。必须大于等于0。默认是 0，即不允许有错误行。 采样窗口为 max_batch_rows * 10。即如果在采样窗口内，错误行数大于 max_error_number，则会导致例行作业被暂停，需要人工介入检查数据质量问题。  被 where 条件过滤掉的行不算错误行

4. strict_mode

   是否开启严格模式，默认为关闭。如果开启后，非空原始数据的列类型变换如果结果为 NULL，则会被过滤。指定方式为 "strict_mode" = "true"

5. timezone

   指定导入作业所使用的时区。默认为使用 Session 的 timezone 参数。该参数会影响所有导入涉及的和时区有关的函数结果

6. format

    指定导入数据格式，默认是csv，支持json格式

7. jsonpaths

   jsonpaths: 导入json方式分为：简单模式和匹配模式。如果设置了jsonpath则为匹配模式导入，否则为简单模式导入，具体可参考示例

8. strip_outer_array

   布尔类型，为true表示json数据以数组对象开始且将数组对象中进行展平，默认值是false

9. json_root

   json_root为合法的jsonpath字符串，用于指定json document的根节点，默认值为""

10. send_batch_parallelism

整型，用于设置发送批处理数据的并行度，如果并行度的值超过 BE 配置中的 `max_send_batch_parallelism_per_job`，那么作为协调点的 BE 将使用 `max_send_batch_parallelism_per_job` 的值

#### 3.3.4 数据源参数说明

数据源的类型。当前支持：Kafka

指定数据源相关的信息。

语法：

```sql
(
    "key1" = "val1",
    "key2" = "val2"
)
```

1. kafka_broker_list

    Kafka 的 broker 连接信息。格式为 ip:host。多个broker之间以逗号分隔。
   示例：

       "kafka_broker_list" = "broker1:9092,broker2:9092"

2. kafka_topic

   指定要订阅的 Kafka 的 topic。
   示例：

       "kafka_topic" = "my_topic"

3. kafka_partitions/kafka_offsets

   指定需要订阅的 kafka partition，以及对应的每个 partition 的起始 offset。

   offset 可以指定从大于等于 0 的具体 offset，或者：

      - OFFSET_BEGINNING: 从有数据的位置开始订阅。
      - OFFSET_END: 从末尾开始订阅。
      - 时间戳，格式必须如："2021-05-11 10:00:00"，系统会自动定位到大于等于该时间戳的第一个消息的offset。注意，时间戳格式的offset不能和数字类型混用，只能选其一。

    如果没有指定，则默认从 OFFSET_END 开始订阅 topic 下的所有 partition。
    示例：


    "kafka_partitions" = "0,1,2,3",
    "kafka_offsets" = "101,0,OFFSET_BEGINNING,OFFSET_END"
    "kafka_partitions" = "0,1",
    "kafka_offsets" = "2021-05-11 10:00:00, 2021-05-11 11:00:00"

4. property

   指定自定义kafka参数。

   - 功能等同于kafka shell中 "--property" 参数。
   - 当参数的 value 为一个文件时，需要在 value 前加上关键词："FILE:"。
   - 关于如何创建文件，请参阅 "HELP CREATE FILE;"
   - 更多支持的自定义参数，请参阅 librdkafka 的官方 CONFIGURATION 文档中，client 端的配置项。

示例:
               

```
"property.client.id" = "12345",
"property.ssl.ca.location" = "FILE:ca.pem"
```

**1.使用 SSL 连接 Kafka 时，需要指定以下参数：**

    "property.security.protocol" = "ssl",
    "property.ssl.ca.location" = "FILE:ca.pem",
    "property.ssl.certificate.location" = "FILE:client.pem",
    "property.ssl.key.location" = "FILE:client.key",
    "property.ssl.key.password" = "abcdefg"
其中： "property.security.protocol" 和 "property.ssl.ca.location" 为必须，用于指明连接方式为 SSL，以及 CA 

证书的位置。

 如果 Kafka server 端开启了 client 认证，则还需设置：

    "property.ssl.certificate.location"
    "property.ssl.key.location"
    "property.ssl.key.password"

分别用于指定 client 的 public key，private key 以及 private key 的密码

**2.指定kafka partition的默认起始offset**

如果没有指定kafka_partitions/kafka_offsets,默认消费所有分区,此时可以指定kafka_default_offsets指定起始 

offset。默认为 OFFSET_END，即从末尾开始订阅。
 值为
                  1) OFFSET_BEGINNING: 从有数据的位置开始订阅。
                  2) ND: 从末尾开始订阅。
                  3) 时间戳，格式同 kafka_offsets

示例：

    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
    "property.kafka_default_offsets" = "2021-05-11 10:00:00"

#### 3.3.5 导入数据格式样例

1.  整型类（TINYINT/SMALLINT/INT/BIGINT/LARGEINT）：1, 1000, 1234
2.  浮点类（FLOAT/DOUBLE/DECIMAL）：1.1, 0.23, .356
3.  日期类（DATE/DATETIME）：2017-10-03, 2017-06-13 12:34:03。
4.  字符串类（CHAR/VARCHAR）（无引号）：I am a student, a
5.   NULL值：\N

### 3.3 查看作业状态

查看**作业**状态的具体命令和示例可以通过 `HELP SHOW ROUTINE LOAD;` 命令查看。

查看**任务**运行状态的具体命令和示例可以通过 `HELP SHOW ROUTINE LOAD TASK;` 命令查看。

只能查看当前正在运行中的任务，已结束和未开始的任务无法查看

### 3.4 修改作业属性

用户可以修改已经创建的作业。具体说明可以通过 `HELP ALTER ROUTINE LOAD;` 命令查看。

### 3.5 作业控制

用户可以通过 `STOP/PAUSE/RESUME` 三个命令来控制作业的停止，暂停和重启。可以通过 `HELP STOP ROUTINE LOAD;`, `HELP PAUSE ROUTINE LOAD;` 以及 `HELP RESUME ROUTINE LOAD;` 三个命令查看帮助和示例。



## 4. 使用示例

### 4.1 创建Doris数据表

```sql
CREATE TABLE `example_table` (
  `category` int,
  `author` varchar(11),  
  `timestamp` int,
  `dt` varchar(50)
) 
DISTRIBUTED BY HASH(id) BUCKETS 2
PROPERTIES( 
"replication_num" = "3"
);
```

### 4.2 创建Routine Load 任务

这个示例是以JSON格式为例

```sql
CREATE ROUTINE LOAD example_db.test1 ON example_tbl
COLUMNS(category, author, price, timestamp, dt=from_unixtime(timestamp, '%Y%m%d'))
PROPERTIES
(
  "desired_concurrent_number"="2",
  "max_batch_interval" = "20",
  "max_batch_rows" = "300000",
  "max_batch_size" = "209715200",
  "strict_mode" = "false",
  "format" = "json",
  "jsonpaths" = "[\"$.category\",\"$.author\",\"$.price\",\"$.timestamp\"]",
  "strip_outer_array" = "true"
)
FROM KAFKA
(
  "kafka_broker_list" = "test-dev-bigdata5:9092,test-dev-bigdata6:9092,test-dev-bigdata7:9092",
  "kafka_topic" = "test_doris_kafka_load",
  "property.group.id" = "test1", 
  "property.client.id" = "test1",
  "kafka_partitions" = "0",
  "kafka_offsets" = "0"
);
```

文本数据格式的示例：

```sql
CREATE ROUTINE LOAD example_db.test_job ON example_tbl
COLUMNS TERMINATED BY ",",
COLUMNS(k1,k2,source_sequence,v1,v2),
ORDER BY source_sequence
PROPERTIES
(
    "desired_concurrent_number"="3",
    "max_batch_interval" = "30",
    "max_batch_rows" = "300000",
    "max_batch_size" = "209715200"
) FROM KAFKA
(
    "kafka_broker_list" = "broker1:9092,broker2:9092,broker3:9092",
    "kafka_topic" = "my_topic",
    "kafka_partitions" = "0,1,2,3",
    "kafka_offsets" = "101,0,0,200"
);   
```

### 4.3 示例数据

```json
[{
		"category": "11",
		"title": "SayingsoftheCentury",
		"price": 895,
		"timestamp": 1589191587
	},
	{
		"category": "22",
		"author": "2avc",
		"price": 895,
		"timestamp": 1589191487
	},
	{
		"category": "33",
		"author": "3avc",
		"title": "SayingsoftheCentury",
		"timestamp": 1589191387
	}
] 
```

## 5.注意事项

### 5.1 例行导入作业和 ALTER TABLE 操作的关系

- 例行导入不会阻塞 SCHEMA CHANGE 和 ROLLUP 操作。但是注意如果 SCHEMA CHANGE 完成后，列映射关系无法匹配，则会导致作业的错误数据激增，最终导致作业暂停。建议通过在例行导入作业中显式指定列映射关系，以及通过增加 Nullable 列或带 Default 值的列来减少这类问题。
- 删除表的 Partition 可能会导致导入数据无法找到对应的 Partition，作业进入暂停。

### 5.2 例行导入作业和其他导入作业的关系（LOAD, DELETE, INSERT）

- 例行导入和其他 LOAD 作业以及 INSERT 操作没有冲突。
- 当执行 DELETE 操作时，对应表分区不能有任何正在执行的导入任务。所以在执行 DELETE 操作前，可能需要先暂停例行导入作业，并等待已下发的 task 全部完成后，才可以执行 DELETE。

### 5.3 例行导入作业和 DROP DATABASE/TABLE 操作的关系

当例行导入对应的 database 或 table 被删除后，作业会自动 CANCEL

### 5.4 kafka 类型的例行导入作业和 kafka topic 的关系

当用户在创建例行导入声明的 `kafka_topic` 在kafka集群中不存在时。

- 如果用户 kafka 集群的 broker 设置了 `auto.create.topics.enable = true`，则 `kafka_topic` 会先被自动创建，自动创建的 partition 个数是由**用户方的kafka集群**中的 broker 配置 `num.partitions` 决定的。例行作业会正常的不断读取该 topic 的数据。
- 如果用户 kafka 集群的 broker 设置了 `auto.create.topics.enable = false`, 则 topic 不会被自动创建，例行作业会在没有读取任何数据之前就被暂停，状态为 `PAUSED`。

所以，如果用户希望当 kafka topic 不存在的时候，被例行作业自动创建的话，只需要将**用户方的kafka集群**中的 broker 设置 `auto.create.topics.enable = true` 即可。

### 5.5 网络问题

1. 创建Routine load 任务中指定的 Broker list 必须能够被Doris服务访问
2. Kafka 中如果配置了`advertised.listeners`, `advertised.listeners` 中的地址必须能够被Doris服务访问
3. 连接kafka集群的时候建议换成Kafka集群对应的主机名

### 5.6 关于指定消费的 Partition 和 Offset

oris 支持指定 Partition 和 Offset 开始消费。新版中还支持了指定时间点进行消费的功能。这里说明下对应参数的配置关系。

有三个相关参数：

- `kafka_partitions`：指定待消费的 partition 列表，如："0, 1, 2, 3"。
- `kafka_offsets`：指定每个分区的起始offset，必须和 `kafka_partitions` 列表个数对应。如："1000, 1000, 2000, 2000"
- `property.kafka_default_offset`：指定分区默认的起始offset

