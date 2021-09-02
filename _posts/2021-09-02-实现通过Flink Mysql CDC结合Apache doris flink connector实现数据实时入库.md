---
layout: post
title: "Flink Mysql CDC结合Doris flink connector实现数据实时入库"
date: 2021-09-02
description: "Flink Mysql CDC结合Doris flink connector实现数据实时入库"
tag: Apache Doris
---

# Flink Mysql CDC结合Doris flink connector实现数据实时入库

Apache doris通过扩展支持通过 Flink 读写 doris 数仓中的数据表，

目前 doris 支持 Flink 1.11.x ，1.12.x，1.13.x，Scala版本：2.12.x

目前Flink doris connector目前控制入库通过两个参数：

1. sink.batch.size	：每多少条写入一次，默认100条
2. sink.batch.interval ：每个多少秒写入一下，默认1秒

这两参数同时起作用，那个条件先到就触发写doris表操作，

**注意：**

这里**注意**的是要启用 http v2 版本，具体在 fe.conf 中配置 `enable_http_server_v2=true`，同时因为是通过 fe http rest api 获取 be 列表，这俩需要配置的用户有 admin 权限。

**Flink Doris Connector 编译**

在 doris 的 docker 编译环境 `apache/incubator-doris:build-env-1.2` 下进行编译，因为 1.3 下面的JDK 版本是 11，会存在编译问题。

在 extension/flink-doris-connector/ 源码目录下执行：

```
sh build.sh
```

编译成功后，会在 `output/` 目录下生成文件 `doris-flink-1.0.0-SNAPSHOT.jar`。将此文件复制到 `Flink` 的 `ClassPath` 中即可使用 `Flink-Doris-Connector`。例如，`Local` 模式运行的 `Flink`，将此文件放入 `jars/` 文件夹下。`Yarn`集群模式运行的`Flink`，则将此文件放入预部署包中。



**针对Flink 1.13.x版本适配问题**

```xml
   <properties>
        <scala.version>2.12</scala.version>
        <flink.version>1.11.2</flink.version>
        <libthrift.version>0.9.3</libthrift.version>
        <arrow.version>0.15.1</arrow.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <doris.home>${basedir}/../../</doris.home>
        <doris.thirdparty>${basedir}/../../thirdparty</doris.thirdparty>
    </properties>
```

只需要将这里的 `flink.version` 改成和你 Flink 集群版本一致，重新编辑即可

**使用示例**

通过flink cdc实现mysql binlog日志数据的消费，然后通过flink doris connector sql实时导入mysql数据到doris表数据中

这个代码已经提交到apache doris的示例代码库里

```
 org.apache.doris.demo.flink.FlinkConnectorMysqlCDCDemo
```

注意: 由于Flink doris connector jar包不在Maven中央仓库中，需要单独编译并添加到你项目的classpath中。参考Flink doris connector的编译和使用: Flink doris connector

1. 首先Mysql 要开启 binlog
   具体如何打开binlog请自行搜索或到Mysql官方文档查询
2. 安装Flink，Flink的安装和使用这里不做介绍，只是在开发环境中给出代码示例
3. 创建Mysql数据库表

```sql
 CREATE TABLE `test` (
  `id` int NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
 ) ENGINE=InnoDB AUTO_INCREMENT=19 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

4. 创建doris表

```sql
CREATE TABLE `doris_test` (
  `id` int NULL COMMENT "",
  `name` varchar(100) NULL COMMENT ""
 ) ENGINE=OLAP
 DUPLICATE KEY(`id`)
 COMMENT "OLAP"
 DISTRIBUTED BY HASH(`id`) BUCKETS 1
 PROPERTIES (
 "replication_num" = "3",
 "in_memory" = "false",
 "storage_format" = "V2"
 );
```

5. 创建Flink Mysql CDC

```java
tEnv.executeSql(
  "CREATE TABLE orders (\n" +
  "  id INT,\n" +
  "  name STRING\n" +
  ") WITH (\n" +
  "  'connector' = 'mysql-cdc',\n" +
  "  'hostname' = 'localhost',\n" +
  "  'port' = '3306',\n" +
  "  'username' = 'root',\n" +
  "  'password' = 'zhangfeng',\n" +
  "  'database-name' = 'demo',\n" +
  "  'table-name' = 'test'\n" +
  ")");
```

6. 创建Flink Doris Table 映射表

```java
 tEnv.executeSql(
  "CREATE TABLE doris_test_sink (" +
  "id INT," +
  "name STRING" +
  ") " +
  "WITH (\n" +
  "  'connector' = 'doris',\n" +
  "  'fenodes' = '10.220.146.10:8030',\n" +
  "  'table.identifier' = 'test_2.doris_test',\n" +
  "  'sink.batch.size' = '2',\n" +
  "  'username' = 'root',\n" +
  "  'password' = ''\n" +
  ")");
```

7. 执行插入操作

```sql
 tEnv.executeSql("INSERT INTO doris_test_sink select id,name from orders");
```