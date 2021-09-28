layout: post
title: "Apache Doris 数据更新操作"
date: 2021-09-30
description: "Apache Doris 数据更新操作"
tag: Apache Doris

# Apache Doris 数据更新操作

# 1. 介绍

Doris 数据更新目前只在Unique Key 模型上，其他模型不支持数据更新操作，更新操作有两种方式：

1. REPLACE方式：这种方式和聚合模型中的Replace原理一致，只要表中存在相同key的值，会采用Replace方式替换Value，
2. UPDATE方式：支持单独修改某一列或者多列，通过Where条件方式

Unique 模型：Doris 系统中的一种数据模型。将列分为两类，Key 和 Value。当用户导入相同 Key 的行时，后者的 Value 会覆盖已有的 Value。与 Mysql 中的 Unique 含义一致

这里主要介绍Update方式使用及要注意的事项，对于正常的插入更新，只需要将数据推介通过Stream Load或者其他load方式，当做正常数据插入就可以了。

**Update方式要在0.15.0及以后版本中使用**

## 2. 适应场景

Update 方式适应场景如下：

- 对满足某些条件的行，修改它的取值。
- 点更新，小范围更新，待更新的行最好是整个表的非常小一部分。
- update 命令只能在 Unique 数据模型的表中操作。

# 2.Runtime Filter相关解释

- 左表：Join查询时，左边的表。进行Probe操作。可被Join Reorder调整顺序。
- 右表：Join查询时，右边的表。进行Build操作。可被Join Reorder调整顺序。
- Fragment：FE会将具体的SQL语句的执行转化为对应的Fragment并下发到BE进行执行。BE上执行对应Fragment，并将结果汇聚返回给FE。
- Join on clause: `A join B on A.a=B.b`中的`A.a=B.b`，在查询规划时基于此生成join conjuncts，包含join Build和Probe使用的expr，其中Build expr在Runtime Filter中称为src expr，Probe expr在Runtime Filter中称为target expr

# 3.基本原理及使用方式

## 3.1 基本原理

利用查询引擎自身的 where 过滤逻辑，从待更新表中筛选出需要被更新的行。再利用 Unique 模型自带的 Value 列新数据替换旧数据的逻辑，将待更新的行变更后，再重新插入到表中。从而实现行级别更新

假设 Doris 中存在一张订单表，其中 订单id 是 Key 列，订单状态，订单金额是 Value 列。数据状态如下：

| 订单id         | 订单金额 | 订单状态 |
| -------------- | -------- | -------- |
| 20210928100001 | 100      | 待付款   |

这时候，用户点击付款后，Doris 系统需要将订单id 为 '20210928100001' 的订单状态变更为 '待发货', 就需要用到 Update 功能。

```text
UPDATE order SET 订单状态='待发货' WHERE 订单id=20210928100001;
```

用户执行 UPDATE 命令后，系统会进行如下三步：

- 第一步：读取满足 WHERE 订单id=20210928100001 的行 （20210928100001，100，'待付款'）

- 第二步：变更该行的订单状态，从'待付款'改为'待发货' （20210928100001，100，'待发货'）

- 第三步：将更新后的行再插入回表中，从而达到更新的效果。

| 订单id         | 订单金额 | 订单状态 |
| -------------- | -------- | -------- |
| 20210928100001 | 100      | 待付款   |

由于表 order 是 UNIQUE 模型，所以相同 Key 的行，之后后者才会生效，所以最终效果如下：

| 订单id         | 订单金额 | 订单状态 |
| -------------- | -------- | -------- |
| 20210928100001 | 100      | 待发货   |

## 3.2 使用方法

### 3.2.1 语法

```sql
UPDATE table_name SET value=xxx WHERE condition;
```

- `table_name`: 待更新的表，必须是 UNIQUE 模型的表才能进行更新。
- value=xxx: 待更新的列，等式左边必须是表的 value 列。等式右边可以是常量，也可以是某个表中某列的表达式变换。 比如 value = 1, 则待更新的列值会变为1。 比如 value = value +1， 则待更新的列值会自增1。
- condition：只有满足 condition 的行才会被更新。condition 必须是一个结果为 Boolean 类型的表达式。 比如 k1 = 1, 则只有当 k1 列值为1的行才会被更新。 比如 k1 = k2, 则只有 k1 列值和 k2 列一样的行才会被更新。 不支持不填写condition，也就是不支持全表更新。
- Update 语法在 Doris 中是一个同步语法，既 Update 语句成功，更新就成功了，数据可见。

# 4. 数据更新性能

## 4.1 Update方式

Update 语句的性能和待更新的行数，以及 condition 的检索效率密切相关。

- 待更新的行数：待更新的行数越多，Update 语句的速度就会越慢。这和导入的原理是一致的。 Doris 的更新比较合适偶发更新的场景，比如修改个别行的值。 Doris 并不适合大批量的修改数据。大批量修改会使得 Update 语句运行时间很久。
- condition 的检索效率：Doris 的 Update 实现原理是先将满足 condition 的行读取处理，所以如果 condition 的检索效率高，则 Update 的速度也会快。 condition 列最好能命中，索引或者分区分桶裁剪。这样 Doris 就不需要扫全表，可以快速定位到需要更新的行。从而提升更新效率。 强烈不推荐 condition 列中包含 UNIQUE 模型的 value 列。

## 4.2 Replace 方式

这种方式和导入方式基本是一样的，对性能影响不大

# 5. 并发控制

默认情况下，并不允许同一时间对同一张表并发进行多个 Update 操作。

主要原因是，Doris 目前支持的是行更新，这意味着，即使用户声明的是 `SET v2 = 1`，实际上，其他所有的 Value 列也会被覆盖一遍（尽管值没有变化）。

这就会存在一个问题，如果同时有两个 Update 操作对同一行进行更新，那么其行为可能是不确定的。也就是可能存在脏数据。

但在实际应用中，如果用户自己可以保证即使并发更新，也不会同时对同一行进行操作的话，就可以手动打开并发限制。通过修改 FE 配置 `enable_concurrent_update`。当配置值为 true 时，则对更新并发无限制

# 6. 使用风险

由于 Doris 目前支持的是行更新，并且采用的是读取后再写入的两步操作，则如果 Update 语句和其他导入或 Delete 语句刚好修改的是同一行时，存在不确定的数据结果。

所以用户在使用的时候，一定要注意*用户侧自己*进行 Update 语句和其他 DML 语句的并发控制。

