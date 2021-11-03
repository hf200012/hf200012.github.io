---
layout: post
title: "Apache Doris 数据备份及恢复"
date: 2021-11-03
description: "Apache Doris 数据备份及恢复"
tag: Apache Doris
---
# Apache Doris 数据备份及恢复

Apache Doris 支持将当前数据以文件的形式，通过 broker 备份到远端存储系统中，之后可以通过恢复命令，从远端存储系统中将数据恢复到任意 Doris 集群。通过这个功能，Doris 可以支持将数据定期的进行快照备份。也可以通过这个功能，在不同集群间进行数据迁移。

使用该功能，需要部署对应远端存储的 broker，如 HDFS 等。可以通过 SHOW BROKER; 查看当前部署的 broker

Broker 是 Doris 集群中一种可选进程，主要用于支持 Doris读写远端存储上的文件和目录，具体请参考 [Broker文章节]

>注意： 该功能需要 Doris 版本 0.8.2+

## 1.原理说明

### 1.1 备份（Backup）

备份操作是将指定表或分区的数据，直接以 Doris 存储的文件的形式，上传到远端仓库中进行存储。当用户提交 Backup 请求后，系统内部会做如下操作：

1. 快照及快照上传

   快照阶段会对指定的表或分区数据文件进行快照。之后，备份都是对快照进行操作。在快照之后，对表进行的更改、导入等操作都不再影响备份的结果。快照只是对当前数据文件产生一个硬链，耗时很少。快照完成后，会开始对这些快照文件进行逐一上传。快照上传由各个 Backend 并发完成。

2. 元数据准备及上传

   数据文件快照上传完成后，Frontend 会首先将对应元数据写成本地文件，然后通过 broker 将本地元数据文件上传到远端仓库。完成最终备份作业。

### 1.2 恢复（Restore）

恢复操作需要指定一个远端仓库中已存在的备份，然后将这个备份的内容恢复到本地集群中。当用户提交 Restore 请求后，系统内部会做如下操作：

1. 在本地创建对应的元数据

   这一步首先会在本地集群中，创建恢复对应的表分区等结构。创建完成后，该表可见，但是不可访问。

2. 本地snapshot

   这一步是将上一步创建的表做一个快照。这其实是一个空快照（因为刚创建的表是没有数据的），其目的主要是在 Backend 上产生对应的快照目录，用于之后接收从远端仓库下载的快照文件。

3. 下载快照

   远端仓库中的快照文件，会被下载到对应的上一步生成的快照目录中。这一步由各个 Backend 并发完成。

4. 生效快照

   快照下载完成后，我们要将各个快照映射为当前本地表的元数据。然后重新加载这些快照，使之生效，完成最终的恢复作业

## 2.最佳实践

### 2.1 备份建议

当前Apache Doris 支持最小分区（Partition）粒度的全量备份（目前不支持增量备份，在未来版本可能会支持支持）。如果需要对数据进行定期备份，首先需要在建表时，合理的规划表的分区及分桶，比如按时间进行分区。然后在之后的运行过程中，按照分区粒度进行定期的数据备份，避免全表备份耗费太多资源及时间。

### 2.2 数据迁移建议

如果用户需要将当前Doris集群的数据迁移到另外一个新的Doris集群，用户可以先将数据备份到远端仓库，再通过远端仓库将数据恢复到另一个集群，完成数据迁移。因为数据备份是通过快照的形式完成的，所以，在备份作业的快照阶段之后的新的导入数据，是不会备份的。因此，在快照完成后，到恢复作业完成这期间，在原集群上导入的数据，都需要在新集群上同样导入一遍。

建议在迁移完成后，对新旧两个集群并行导入一段时间。完成数据和业务正确性校验后，再将业务迁移到新的集群

## 3. 数据备份及恢复说明

1. 备份恢复相关的操作目前只允许拥有 **ADMIN** 权限的用户执行。
2. 一个 Database 内，只允许有一个正在执行的备份或恢复作业。
3. 备份和恢复都支持最小分区（Partition）级别的操作，当表的数据量很大时，建议按分区分别执行，以降低失败重试的代价。
4. 因为备份恢复操作，操作的都是实际的数据文件。所以当一个表的分片过多，或者一个分片有过多的小版本时，可能即使总数据量很小，依然需要备份或恢复很长时间。用户可以通过 `SHOW PARTITIONS FROM table_name;` 和 `SHOW TABLET FROM table_name;` 来查看各个分区的分片数量，以及各个分片的文件版本数量，来预估作业执行时间。文件数量对作业执行的时间影响非常大，所以建议在建表时，合理规划分区分桶，以避免过多的分片。
5. 当通过 `SHOW BACKUP` 或者 `SHOW RESTORE` 命令查看作业状态时。有可能会在 `TaskErrMsg` 一列中看到错误信息。但只要 `State` 列不为 `CANCELLED`，则说明作业依然在继续。这些 Task 有可能会重试成功。当然，有些 Task 错误，也会直接导致作业失败。
6. 如果恢复作业是一次覆盖操作（指定恢复数据到已经存在的表或分区中），那么从恢复作业的 `COMMIT` 阶段开始，当前集群上被覆盖的数据有可能不能再被还原。此时如果恢复作业失败或被取消，有可能造成之前的数据已损坏且无法访问。这种情况下，只能通过再次执行恢复操作，并等待作业完成。因此，我们建议，如无必要，尽量不要使用覆盖的方式恢复数据，除非确认当前数据已不再使用

## 4.备份（Backup）操作

和备份恢复功能相关的命令如下。以下命令，都可以通过 mysql-client 连接 Doris 后，使用 `help cmd;` 的方式查看详细帮助。

### 4.1 CREATE REPOSITORY

创建一个远端仓库路径，用于备份或恢复。该命令需要借助 Broker 进程访问远端存储，不同的 Broker 需要提供不同的参数，具体请参阅 [Broker文档](https://doris.apache.org/master/zh-CN/administrator-guide/broker.html)，也可以直接通过S3 协议备份到支持AWS S3协议的远程存储上去，

示例：

```sql
CREATE REPOSITORY `hdfs_ods_dw_backup` 
WITH BROKER `broker_name_02` 
ON LOCATION "hdfs://gaia-pro-cdh-namenode1:8020/tmp/doris_backup" 
PROPERTIES (
  "username" = "",
  "password" = "" 
)
```

这里我们是创建一个HDFS的远端仓库，Doris支持多种远端存储：HDFS，BOS，S3等

### 4.2 BACKUP

执行一次备份操作

语法：

```sql
BACKUP SNAPSHOT [db_name].{snapshot_name}
TO `repository_name`
ON (
   `table_name` [PARTITION (`p1`, ...)],
    ...
)
PROPERTIES ("key"="value", ...);
```

> 注意：Doris仅支持备份 OLAP 类型的表，不支持ODBC等其他类型的外表
>
> 1. 同一数据库下只能有一个正在执行的 BACKUP 或 RESTORE 任务。
> 2. ON 子句中标识需要备份的表和分区。如果不指定分区，则默认备份该表的所有分区。
> 3. Doris支持一次同时备份多张表，但是不支持整库备份，如果你一个库下面表不多，每次备份的数据量不大可以，将所有表列出达到整库备份的效果。
> 4. PROPERTIES 目前支持以下属性：
>         "type" = "full"：表示这是一次全量更新（默认）。
>          "timeout" = "3600"：任务超时时间，默认为一天。单位秒

下面这个示例，对两张表进行全量备份

```sql
BACKUP SNAPSHOT ods_dw.snapshot_20211103
TO hdfs_ods_dw_backup
ON (
     table_1,
     table_2
)
PROPERTIES ("type" = "full")
```

这个示例是实现对数据库 example_db 下，表 example_tbl 的 p1, p2 分区，以及表 example_tbl2 到仓库 example_repo 中：

```sql
 BACKUP SNAPSHOT example_db.snapshot_label2
 TO example_repo
 ON 
 (
     example_tbl PARTITION (p1,p2),
     example_tbl2
 );
```

### 4.3 查看备份任务

你可以通过 `SHOW BACKUP [FROM db_name]`的方式查看指定数据库下的备份任务，会返回备份作业的相关信息。

具体返回信息的字段说明如下：

- JobId：本次备份作业的 id。
- SnapshotName：用户指定的本次备份作业的名称（Label）。
- DbName：备份作业对应的 Database。
- State：备份作业当前所在阶段：
  - PENDING：作业初始状态。
  - SNAPSHOTING：正在进行快照操作。
  - UPLOAD_SNAPSHOT：快照结束，准备上传。
  - UPLOADING：正在上传快照。
  - SAVE_META：正在本地生成元数据文件。
  - UPLOAD_INFO：上传元数据文件和本次备份作业的信息。
  - FINISHED：备份完成。
  - CANCELLED：备份失败或被取消。
- BackupObjs：本次备份涉及的表和分区的清单。
- CreateTime：作业创建时间。
- SnapshotFinishedTime：快照完成时间。
- UploadFinishedTime：快照上传完成时间。
- FinishedTime：本次作业完成时间。
- UnfinishedTasks：在 SNAPSHOTTING，UPLOADING 等阶段，会有多个子任务在同时进行，这里展示的当前阶段，未完成子任务的 task id。
- TaskErrMsg：如果有子任务执行出错，这里会显示对应子任务的错误信息。
- Status：用于记录在整个作业过程中，可能出现的一些状态信息。
- Timeout：作业的超时时间，单位是秒。

### 4.4 查看远端仓库镜像

该语句用于查看仓库中已存在的备份，语法如下：

```sql
SHOW SNAPSHOT ON `repo_name`
        [WHERE SNAPSHOT = "snapshot" [AND TIMESTAMP = "backup_timestamp"]];
```

#### 4.4.1 示例

1. 查看仓库 example_repo 中已有的备份：

    ```sql
    SHOW SNAPSHOT ON example_repo; 
    ```

2. 仅查看仓库 example_repo 中名称为 backup1 的备份：
   
    ```sql
    SHOW SNAPSHOT ON example_repo WHERE SNAPSHOT = "backup1";
    ```
    
3. 查看仓库 example_repo 中名称为 backup1 的备份，时间版本为 "2021-05-05-15-34-26" 的详细信息：

    ```sql
    SHOW SNAPSHOT ON example_repo
        WHERE SNAPSHOT = "backup1" AND TIMESTAMP = "2021-05-05-15-34-26";
    ```

#### 4.4.2 返回字段说明

查看远端仓库中已存在的备份。

- Snapshot：备份时指定的该备份的名称（Label）。
- Timestamp：备份的时间戳。
- Status：该备份是否正常。

### 4.5 取消Backup

下面的语句用于取消一个正在执行的备份作业

```sql
CANCEL BACKUP FROM db_name;
```

示例：

 取消 example_db 下的 BACKUP 任务      

```sql
CANCEL BACKUP FROM example_db;
```

## 5.恢复（Restore）操作

该语句用于将之前通过 BACKUP 命令备份的数据，恢复到指定数据库下。该命令为异步操作。提交成功后，需通过 SHOW RESTORE 命令查看进度。

> 注意：
>
> 1. 仅支持恢复 OLAP 类型的表
> 2. 支持一次恢复多张表，这个需要和你对应的备份里的表一致

### 5.1 使用语法：

```sql
RESTORE SNAPSHOT [db_name].{snapshot_name}
FROM `repository_name`
ON (
  `table_name` [PARTITION (`p1`, ...)] [AS `tbl_alias`],
  ...
)
PROPERTIES ("key"="value", ...);
```

> 说明：
>
> 1. 同一数据库下只能有一个正在执行的 BACKUP 或 RESTORE 任务。
> 2.  ON 子句中标识需要恢复的表和分区。如果不指定分区，则默认恢复该表的所有分区。所指定的表和分区必须已存在于仓库备份中
> 3.  可以通过 AS 语句将仓库中备份的表名恢复为新的表。但新表名不能已存在于数据库中。分区名称不能修改。
> 4. 可以将仓库中备份的表恢复替换数据库中已有的同名表，但须保证两张表的表结构完全一致。表结构包括：表名、列、分区、Rollup等等。
> 5. 可以指定恢复表的部分分区，系统会检查分区 Range 或者 List 是否能够匹配。
> 6. PROPERTIES 目前支持以下属性：
>                 "backup_timestamp" = "2018-05-04-16-45-08"：指定了恢复对应备份的哪个时间版本，必填。该信息可以通过 `SHOW SNAPSHOT ON repo;` 语句获得。
>                 "replication_num" = "3"：指定恢复的表或分区的副本数。默认为3。若恢复已存在的表或分区，则副本数必须和已存在表或分区的副本数相同。同时，必须有足够的 host 容纳多个副本。
>                 "timeout" = "3600"：任务超时时间，默认为一天。单位秒。
>                 "meta_version" = 40：使用指定的 meta_version 来读取之前备份的元数据。注意，该参数作为临时方案，仅用于恢复老版本 Doris 备份的数据。最新版本的备份数据中已经包含 meta version，无需再指定。

### 5.2 使用示例

1. 从 example_repo 中恢复备份 snapshot_1 中的表 backup_tbl 到数据库 example_db1，时间版本为 "2021-05-04-16-45-08"。恢复为 1 个副本：

```sql
RESTORE SNAPSHOT example_db1.`snapshot_1`
FROM `example_repo`
ON ( `backup_tbl` )
PROPERTIES
(
   "backup_timestamp"="2021-05-04-16-45-08",
    "replication_num" = "1"
);
```

2. 从 example_repo 中恢复备份 snapshot_2 中的表 backup_tbl 的分区 p1,p2，以及表 backup_tbl2 到数据库 example_db1，并重命名为 new_tbl，时间版本为 "2021-05-04-17-11-01"。默认恢复为 3 个副本：
   
```sql
RESTORE SNAPSHOT example_db1.`snapshot_2`
FROM `example_repo`
ON
(
  `backup_tbl` PARTITION (`p1`, `p2`),
   `backup_tbl2` AS `new_tbl`
)
PROPERTIES
(
   "backup_timestamp"="2021-05-04-17-11-01"
);
```

### 5.3 查看恢复任务

可以通过下面的语句查看数据恢复的情况

```sql
SHOW RESTORE [FROM db_name]  
```

  执行返回的字段信息

- JobId：本次恢复作业的 id。
- Label：用户指定的仓库中备份的名称（Label）。
- Timestamp：用户指定的仓库中备份的时间戳。
- DbName：恢复作业对应的 Database。
- State：恢复作业当前所在阶段：
  - PENDING：作业初始状态。
  - SNAPSHOTING：正在进行本地新建表的快照操作。
  - DOWNLOAD：正在发送下载快照任务。
  - DOWNLOADING：快照正在下载。
  - COMMIT：准备生效已下载的快照。
  - COMMITTING：正在生效已下载的快照。
  - FINISHED：恢复完成。
  - CANCELLED：恢复失败或被取消。
- AllowLoad：恢复期间是否允许导入。
- ReplicationNum：恢复指定的副本数。
- RestoreObjs：本次恢复涉及的表和分区的清单。
- CreateTime：作业创建时间。
- MetaPreparedTime：本地元数据生成完成时间。
- SnapshotFinishedTime：本地快照完成时间。
- DownloadFinishedTime：远端快照下载完成时间。
- FinishedTime：本次作业完成时间。
- UnfinishedTasks：在 `SNAPSHOTTING`，`DOWNLOADING`, `COMMITTING` 等阶段，会有多个子任务在同时进行，这里展示的当前阶段，未完成的子任务的 task id。
- TaskErrMsg：如果有子任务执行出错，这里会显示对应子任务的错误信息。
- Status：用于记录在整个作业过程中，可能出现的一些状态信息。
- Timeout：作业的超时时间，单位是秒

### 5.4 取消Restore作业

下面的语句用于取消一个正在执行数据恢复的作业，

```sql
CANCEL RESTORE FROM db_name;
```

> 当取消处于 COMMIT 或之后阶段的恢复左右时，可能导致被恢复的表无法访问。此时只能通过再次执行恢复作业进行数据恢复

示例：

取消 example_db 下的 RESTORE 任务。

```sql
CANCEL RESTORE FROM example_db;
```

## 删除远端仓库

该语句用于删除一个已创建的仓库。仅 root 或 superuser 用户可以删除仓库。这里的用户是指Doris的用户
 语法：

```sql
   DROP REPOSITORY `repo_name`;         
```

>说明：
>
>删除仓库，仅仅是删除该仓库在 Doris 中的映射，不会删除实际的仓库数据。删除后，可以再次通过指定相同的 broker 和 LOCATION 映射到该仓库。
示例：

1. 删除名为 hdfs_repo 的仓库：

   ```sql
   DROP REPOSITORY `hdfs_repo`;
   ```

   
