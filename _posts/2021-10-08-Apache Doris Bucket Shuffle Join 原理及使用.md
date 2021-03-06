---
layout: post
title: "Apache Doris Bucket Shuffle Join 原理及使用"
date: 2021-10-08
description: "Apache Doris Bucket Shuffle Join 原理及使用"
tag: Apache Doris
---

# Apache Doris Bucket Shuffle Join 原理及使用

## 1. 概述

Bucket Shuffle Join 是在 Doris 0.14 版本中正式加入的新功能。旨在为某些 Join 查询提供本地性优化，来减少数据在节点间的传输耗时，来加速查询。

Bucket Shuffle Join 和 Colocate Join的区别在于：Colocate Join是本地计算，不会产生网络传输，而Bucket Shuffle Join是本地性优化，减少网络传输，还是回产生网络传输。

## 2.名词解释

- 左表：Join查询时，左边的表。进行Probe操作。可被Join Reorder调整顺序。
- 右表：Join查询时，右边的表。进行Build操作。可被Join Reorder调整顺序

## 3.原理

Doris支持的常规分布式Join方式包括了shuffle join 和 broadcast join。这两种join都会导致不小的网络开销:

举个例子，当前存在A表与B表的Join查询，它的Join方式为HashJoin，不同Join类型的开销如下：

- **Broadcast Join**: 如果根据数据分布，查询规划出A表有3个执行的HashJoinNode，那么需要将B表全量的发送到3个HashJoinNode，那么它的网络开销是`3B`，它的内存开销也是`3B`。
- **Shuffle Join**: Shuffle Join会将A，B两张表的数据根据哈希计算分散到集群的节点之中，所以它的网络开销为 `A + B`，内存开销为`B`。

在Doris FE 元数据中保存了 Doris 中每个表的数据分布信息，如果 Join 语句命中了表的数据分布列，我们应该使用数据分布信息来减少join语句的网络与内存开销，这就是Bucket Shuffle Join的思路来源。

![image.png](/images/bucket_shuffle_join.png)

上面的图片展示了Bucket Shuffle Join的工作原理。SQL语句为 A表 join B表，并且join的等值表达式命中了A的数据分布列。而Bucket Shuffle Join会根据A表的数据分布信息，将B表的数据发送到对应的A表的数据存储计算节点。Bucket Shuffle Join开销如下：

- 网络开销： `B < min(3B, A + B)`
- 内存开销： `B <= min(3B, B)`

可见，相比于Broadcast Join与Shuffle Join， Bucket Shuffle Join有着较为明显的性能优势。减少数据在节点间的传输耗时和Join时的内存开销。相对于Doris原有的Join方式，它有着下面的优点

- 首先，Bucket-Shuffle-Join降低了网络与内存开销，使一些Join查询具有了更好的性能。尤其是当FE能够执行左表的分区裁剪与桶裁剪时。
- 其次，同时与Colocate Join不同，它对于表的数据分布方式并没有侵入性，这对于用户来说是透明的。对于表的数据分布没有强制性的要求，不容易导致数据倾斜的问题。
- 最后，它可以为Join Reorder提供更多可能的优化空间。

## 4.使用方式

### 4.1 设置Session变量

将session变量`enable_bucket_shuffle_join`设置为`true`，则FE在进行查询规划时就会默认将能够转换为Bucket Shuffle Join的查询自动规划为Bucket Shuffle Join。

```text
set enable_bucket_shuffle_join = true;
```

在FE进行分布式查询规划时，优先选择的顺序为 Colocate Join -> Bucket Shuffle Join -> Broadcast Join -> Shuffle Join。但是如果用户显式hint了Join的类型，如：

```sql
select * from test join [shuffle] baseall on test.k1 = baseall.k1;
```

则上述的选择优先顺序则不生效。

该session变量在0.14版本默认为`true`, 而0.13版本需要手动设置为`true`。

>**注意**
>
>1.目前 Bucket Shuffle Join不能像Shuffle Join那样可以显示指定Join方式，只能让执行引擎自动选择，选择的顺序：Colocate Join -> Bucket Shuffle Join -> Broadcast Join -> Shuffle Join。
>
>2.如果显示的指定了Join的方式，上面的选择优先顺序则不生效，按照指定的方式执行

### 4.2 查看SQL 语句 Join类型

可以通过`explain`命令来查看Join是否为Bucket Shuffle Join：

```text
|   2:HASH JOIN                                                         
|   |  join op: INNER JOIN (BUCKET_SHUFFLE)                            
|   |  hash predicates:                                                 
|   |  colocate: false, reason: table not in the same group             
|   |  equal join conjunct: `test`.`k1` = `baseall`.`k1`                               
```

在Join类型之中会指明使用的Join方式为：`BUCKET_SHUFFLE`

### 4.3 Bucket Shuffle Join的规划规则

在绝大多数场景之中，用户只需要默认打开seesion变量的开关就可以透明的使用这种Join方式带来的性能提升，但是如果了解Bucket Shuffle Join的规划规则，可以帮助我们利用它写出更加高效的SQL。

- Bucket Shuffle Join只生效于Join条件为`等值`的场景，原因与Colocate Join类似，它们都依赖hash来计算确定的数据分布。
- 在等值Join条件之中包含两张表的分桶列，当左表的分桶列为等值的Join条件时，它有很大概率会被规划为Bucket Shuffle Join。
- 由于不同的数据类型的hash值计算结果不同，所以Bucket Shuffle Join要求左表的分桶列的类型与右表等值join列的类型需要保持一致，否则无法进行对应的规划。
- Bucket Shuffle Join只作用于`Doris原生的OLAP表`，对于ODBC，MySQL，ES等外表，当其作为左表时是无法规划生效的。
- 对于分区表，由于每一个分区的数据分布规则可能不同，所以Bucket Shuffle Join只能保证左表为单分区时生效。所以在SQL执行之中，需要尽量使用`where`条件使分区裁剪的策略能够生效。
- 假如左表为Colocate的表，那么它每个分区的数据分布规则是确定的，Bucket Shuffle Join能在Colocate表上表现更好

