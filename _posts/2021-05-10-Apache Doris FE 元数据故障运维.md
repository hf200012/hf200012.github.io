---
layout: post
title: "Apache Doris FE元数据故障运维"
date: 2021-03-12 
description: "Apache Doris FE元数据故障运维"
tag: Apache Doris
---

[Doris](https://link.zhihu.com/?target=https%3A//www.oschina.net/action/GoToLink%3Furl%3Dhttp%3A%2F%2Fdoris.apache.org)是一个非常优秀的MPP数据仓库，

在前两天线上出现一个问题，我三个FE，出现了一个FE挂掉，然后我重启启动不起来，

我备份这个FE元数据后，将这个节点元数据清除掉，使用

```text
--先删除
ALTER SYSTEM DROP FOLLOWER "FE:9010"
--在添加
ALTER SYSTEM ADD FOLLOWER "FE:9010"
```

删除改节点，然后在使用--helper方式把该节点作为一个新的FE加入到集群中，但是这时候启动会报错，同时会导致Master FE挂掉，具体异常信息如下

```text
repImpl=com.sleepycat.je.rep.impl.RepImpl@68fa4d19 props={REFRESH_VLSN=17921230, PORT=9010, HOSTNAME=172.22.197.238, P_NODETYPE1=ELECTABLE, NODE_NAME=172.22.197.238_9010_1611290318143, P_NODETYPE0=SECONDARY, P_NODENAME1=172.22.197.240_9010_1608972313975, P_PORT1=9010, P_NODENAME0=172.22.197.238_9010_1611290318143, P_PORT0=9010, P_HOSTNAME1=172.22.197.240, GROUP_NAME=PALO_JOURNAL_GROUP, P_HOSTNAME0=172.22.197.238, ENV_DIR=/hdd_data01/doris-meta/bdb, P_NUMPROVIDERS=2}
 at com.sleepycat.je.rep.InsufficientLogException.wrapSelf(InsufficientLogException.java:315) ~[je-7.3.7.jar:7.3.7]
 at com.sleepycat.je.dbi.EnvironmentImpl.checkIfInvalid(EnvironmentImpl.java:1766) ~[je-7.3.7.jar:7.3.7]
```



![img](https://pic4.zhimg.com/80/v2-1c5a9e0826fd87c9b5b0308a493dc477_1440w.jpg)



后来在社区，缪小姐姐及陈明雨大神的协助下，进行了各种尝试定位，认为是在启动的时候元数据同步异常，这个异常可能是因为我当时的load数据任务在同步修改元数据，造成的，后来在凌晨，生产开货完成以后，停掉所有load任务，然后执行删除问题FE节点元数据，然后在重新使用--helper启动，依然报错，最后没办法，尝试将master节点的fe元数据拷贝到问题节点FE，将问题节点FE的元数据目录删除，然后重建，将赋值过来的元数据，拷贝到元数据目录

具体步骤：

1. 停止所有load任务
2. 删除元数据目录，并重建目录
3. 从master节点拷贝元数据到问题节点fe（将 fe.conf 中的 metadata_failure_recovery=true 配置项删除，或者设置为 false，**这个非常重要**），注意要修改image/ROLE 里的name,拷贝过来的是master的名称，改成该节点的名称
4. 执行 ALTER SYSTEM DROP FOLLOWER 删除改节点
5. 在问题节点使用--helper启动服务
6. 在mysql下执行 ALTER SYSTEM ADD FOLLOWER 将FE节点从新加入进去
7. 启动正常

**注意：**

```text
1.问题节点：将 fe.conf 中的 metadata_failure_recovery=true 配置项删除，或者设置为 false
2.Master节点启动使用 metadata_failure_recovery=true启动，进行恢复，启动正常以后，将这个配置删除或者设置为false，停掉Master fe，然后在重启，启动完成以后，要确认Master查询，导入是正常的
```

上述步骤执行完成以后，然后在问题节点在使用--helper方式启动fe，这个时候正常启动，问题解决

重新启动所有Load任务

不过这个还是一个问题，理论上，将问题FE节点元数据删除以后，把它当做一个新的FE节点，使用--helper加入进去，是应该没问题的，启动以后会自动从master FE同步元数据，但是出现了同步失败的情况（这个时候也没有任何load任务，对元数据进行修改的情况），这个问题我会提交给社区进行排查