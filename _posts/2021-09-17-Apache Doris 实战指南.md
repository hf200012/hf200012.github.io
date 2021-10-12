---
layout: post
title: "Apache Doris 实战指南"
date: 2021-09-17
description: "Apache Doris 实战指南"
tag: Apache Doris
---
# 第一部分 Apache Doris 使用入门

## 1.1 Apache Doris 介绍

[Apache Doris 架构及组件介绍 ](https://hf200012.github.io/2021/09/Apache-doris架构及组件介绍/)

## 1.2 Apache Doris 安装

### 1.2.1 Doris 编译安装部署手册

[Apache Doris 环境编译安装部署 ](https://hf200012.github.io/2021/09/Apache-Doris-环境安装部署/)

[Apache Doris 升级手册 ](https://hf200012.github.io/2021/09/Apache-Doris-升级手册/)

[使用supervisor实现Apache Doris进程自动拉起](https://hf200012.github.io/2020/12/使用supervisor实现Apache-Doris进程自动拉起/)

### 1.2.2 Doris FE 高可用及负载均衡

[Apache Doris FE使用ProxySQL实现负载均衡 ](https://hf200012.github.io/2021/09/Apache-doris-FE使用ProxySQL实现负载均衡/)

## 1.3. Apache Doris 模型介绍

### 1.3.1 Doris 数据模型

[Apache doris 数据模型](https://hf200012.github.io/2021/09/Apache-Doris数据模型/)

### 1.3.2 Doris 数据划分

[Apache Doris 关系模型与数据划分](https://hf200012.github.io/2021/08/Apache-Doris关系模型与数据划分/)

### 1.3.3 Doris 物化视图及Rollup

[Apache Doris 物化视图介绍](https://hf200012.github.io/2021/09/Apache-Doris-物化视图介绍/)

### 1.3.4 RuntimeFilter 原理及使用

[Apache Doris RuntimeFilter 原理及使用 ](https://hf200012.github.io/2021/09/Apache-Doris-RuntimeFilter-原理及使用/)

### 1.3.5 Doris 数据动态分区使用

[Apache Doris 动态分区介绍及使用方法 ](https://hf200012.github.io/2021/09/Apache-Doris-动态分区介绍及使用方法/)

### 1.3.6 Doris 索引

[Apache doris 排序键及ShortKey Index](https://hf200012.github.io/2021/09/Apache-Doris-排序键及ShortKey-Index/)

## 1.4. Apache Doris使用实战

### 1.4.1 Doris数据导入

[Apache Doris 数据导入总览](https://hf200012.github.io/2021/09/Apache-Doris-数据导入/)

#### 1.4.1.1 Broker load 使用

[Apache Doris Broker 数据导入](https://hf200012.github.io/2021/09/Apache-Doris-Broker数据导入/)

#### 3.4.1.2 Routine load使用

[Apache Doris Routine Load数据导入使用方法](https://hf200012.github.io/2021/09/Apache-Doris-Routine-Load数据导入使用方法/)

#### 3.4.1.3 Spark load 使用

#### 3.4.1.4 Stream load 使用

[Apache Doris Stream Load数据导入 ](https://hf200012.github.io/2021/09/Apache-Doris-Stream-Load数据导入/)

#### 3.4.1.5 Insert into

[Apache Doris 数据导入之INSERT](https://hf200012.github.io/2021/10/Apache-Doris-数据导入之INSERT/)

#### 3.4.1.6 数据更新顺序保证

[Apache Doris Sequence介绍及使用方法 ](https://hf200012.github.io/2021/09/Apache-Doris-sequence介绍及使用方法/)

### 1.4.2 数据导出

#### 1.4.2.1 Export 导出

#### 1.4.2.2 查询结果导出

### 1.4.3 Doris数据删除

### 1.4.4 Doris数据更新

[Apache doris 数据更新操作](https://hf200012.github.io/2021/09/Apache-Doris-数据更新操作/)

## 1.5 Apache Doris Join 原理及使用

### 1.5.1 Colocation Join

[Apache Doris Colocate Join 原理及使用 ](https://hf200012.github.io/2021/10/Apache-Doris-Colocate-Join-原理及使用/)

### 1.5.2 Bucket Shuffle Join

[Apache Doris Bucket Shuffle Join 原理及使用](https://hf200012.github.io/2021/10/Apache-Doris-Bucket-Shuffle-Join-原理及使用/)

## 1.6 Apache Doris 扩展组件介绍及使用

### 1.6.1 Spark Doris Connector 

[Spark Doris Connector设计方案 ](https://hf200012.github.io/2021/10/Apache-Doris-Spark-Connector设计方案/)

### 1.6.2 Flink Doris Connector

[Flink Doris Connector设计方案](https://hf200012.github.io/2021/10/Apache-Doris-Flink-Connector设计方案/)

[Flink 使用 sql 读取 kafka 利用doris flink connector写入到doris表中 ](https://hf200012.github.io/2021/09/Flink-使用-SQL-读取-Kafka-利用Doris-Flink-Connector写入到Doris表中/)

[Flink Mysql CDC结合Doris flink connector实现数据实时入库](https://hf200012.github.io/2021/09/实现通过Flink-Mysql-CDC结合Apache-doris-flink-connector实现数据实时入库/)

### 1.6.3 Datax DorisWriter 

[Apache Doris Datax DorisWriter扩展使用方法 ](https://hf200012.github.io/2021/09/Apache-doris-Datax-DorisWriter扩展使用方法/)

### 1.6.4 Doris 日志审计插件

[Apache doris sql日志审计 ](https://hf200012.github.io/2021/09/Apache-Doris-SQL日志审计/)

### 1.6.5 Doris to logstash 插件

### 1.6.6 Doris ODBC 外表

[Apache doris ODBC外表使用方式 ](https://hf200012.github.io/2021/09/Apache-doris-ODBC外表使用方式/)

[Apache Doris ODBC mysql外表注意事项 ](https://hf200012.github.io/2021/09/Apache-doris-ODBC-mysql外表注意事项/)

### 1.6.7 Doris On ElasticSearch

# 第二部分 基于Doris的数据中台实践

## 2.1  数据中台构建内容

### 2.1.1 什么是数据中台

[基于Apache doris怎么构建数据中台(一)-什么是数据中台](https://hf200012.github.io/2021/08/基于Apache-doris怎么构建数据中台(一)-什么是数据中台/)

### 2.2.2 数据中台构建内容

[基于Apache doris怎么构建数据中台(二)-数据中台建设内容](https://hf200012.github.io/2021/08/基于Apache-doris怎么构建数据中台(二)-数据中台建设内容/)

## 2.3 基于Doris的元数据管理系统构建

[元数据管理系统 ](https://hf200012.github.io/2021/08/元数据管理系统/)

[基于Apache doris怎么构建数据中台(三)-数据资产管理 ](https://hf200012.github.io/2021/08/基于Apache-doris怎么构建数据中台(三)-数据资产管理/)

## 2.4 怎么基于Doris现数据快速接入

[基于Apache doris怎么构建数据中台(四)-数据接入系统](https://hf200012.github.io/2021/08/基于Apache-doris怎么构建数据中台(四)-数据接入系统/)

## 2.5. 基于规则的数据质量引擎构建

[基于Apache doris怎么构建数据中台(五)-数据质量管理](https://hf200012.github.io/2021/09/基于Apache-doris怎么构建数据中台(五)-数据质量/)

## 2.6 基于Doris实现数据服务快速开发

[基于Apache doris怎么构建数据中台(六)-数据服务管理](https://hf200012.github.io/2021/09/基于Apache-doris怎么构建数据中台(六)-数据服务/)

## 2.7 基于Doris的数据指标体系构建

[如何构建公司的数据指标体系 ](https://hf200012.github.io/2021/07/如何构建公司的数据指标体系/)

[基于Apache Doris怎么构建数据中台(七)-数据指标管理 ](https://hf200012.github.io/2021/09/基于Apache-Doris怎么构建数据中台(七)-数据指标管理/)

## 2.8 Doris的数仓管理设计



## 2.9 数据安全设计

# 第三部分 Apache doris运维

## 3.1 Apache Doris 运维操作

### 3.1.1 磁盘空间管理

### 3.1.2 元数据运维

### 3.1.3 监控及预警

### 3.1.4 副本管理

### 3.1.5 Tablet元数据运维及恢复

### 3.1.6 Doris 日常使用问题答疑

[Apache doris 使用过程中常见问题汇总](https://hf200012.github.io/2021/09/Apache-doris-使用过程中常见问题汇总/)

[Apache Doris常见问题答疑(一)](https://hf200012.github.io/2021/09/Apache-Doris常见问题答疑(一)/)

[Apache Doris常见问题答疑(二)](https://hf200012.github.io/2021/09/Apache-Doris常见问题答疑(二)/)

