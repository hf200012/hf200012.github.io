---
layout: post
title: "Apache Doris 元数据运维"
date: 2021-11-03
description: "Apache Doris 元数据运维"
tag: Apache Doris
---
# Apache Doris 元数据运维

## 1. 元数据恢复

Apache Doris在实际使用中可能是因为某些原因 FE（Frontend）出现无法启动 bdbje、FE 之间元数据无法同步等问题。故障表现包括：无法进行元数据写操作、没有 MASTER 等等，这时就需要手动恢复 FE。

>**重要提示**
>
>当前元数据的设计是无法向后兼容的。即如果新版本有新增的元数据结构变动（可以查看 FE 代码中的 `FeMetaVersion.java` 文件中是否有新增的 VERSION），那么在升级到新版本后，通常是无法再回滚到旧版本的。所以，在升级 FE 之前，请务必按照 集群部署升级章节中的操作，测试元数据兼容性。

### 1.1 恢复原理

手动恢复 FE 的大致原理，是先通过当前 FE meta_dir 中的元数据，找出元数据版本最新的元数据，使用元数据恢复模式启动，然后其他FE，重新作为一个新的FE节点添加到集群中。

>如果你是单节点FE集群，直接通过元数据恢复模式启动，在FE的fe.conf中添加metadata_failure_recovery=true参数，重新启动FE即可

### 1.2 恢复示例

这里我们以一个高可用的Doris集群为例（三个FE Follower）为例进行讲解。

整个操作步骤请严格按照下面的步骤进行：

#### 1.2.1 停止所有 FE 进程，

首先要停止所有FE进程，同时停止一切业务访问。保证在元数据恢复期间，不会因为外部访问导致其他不可预期的问题造成元数据不一致。

#### 1.2.2 找出元数据版本最新的FE节点

- 进入到每个FE节点的元数据目录（fe.conf中配置的meta_dir目录）
- 备份整个元数据目录，防止在操作过程中的误操作导致元数据损坏
- 通常情况下在集群出问题之前的FE Master节点，元数据版本是最新的。
- 确保你操作的节点元数据版本是最新的可以通过查看你元数据目录下的image目录，image.xxxx 文件的后缀，数字越大，则表示元数据越新，如下所示：

```
# tree
.
├── bdb
│   ├── 000039e7.jdb
│   ├── 00003a03.jdb
│   ├── 00003a14.jdb
│   ├── 00003a2e.jdb
│   ├── 00003a5b.jdb
│   ├── 00003a63.jdb
│   ├── 00003aff.jdb
...
│   ├── 000041cb.jdb
│   ├── 000041cc.jdb
│   ├── je.config.csv
│   ├── je.info.0
│   ├── je.info.0.lck
│   ├── je.lck
│   ├── je.stat.0.csv
...
│   ├── je.stat.8.csv
│   └── je.stat.csv
└── image
    ├── image.186235646
    ├── ROLE
    └── VERSION
```

现在，我们要使用这个拥有最新元数据的 FE 节点，进行恢复，这里要确保是 FOLLOWER 节点恢复，而不是Observer节点，如果使用 OBSERVER 节点的元数据进行恢复会比较麻烦，cat ROLE(image文件夹下的ROLE文件)可以看到该节点角色

#### 1.2.3 在元数据版本最新的节点开始执行恢复操作

1. 在 fe.conf 中添加配置：`metadata_failure_recovery=true`
   `metadata_failure_recovery=true` 的含义是，清空 "bdbje" 的元数据。这样 bdbje 就不会再联系之前的其他 FE ，而作为一个独立的 FE 启动。这个参数只有在恢复启动时才需要设置为 true。恢复完成后，一定要设置为 false，否则一旦重启，bdbje 的元数据又会被清空，导致其他 FE 无法正常工作
2.  执行 `sh bin/start_fe.sh --daemon`,启动该FE，然后验证是否正常启动，可以通过log/fe.out日志看到，
3.  启动正常之后，先连接到这个 FE，执行一些查询导入，检查是否能够正常访问。如果不正常，有可能是操作有误，查看FE启动日志，排查问题后重新启动FE。 
4. 如果成功，通过 `show frontends;` 命令，应该可以看到之前集群所添加的所有 FE，并且当前 FE 是 master 
5. 然后将fe.conf 中的配置：`metadata_failure_recovery=true`，改成 false，重启改节点FE，验证是否启动成功，如果启动成功，执行下面的步骤。 
6. 通过Mysql 客户端连接当前FE，执行`ALTER SYSTEM DROP FOLLOWER "IP:edit_log_port"` 将其他的FE从集群中删除
7.  进入其他的FE节点，将改节点的元数据先备份，然后清空元数据目录下的数据 
8. 清空元数据目录之后，执行 `sh bin/start_fe.sh --helper "master_FE_IP:edit_log_port" --daemon`作为一个新节点加入到集群中 
9. 在通过Mysql 客户端连接刚才的Master FE，执行`ALTER SYSTEM DROP FOLLOWER "IP:edit_log_port"` 将改FE节点加入到集群中， 
10. 观察新加入的FE节点日志log/fe.out，fe.log，如果正常启动，将自己转换成Follower，并且开始从Master同步元数据，说明启动征程
    3.11 另外其他的FE节点的接入方法一样

#### 1.2.4 从Observer恢复

如果你的集群只有一个FE Follower节点，元数据已经损坏没办法恢复，但是你有FE Observer，这时可以尝试通过Observer来进行恢复集群。

1. 先将 meta_dir/image/ROLE 文件中的 `role=OBSERVER` 改为 `role=FOLLOWER 
2. 然后执行上面章节的步骤前三步 (1.2.3章节1-3步骤) 
3. 如果是用一个 OBSERVER 节点的元数据进行恢复的，那么完成如上步骤后，`show frontends;` 会发现，当前这个 FE 的角色为 OBSERVER，但是 IsMaster 显示为 true。这是因为，这里看到的 “OBSERVER” 是记录在 Doris 的元数据中的，而是否是 master，是记录在 bdbje 的元数据中的。因为我们是从一个 OBSERVER 节点恢复的，所以这里出现了不一致。这种内部不一致的状态，不能支持后续的导入等修改操作，所以，还需要继续按以下步骤修复： 
4.  先把除了这个 `OBSERVER`以外的所有 FE 节点 DROP 掉。 
5.  通过 `ADD FOLLOWER` 命令，添加一个新的 FOLLOWER FE，假设在 Doris_FE_A 上 
6. 在 Doris_FE_A 上启动一个全新的 FE，通过 --helper 的方式加入集群，这里的--helper参数里的master IP地址就是你刚才恢复的Observer的这台机器的IP地址 
7. 启动成功后，通过 `show frontends;` 语句，你应该能看到两个 FE，一个是之前的 OBSERVER，一个是新添加的 FOLLOWER，并且 OBSERVER 是 master 
8. 确认这个新的 FOLLOWER 是可以正常工作之后，用这个新的 FOLLOWER 的元数据，然后停掉原先Observer这台机器上的FE，并通过`ALTER SYSTEM DROP OBSERVER`方式删除掉 
9. 在重新添加启动一个新的Observer，即可









