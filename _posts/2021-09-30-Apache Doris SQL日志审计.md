---
layout: post
title: "Apache Doris SQL 日志审计"
date: 2021-09-31
description: "Apache Doris SQL 日志审计"
tag: Apache Doris
---
# Apache Doris SQL 日志审计

# 1. 介绍

Doris 的审计日志插件是在 FE 的插件框架基础上开发的。是一个可选插件。用户可以在运行时安装或卸载这个插件。

该插件可以将 FE 的审计日志定期的导入到指定 Doris 集群中，以方便用户通过 SQL 对审计日志进行查看和分析。

这里的数据其实是Doris FE log目录下的 `fe.audit.log ` 文件中的数据

# 2. 安装部署

## 2.1 编译

在 Doris 代码目录下执行 

`sh build_plugin.sh`

编译完成后会在 `fe_plugins/output` 目录下得到 `auditloader.zip` 文件

## 2.2 安装

您可以将这个文件放置在一个 http 服务器上，或者拷贝`auditloader.zip`(或者解压`auditloader.zip`)到所有 FE 的指定目录下。这里我们使用后者。

FE的插件框架当前是实验性功能，Doris中默认关闭，需要在FE的配置文件中，增加`plugin_enable = true`启用plugin框架

### 2.2.1 配置信息

plugin.conf 配置信息：

```
# 批量的最大大小，默认为 50MB
max_batch_size=52428800

# 批量加载的最大间隔，默认为 60 秒
max_batch_interval_sec=60

# 加载审计的 Doris FE 主机，默认为 127.0.0.1:8030。
# 这应该是Stream load 的主机端口
frontend_host_port=127.0.0.1:8030

# 审计表的数据库
database=doris_audit_db__

# 审计表名，保存审计数据
table=doris_audit_tbl__

# 用来连接doris的用户. 此用户必须对审计表具有 LOAD_PRIV 权限.
user=root

# 用来连接doris的用户密码
password=
```

安装插件前，需要创建之前在 `plugin.conf` 中指定的审计数据库和表。其中建表语句如下：

这里的表名，你可以根据自己情况修改，修改后要注意修改 `plugin.conf` 中的内容

```sql
create table doris_audit_tbl__
(
    query_id varchar(48) comment "查询唯一ID",
    time datetime not null comment "查询开始时间",
    client_ip varchar(32) comment "查询客户端IP",
    user varchar(64) comment "查询用户名",
    db varchar(96) comment "查询的数据库",
    state varchar(8) comment "查询状态： EOF, ERR, OK",
    query_time bigint comment "查询执行时间（毫秒）",
    scan_bytes bigint comment "查询扫描的字节数",
    scan_rows bigint comment "查询扫描的记录行数",
    return_rows bigint comment "查询返回的结果记录数",
    stmt_id int comment "SQL语句的增量ID",
    is_query tinyint comment "这个是否是查询： 1 or 0",
    frontend_ip varchar(32) comment "执行这个语句的FE IP",
    stmt varchar(5000) comment "原始语句，如果超过 5000 字节，则进行修剪"
) engine=OLAP
duplicate key(query_id, time, client_ip)
partition by range(time) ()
distributed by hash(query_id) buckets 1
properties(
    "dynamic_partition.time_unit" = "DAY",
    "dynamic_partition.start" = "-30",
    "dynamic_partition.end" = "3",
    "dynamic_partition.prefix" = "p",
    "dynamic_partition.buckets" = "1",
    "dynamic_partition.enable" = "true",
    "replication_num" = "3"
);
```

其中 `dynamic_partition` 属性根据自己的需要，选择审计日志保留的天数。

之后，连接到 Doris 后使用 `INSTALL PLUGIN` 命令完成安装。安装成功后，可以通过 `SHOW PLUGINS` 看到已经安装的插件，并且状态为 `INSTALLED`。

完成后，插件会不断的以指定的时间间隔将审计日志插入到这个表中。

### 2.2.1 安装SQL语法

```sql
INSTALL PLUGIN FROM [source] [PROPERTIES ("key"="value", ...)]
```

source 支持三种类型：

1. 指向一个 zip 文件的绝对路径。
2. 指向一个插件目录的绝对路径。
3. 指向一个 http 或 https 协议的 zip 文件下载路径

PROPERTIES 支持设置插件的一些配置,如设置zip文件的md5sum的值等

### 2.2.2 安装示例

1. 安装一个本地 zip 文件插件：
	
	```sql
	INSTALL PLUGIN FROM "/home/users/doris/auditdemo.zip";
	```
	
2. 安装一个本地目录中的插件：

    ```sql
    INSTALL PLUGIN FROM "/home/users/doris/auditdemo/";
    ```

3. 下载并安装一个插件：

    ```sql
    INSTALL PLUGIN FROM "http://mywebsite.com/plugin.zip";
    ```
    
4. 下载并安装一个插件,同时设置了zip文件的md5sum的值：   
   
    ```sql
    INSTALL PLUGIN FROM "http://mywebsite.com/plugin.zip" PROPERTIES("md5sum" = "73877f6029216f4314d712086a146570");
    ```

安装完成以后，你就可以在你创建的审计数据库中的审计表里看到SQL操作审计日志数据了
