---
layout: post
title: "Apache Doris 动态分区介绍及使用方法"
date: 2021-09-28
description: "Apache Doris 动态分区介绍及使用方法"
tag: Apache Doris
---
# Apache Doris 动态分区介绍及使用方法

# 1. 介绍

在某些使用场景下，用户会将表按照天进行分区划分，每天定时执行例行任务，这时需要使用方手动管理分区，否则可能由于使用方没有创建分区导致数据导入失败，这给使用方带来了额外的维护成本。

通过动态分区功能，用户可以在建表时设定动态分区的规则。FE 会启动一个后台线程，根据用户指定的规则创建或删除分区。用户也可以在运行时对现有规则进行变更

动态分区是在 Doris 0.12 版本中引入的新功能。旨在对表级别的分区实现生命周期管理(TTL)，减少用户的使用负担。

目前实现了动态添加分区及动态删除分区的功能。

从0.15.0以后版本开始支持列表分区，动态创建历史分分区功能。

# 2. 使用方式

动态分区的规则可以在建表时指定，或者在运行时进行修改。当前仅支持对单分区列的分区表设定动态分区规则。

## 2.1 建表时指定

语法：

```sql
CREATE TABLE tbl1
(...)
PROPERTIES
(
    "dynamic_partition.prop1" = "value1",
    "dynamic_partition.prop2" = "value2",
    ...
)
```

示例：

```sql
CREATE TABLE user_log_1
(
    user_id VARCHAR(20),
    ts datetime,
    item_id VARCHAR(30),
    category_id VARCHAR(30),
    behavior VARCHAR(30)
)
DUPLICATE KEY(user_id, ts)
PARTITION BY RANGE(ts) (
    PARTITION P_20210926 VALUES [('2021-09-26'), ('2021-09-27')),
    PARTITION P_20210927 VALUES [('2021-09-27'), ('2021-09-28')),
)
DISTRIBUTED BY HASH(user_id)
PROPERTIES
(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-7",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "32"
);
```

## 2.2 运行时修改

语法：

```sql
ALTER TABLE tbl1 SET
(
    "dynamic_partition.prop1" = "value1",
    "dynamic_partition.prop2" = "value2",
    ...
)
```

示例：

```sql
ALTER TABLE tbl1 user_log_1
(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-7",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "32"
)
```

## 2.3 动态分区规则参数

动态分区的规则参数都以 `dynamic_partition.` 为前缀：

- `dynamic_partition.enable`

  是否开启动态分区特性。可指定为 `TRUE` 或 `FALSE`。如果不填写，默认为 `TRUE`。如果为 `FALSE`，则 Doris 会忽略该表的动态分区规则。

- `dynamic_partition.time_unit`

  动态分区调度的单位。可指定为 `HOUR`、`DAY`、`WEEK`、`MONTH`。分别表示按天、按星期、按月进行分区创建或删除。

  当指定为 `HOUR` 时，动态创建的分区名后缀格式为 `yyyyMMddHH`，例如`2020032501`。小时为单位的分区列数据类型不能为 DATE。

  当指定为 `DAY` 时，动态创建的分区名后缀格式为 `yyyyMMdd`，例如`20200325`。

  当指定为 `WEEK` 时，动态创建的分区名后缀格式为`yyyy_ww`。即当前日期属于这一年的第几周，例如 `2020-03-25` 创建的分区名后缀为 `2020_13`, 表明目前为2020年第13周。

  当指定为 `MONTH` 时，动态创建的分区名后缀格式为 `yyyyMM`，例如 `202003`。

- `dynamic_partition.time_zone`

  动态分区的时区，如果不填写，则默认为当前机器的系统的时区，例如 `Asia/Shanghai`，如果想获取当前支持的时区设置，可以参考 `https://en.wikipedia.org/wiki/List_of_tz_database_time_zones`。

- `dynamic_partition.start`

  动态分区的起始偏移，为负数。根据 `time_unit` 属性的不同，以当天（星期/月）为基准，分区范围在此偏移之前的分区将会被删除。如果不填写，则默认为 `-2147483648`，即不删除历史分区。

- `dynamic_partition.end`

  动态分区的结束偏移，为正数。根据 `time_unit` 属性的不同，以当天（星期/月）为基准，提前创建对应范围的分区。

- `dynamic_partition.prefix`

  动态创建的分区名前缀。

- `dynamic_partition.buckets`

  动态创建的分区所对应的分桶数量。

- `dynamic_partition.replication_num`

  动态创建的分区所对应的副本数量，如果不填写，则默认为该表创建时指定的副本数量。

- `dynamic_partition.start_day_of_week`

  当 `time_unit` 为 `WEEK` 时，该参数用于指定每周的起始点。取值为 1 到 7。其中 1 表示周一，7 表示周日。默认为 1，即表示每周以周一为起始点。

- `dynamic_partition.start_day_of_month`

  当 `time_unit` 为 `MONTH` 时，该参数用于指定每月的起始日期。取值为 1 到 28。其中 1 表示每月1号，28 表示每月28号。默认为 1，即表示每月以1号位起始点。暂不支持以29、30、31号为起始日，以避免因闰年或闰月带来的歧义。

- `dynamic_partition.create_history_partition`

  默认为 false。当置为 true 时，Doris 会自动创建所有分区，具体创建规则见下文。同时，FE 的参数 `max_dynamic_partition_num` 会限制总分区数量，以避免一次性创建过多分区。当期望创建的分区个数大于 `max_dynamic_partition_num` 值时，操作将被禁止。

  当不指定 `start` 属性时，该参数不生效。

- `dynamic_partition.history_partition_num`

  当 `create_history_partition` 为 `true` 时，该参数用于指定创建历史分区数量。默认值为 -1， 即未设置。

- `dynamic_partition.hot_partition_num`

  指定最新的多少个分区为热分区。对于热分区，系统会自动设置其 `storage_medium` 参数为SSD，并且设置 `storage_cooldown_time`。

  `hot_partition_num` 是往前 n 天和未来所有分区

  我们举例说明。假设今天是 2021-05-20，按天分区，动态分区的属性设置为：hot_partition_num=2, end=3, start=-3。则系统会自动创建以下分区，并且设置 `storage_medium` 和 `storage_cooldown_time` 

  参数：
  
  ```text
  p20210517：["2021-05-17", "2021-05-18") storage_medium=HDD storage_cooldown_time=9999-12-31 23:59:59
  p20210518：["2021-05-18", "2021-05-19") storage_medium=HDD storage_cooldown_time=9999-12-31 23:59:59
  p20210519：["2021-05-19", "2021-05-20") storage_medium=SSD storage_cooldown_time=2021-05-21 00:00:00
  p20210520：["2021-05-20", "2021-05-21") storage_medium=SSD storage_cooldown_time=2021-05-22 00:00:00
  p20210521：["2021-05-21", "2021-05-22") storage_medium=SSD storage_cooldown_time=2021-05-23 00:00:00
  p20210522：["2021-05-22", "2021-05-23") storage_medium=SSD storage_cooldown_time=2021-05-24 00:00:00
  p20210523：["2021-05-23", "2021-05-24") storage_medium=SSD storage_cooldown_time=2021-05-25 00:00:00
  ```

### 2.3.1 创建历史分区规则

Doris 0.14.0及之前的版本不支持创建历史分区，不过百度发布的0.14.12版本支持，社区版本要等到0.15发布才会支持。

当 `create_history_partition` 为 `true`，即开启创建历史分区功能时，Doris 会根据 `dynamic_partition.start` 和 `dynamic_partition.history_partition_num` 来决定创建历史分区的个数。

假设需要创建的历史分区数量为 `expect_create_partition_num`，根据不同的设置具体数量如下：

1. `create_history_partition` = `true`
   - `dynamic_partition.history_partition_num` 未设置，即 -1.
     `expect_create_partition_num` = `end` - `start`;
   - `dynamic_partition.history_partition_num` 已设置
     `expect_create_partition_num` = `end` - max(`start`, `-histoty_partition_num`);
2. `create_history_partition` = `false`
   不会创建历史分区，`expect_create_partition_num` = `end` - 0;

当 `expect_create_partition_num` 大于 `max_dynamic_partition_num`（默认500）时，禁止创建过多分区。

**举例说明：**

1. 假设今天是 2021-09-27，按天分区，动态分区的属性设置为：`create_history_partition=true, end=3, start=-3, history_partition_num=1`，则系统会自动创建以下分区：

   ```text
   p20210926
   p20210927
   p20210928
   p20210930
   p20211001
   ```

建表语句：

```sql
CREATE TABLE user_log_1
(
    user_id VARCHAR(20),
    ts datetime,
    item_id VARCHAR(30),
    category_id VARCHAR(30),
    behavior VARCHAR(30)
)
DUPLICATE KEY(user_id, ts)
PARTITION BY RANGE(ts) (
    PARTITION P_20210926 VALUES [('2021-09-26'), ('2021-09-27')),
    PARTITION P_20210927 VALUES [('2021-09-27'), ('2021-09-28')),
)
DISTRIBUTED BY HASH(user_id)
PROPERTIES
(
    "dynamic_partition.enable" = "true",
    "dynamic_partition.create_history_partition" = "true",
    "dynamic_partition.history_partition_num" = "1",
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-3",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "32"
);
```
2. `history_partition_num=-1` 即不设置历史分区数量，其余属性与 1 中保持一直，则系统会自动创建以下分区：

   ```
   p20210924
   p20210925
   p20210926
   p20210927
   p20210928
   p20210930
   p20211001
   ```

### 2.3.2  注意事项

 动态分区使用过程中，如果因为一些意外情况导致 `dynamic_partition.start` 和 `dynamic_partition.end` 之间的某些分区丢失，那么当前时间与 `dynamic_partition.end` 之间的丢失分区会被重新创建，`dynamic_partition.start`与当前时间之间的丢失分区不会重新创建。

## 2.4 查看动态分区表调度情况

通过以下命令可以进一步查看当前数据库下，所有动态分区表的调度情况

```sql
SHOW DYNAMIC PARTITION TABLES;
```

![image-20210927165732832](/images/load/image-20210927165732832.png)

- LastUpdateTime: 最后一次修改动态分区属性的时间
- LastSchedulerTime: 最后一次执行动态分区调度的时间
- State: 最后一次执行动态分区调度的状态
- LastCreatePartitionMsg: 最后一次执行动态添加分区调度的错误信息
- LastDropPartitionMsg: 最后一次执行动态删除分区调度的错误信息

# 3. 高级操作

## 3.1 dynamic_partition_enable

是否开启 Doris 的动态分区功能。默认为 false，即关闭。该参数只影响动态分区表的分区操作，不影响普通表。可以通过修改 fe.conf 中的参数并重启 FE 生效。也可以在运行时执行以下命令生效：

MySQL 协议：

```sql
ADMIN SET FRONTEND CONFIG ("dynamic_partition_enable" = "true")
```

HTTP 协议：

```shell
curl --location-trusted -u username:password -XGET http://fe_host:fe_http_port/api/_set_config?dynamic_partition_enable=true
```

若要全局关闭动态分区，则设置此参数为 false 即可。

## 3.2 dynamic_partition_check_interval_seconds

动态分区线程的执行频率，默认为600(10分钟)，即每10分钟进行一次调度。可以通过修改 fe.conf 中的参数并重启 FE 生效。也可以在运行时执行以下命令修改：

MySQL 协议：
```sql
ADMIN SET FRONTEND CONFIG ("dynamic_partition_check_interval_seconds" = "7200")
```
HTTP 协议：

```
curl --location-trusted -u username:password -XGET http://fe_host:fe_http_port/api/_set_config?dynamic_partition_check_interval_seconds=432000
```



### 3.3 动态分区表与手动分区表相互转换动态分区表与手动分区表相互转换

对于一个表来说，动态分区和手动分区可以自由转换，但二者不能同时存在，有且只有一种状态。

### 3.4 手动分区转换为动态分区

如果一个表在创建时未指定动态分区，可以通过 `ALTER TABLE` 在运行时修改动态分区相关属性来转化为动态分区，具体示例可以通过 `HELP ALTER TABLE` 查看。

开启动态分区功能后，Doris 将不再允许用户手动管理分区，会根据动态分区属性来自动管理分区。

**注意**：如果已设定 `dynamic_partition.start`，分区范围在动态分区起始偏移之前的历史分区将会被删除。

### 3.5 动态分区转换为手动分区

通过执行 `ALTER TABLE tbl_name SET ("dynamic_partition.enable" = "false")` 即可关闭动态分区功能，将其转换为手动分区表。

关闭动态分区功能后，Doris 将不再自动管理分区，需要用户手动通过 `ALTER TABLE` 的方式创建或删除分区。
