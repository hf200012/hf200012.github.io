---
layout: post
title: "Apache doris ODBC外表使用方式"
date: 2021-09-01 
description: "Apache doris ODBC外表使用方式"
tag:  Apache Doris
---

# Apache doris ODBC外表使用方式

## 1.概述

ODBC External Table Of Doris 提供了Doris通过数据库访问的标准接口(ODBC)来访问外部表，外部表省去了繁琐的数据导入工作，让Doris可以具有了访问各式数据库的能力，并借助Doris本身的OLAP的能力来解决外部表的数据分析问题：

1. 支持各种数据源接入Doris
2. 支持Doris与各种数据源中的表联合查询，进行更加复杂的分析操作
3. 通过insert into将Doris执行的查询结果写入外部的数据源

本文主要介绍Doris ODBC的安装使用方式

这里以最常用的Mysql为例。

## 2.ODBC驱动安装

### 2.1 安装Mysql ODBC驱动

Mysql ODBC驱动下载地址：https://dev.mysql.com/downloads/connector/odbc/

这里我们下载的是RPM安装包：mysql-connector-odbc-8.0.11-1.el7.x86_64.rpm

然后执行 

```shell
yum localinstall -y mysql-connector-odbc-8.0.11-1.el7.x86_64.rpm
```

### 2.2 配置Mysql ODBC驱动

编辑 /etc/odbcinst.ini

```shell
[MySQL]
Description=ODBC for MySQL
Driver=/usr/lib/libmyodbc5.so
Setup=/usr/lib/libodbcmyS.so
Driver64=/usr/lib64/libmyodbc5.so
Setup64=/usr/lib64/libodbcmyS.so
FileUsage=1

[MySQL ODBC 8.0 Unicode Driver]
Driver=/usr/lib64/libmyodbc8w.so
UsageCount=1

[MySQL ODBC 8.0 ANSI Driver]
Driver=/usr/lib64/libmyodbc8a.so
UsageCount=1

```

### 2.3 测试驱动

配置 /etc/odbc.ini

```shell
[mysql]
Description     = Data source MySQL
Driver          = MySQL ODBC 8.0 Driver
Server          = 192.168.1.120
Host            = 192.168.1.120
Database        = dbname
Port            = 3306
User            = root
Password        = 123456
```

一般是通过uncode 方式连接，Driver 必须是MySQL ODBC 8.0 Driver ， 

其他参数按mysql server 的设置。Database 选择已经建立好的数据库。

### 2.4 测试ODBC连接

执行:

```sql
$ isql mysql
+---------------------------------------+
| Connected!                            |
|                                       |
| sql-statement                         |
| help [tablename]                      |
| quit                                  |
|                                       |
+---------------------------------------+
SQL> select * from test limit 2;
+---------------------------------------------------+-----------+
| name                                              | age       |
+---------------------------------------------------+-----------+
| zhangfeng                                         | 20        |
| zhang                                             | 22        |
+---------------------------------------------------+-----------+
SQLRowCount returns 2
2 rows fetched
```

一切正常

## 3. 配置Doris be ODBC

Doris 使用ODBC连接mysql或者其他数据库，只需要在BE节点安装和配置ODBC即可

修改BE节点odbc配置信息

```
├── bin
│   ├── be.pid
│   ├── start_be.sh
│   └── stop_be.sh
├── conf
│   ├── be.conf
│   └── odbcinst.ini   //修改这个文件的配置信息
├── lib
│   ├── meta_tool
│   ├── palo_be
│   ├── small_file
│   ├── udf
│   └── udf-runtime
```

示例：

我这里只使用了Mysql

```
# Driver from the postgresql-odbc package
# Setup from the unixODBC package
[PostgreSQL]
Description     = ODBC for PostgreSQL
Driver          = /usr/lib/psqlodbc.so
Setup           = /usr/lib/libodbcpsqlS.so
FileUsage       = 1


# Driver from the mysql-connector-odbc package
# Setup from the unixODBC package
#[MySQL ODBC 8.0 Unicode Driver]
#Description     = ODBC for MySQL
#Driver          = /usr/lib64/libmyodbc8w.so
#FileUsage       = 1
[MySQL Driver]
Description     = ODBC for MySQL
Driver          = /usr/lib/libmyodbc8a.so
FileUsage       = 1

# Driver from the oracle-connector-odbc package
# Setup from the unixODBC package
[Oracle 19 ODBC driver]
Description=Oracle ODBC driver for Oracle 19
Driver=/usr/lib/libsqora.so.19.1
```

**配置完以后重启BE即可，所有BE节点操作一样**

## 4. Doris 使用ODBC访问外部数据源

### 4.1 不使用Resource方式访问

```sql
CREATE EXTERNAL TABLE `test_mysql` (
  `k1` decimal(9, 3) NOT NULL COMMENT "",
  `k2` char(10) NOT NULL COMMENT "",
  `k3` datetime NOT NULL COMMENT "",
  `k5` varchar(20) NOT NULL COMMENT "",
  `k6` double NOT NULL COMMENT ""
) ENGINE=ODBC
COMMENT "ODBC"
PROPERTIES (
"host" = "192.168.0.1",
"port" = "3306",
"user" = "root",
"password" = "123456",
"database" = "test",
"table" = "test",
"driver" = "MySQL Driver",  --注意这里的名称和odbcinst.ini里的mysql[]里的名称一致
"odbc_type" = "mysql"
);
```

然后在doris里执行查询验证

### 4.2 通过ODBC_Resource来创建ODBC外表 (推荐使用的方式)

首先创建Resource

```
CREATE EXTERNAL RESOURCE `mysql_odbc`
PROPERTIES (
"type" = "odbc_catalog",
"host" = "192.168.0.1",
"port" = "3306",
"user" = "root",
"password" = "123456",
"driver" = "MySQL Driver",  --注意这里的名称和odbcinst.ini里的mysql[]里的名称一致
"odbc_type" = "mysql"
);
```

然后创建 ODBC 外表

```sql
CREATE EXTERNAL TABLE `test_mysql` (
  `k1` decimal(9, 3) NOT NULL COMMENT "",
  `k2` char(10) NOT NULL COMMENT "",
  `k3` datetime NOT NULL COMMENT "",
  `k5` varchar(20) NOT NULL COMMENT "",
  `k6` double NOT NULL COMMENT ""
) ENGINE=ODBC
COMMENT "ODBC"
PROPERTIES (
"odbc_catalog_resource" = "mysql_odbc",
"database" = "test",
"table" = "baseall"
);
```

