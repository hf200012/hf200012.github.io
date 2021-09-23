---
layout: post
title: "Apache Doris 数据导入总览"
date: 2021-09-22
description: "Apache Doris 数据导入总览"
tag: Apache Doris
---
# Apache Doris 数据导入总览

# 1.导入总览介绍

Apache Doris 的数据导入功能是提供用户将数据导入到 Doris 中，导入成功之后，用户可以通过 Mysql 客户端使用SQL对数据进行查询分析。

Doris 为满足不同场景的数据数据导入需求，提供了一下几种数据导入方式，每种数据导入方式都支持不同的数据源，存在不同的使用方式：同步，异步，每种导入方式也支持不同的数据格式：CSV，JSON，Parquet、ORC等

## 1.1 Broker Load方式

这种方式需要安装一个 Doris Broker服务，具体参照 Apache Doris 安装指南

这种方式通过 Broker 进程访问并读取外部数据源（如 HDFS，）导入到 Doris。用户通过 Mysql 协议，通过 Doris SQL 语句的方式提交导入作业后，异步执行。通过 `SHOW LOAD` 命令查看导入结果。

这种方式大数据量的数据迁移使用。

## 1.2 Stream Load 方式

用户通过 HTTP 协议提交请求并携带原始数据（可以是文件，也可以是内存数据）创建导入。主要用于快速将本地文件或数据流中的数据导入到 Doris。导入命令同步返回导入结果，这种导入方式支持两种格式的数据CVS和JSON，

通过 `SHOW STREAM LOAD`方式来查看Stream load作业情况，这个要最新版本里才支持（百度发行版：0.14.13、apache 0.15版本及以后版本）

这是一种同步的数据导入方式

## 1.3 Insert 方式

这种导入方式和 MySQL 中的 Insert 语句类似，Apache Doris 提供 `INSERT INTO tbl SELECT ...;` 的方式从 Doris 的表（或者ODBC方式的外表）中读取数据并导入到另一张表。或者通过 `INSERT INTO tbl VALUES(...);` 插入单条数据，单条插入方式不建议在生产和测试环境中使用，只是演示使用。

`INSERT INTO tbl SELECT …`这种方式一般是在Doris内部对数据进行加工处理，生成中间汇总表，或者在Doris内部对数据进行ETL操作使用

这种方式是一种同步的数据导入方式

## 1.4 Routine Load方式

这种方式是以Kafka为数据源，从Kafka中读取数据并导入到Doris对应的数据表中，用户通过 Mysql 客户端提交 Routine Load数据导入作业，Doris 会在生成一个常驻线程，不间断的从Kafka中读取数据并存储在对应Doris表中，并自动维护Kafka Offset位置。

通过`SHOW ROUTINE LOAD`来查看Routine load作业情况

这是一种同步的数据导入方式。

## 1.5 Spark Load方式

Spark load 通过借助于外部的 Spark 计算资源实现对导入数据的预处理，提高 Doris 大数据量的导入性能并且节省 Doris 集群的计算资源。主要用于初次迁移，大数据量导入 Doris 的场景。

这种方式需要借助于Broker服务。

Spark load 是一种异步导入方式，用户需要通过 MySQL 协议创建 Spark 类型导入任务，并通过 `SHOW LOAD` 查看导入结果。

## 1.6 S3协议直接导入

这种导入类似于Broker Load，通过S3协议直接导入数据

# 2.导入基本原理

## 2.1 角色介绍

1.  Frontend （FE）： 负责Doris 元数据管理及节点调度，在导入流程中主要负责导入规划生成及导入任务调度。
2. Backend （BE）：主要负责Doris数据存储及计算，在导入流程中主要负责数据的ETL加工及数据存储
3. Broker Load ：Broker服务为一个独立的无状态进程。封装了文件系统（例如HDFS等）接口，提供 Doris 读取远端存储系统中文件的能力。
4. 导入作业：导入作业读取用户提交的源数据，转换或清洗后，将数据导入到 Doris 系统中。导入完成后，数据即可被用户查询到。
5. Label：每个导入作业都有一个 Label。这个Label 在一个数据库内唯一，可由用户指定或系统自动生成，用于标识一个导入作业。相同的 Label 仅可用于一个成功的导入作业。
6. MySQL 协议/HTTP 协议：Doris 提供两种访问协议接口。 MySQL 协议和 HTTP 协议。部分导入方式使用 MySQL 协议接口提交作业，部分导入方式使用 HTTP 协议接口提交作业。

## 2.2 作业阶段

一个导入作业分为四个阶段：PENDING、ETL，LOADING，FINISHED，前三个阶段是非必须阶段，可选。前两个阶段在作业没有完成之前都可以取消（CANCELLED）

1. PENDING：这阶段是可选阶段，该阶段只有 Broker Load 才有。Broker Load 被用户提交后会短暂停留在这个阶段，直到被 FE 中的 Scheduler 调度。 其中 Scheduler 的调度间隔为5秒
2. ETL（非必须）： 该阶段在版本 0.10.0(包含) 之前存在，主要是用于将原始数据按照用户声明的方式进行变换，并且过滤不满足条件的原始数据。在 0.10.0 后的版本，ETL 阶段不再存在，其中数据 transform 的工作被合并到 LOADING 阶段
3. LOADING ： 该阶段在版本 0.10.0（包含）之前主要用于将变换后的数据推到对应的 BE 存储中。在 0.10.0 后的版本，该阶段先对数据进行清洗和变换，然后将数据发送到 BE 存储中。当所有导入数据均完成导入后，进入等待生效过程，此时 Load job 依旧是 LOADING。
4. FINISHED ： 在 Load job 涉及的所有数据均生效后，Load job 的状态变成 FINISHED。FINISHED 后导入的数据均可查询。
5. CANCELLED： 在作业 FINISHED 之前，作业都可能被取消并进入 CANCELLED 状态。如用户手动取消，或导入出现错误等。CANCELLED 也是 Load Job 的最终状态，不可被再次执行。

上面四个阶段，除了 PENDING 到 LOADING 阶段是 调度器（Scheduler ）轮训调度的，其他阶段之前的转移都是回调机制实现。

# 3. 数据导入的原子性保证

Doris 对所有导入方式提供原子性保证。既保证同一个导入作业内的数据，原子生效。不会出现仅导入部分数据的情况。

同时，每一个导入作业都有一个由用户指定或者系统自动生成的 Label。Label 在一个 Database 内唯一。当一个 Label 对应的导入作业成功后，不可再重复使用该 Label 提交导入作业。如果 Label 对应的导入作业失败，则可以重复使用。

用户可以通过 Label 机制，来保证 Label 对应的数据最多被导入一次，即At-Most-Once 语义

# 4. Doris数据导入最佳实践

我们在使用Doris数据导入的时候，一般都是程序接入的方式，这样可以保证数据会定期的被加载到Doris中，下面给出集中最佳实践：

1. 选择合适的导入方式：根据数据源所在位置选择导入方式。例如：如果原始数据存放在 HDFS 上，则使用 Broker load 导入。
2. 确定导入方式的协议：如果选择了 Broker load 导入方式，则外部系统需要能使用 MySQL 协议定期提交和查看导入作业。
3. 确定导入方式的类型：导入方式为同步或异步。比如 Broker load 为异步导入方式，则外部系统在提交创建导入后，必须调用查看导入命令，根据查看导入命令的结果来判断导入是否成功。
4. 制定 Label 生成策略：Label 生成策略需满足，每一批次数据唯一且固定的原则。这样 Doris 就可以保证 At-Most-Once。
5. 程序自身保证 At-Least-Once：外部系统需要保证自身的 At-Least-Once，这样就可以保证导入流程的 Exactly-Once。
6. 如果是ODBC外部数据源或者Doris内部数据加工，这种建议采用`INSERT INTO tbl SELECT …`，然后可以通过外部任务调度器（比如：DolphinScheduler），定时的对导入及数据加工任务进行调度执行
7. 如果你要实时的从外部Kafka数据源中读取数据并加载到Doris中，这种建议使用Routine Load，需要注意的是如果数据是JSON格式数据（JSON数据不支持嵌套），这种你也可以使用Stream load方式来完成数据导入。如果数据量比较大，可以借助以Flink或者Spark集群对数据进行做一些预处理，然后通过Stream Load方式导入到Doris中，大大提高数据导入效率。

# 5.总结

以上我们介绍了 Doris 数据导入的几种导入方式及数据导入的原子性保障机制，最后给出了Doris数据导入的最佳实践，后面我们将具体介绍每种数据导入的具体使用方式及相关参数说明。