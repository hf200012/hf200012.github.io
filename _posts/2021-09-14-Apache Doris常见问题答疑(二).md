---
layout: post
title: "Apache Doris常见问题答疑(二)"
date: 2021-09-14
description: "Apache Doris常见问题答疑(二)"
tag: Apache Doris
---
**Q：show backends/frontends 查看到的信息不完整**

**A：**在执行如 `show backends/frontends` 等某些语句后，结果中可能会发现有部分列内容不全。比如show backends结果中看不到磁盘容量信息等。

通常这个问题会出现在集群有多个FE的情况下，如果用户连接到非Master FE节点执行这些语句，就会看到不完整的信息。这是因为，部分信息仅存在于Master FE节点。比如BE的磁盘使用量信息等。所以只有在直连Master FE后，才能获得完整信息。

当然，用户也可以在执行这些语句前，先执行 `set forward_to_master=true;` 这个会话变量设置为true后，后续执行的一些信息查看类语句会自动转发到Master FE获取结果。这样，不论用户连接的是哪个FE，都可以获取到完整结果了。

**Q：通过Java程序调用stream load导入数据，在一批次数据量较大时，可能会报错Broken Pipe**

A：

除了Broken Pipe外，还可能出现一些其他的奇怪的错误。

这个情况通常出现在开启httpv2后。因为httpv2是使用spring boot实现的http 服务，并且使用tomcat作为默认内置容器。但是jetty对307转发的处理似乎有些问题，所以后面将内置容器修改为了jetty。此外，在java程序中的 apache http client的版本需要使用4.5.13以后的版本。之前的版本，对转发的处理也存在一些问题。

所以这个问题可以有两种解决方式：

1.关闭 httpv2

  在 fe.conf 中添加 `enable_http_server_v2=false` 后重启FE。但是这样无法再使用新版UI界面，并且之后的一些基于httpv2的新接口也无法使用。（正常的导入查询不受影响）。

2. 升级

  可以升级到百度的Palo发行版0.14.13及之后的版本，已修复这个问题。

**Q：节点新增加了新的磁盘，为什么数据没有均衡到新的磁盘上？**

**A：**当前Doris的均衡策略是以节点为单位的。也就是说，是按照节点整体的负载指标（分片数量和总磁盘利用率）来判断集群负载。并且将数据分片从高负载节点迁移到低负载节点。如果每个节点都增加了一块磁盘，则从节点整体角度看，负载并没有改变，所以无法触发均衡逻辑。

此外，Doris目前并不支持单个节点内部，各个磁盘间的均衡操作。所以新增磁盘后，不会将数据均衡到新的磁盘。

但是，数据在节点之间迁移时，Doris会考虑磁盘的因素。比如一个分片从A节点迁移到B节点，会优先选择B节点中，磁盘空间利用率较低的磁盘。

这里我们提供3种方式解决这个问题：

1. 重建新表

  通过`create table like` 语句建立新表，然后使用 `insert into select` 的方式将数据从老表同步到新表。因为创建新表时，新表的数据分片会分布在新的磁盘中，从而数据也会写入新的磁盘。这种方式适用于数据量较小的情况（几十GB以内）。

2. 通过Decommission命令

  decommission 命令用于安全下线一个BE节点。该命令会先将该节点上的数据分片迁移到其他节点，然后在删除该节点。前面说过，在数据迁移时，会优先考虑磁盘利用率低的磁盘，因此该方式可以“强制”让数据迁移到其他节点的磁盘上。当数据迁移完成后，我们在 cancel 掉这个 decommission 操作，这样，数据又会重新均衡回这个节点。当我们对所有 BE 节点都执行一遍上述步骤后，数据将会均匀的分布在所有节点的所有磁盘上。

  注意，在执行 decommission 命令前，先执行以下命令，以避免节点下线完成后被删除。

```
 admin set frontend config("drop_backend_after_decommission" = "false");
```

3. 使用API手动迁移数据

 Doris提供了 HTTP API，可以手动指定一个磁盘上的数据分片迁移到另一个磁盘上，具体可参阅：

http://doris.incubator.apache.org/master/zh-CN/administrator-guide/http-actions/tablet-migration-action.html