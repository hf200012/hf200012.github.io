---
layout: post
title: "Apache Doris 环境安装部署"
date: 2021-09-16
description: "Apache Doris 环境安装部署"
tag: Apache Doris
---
# Apache Doris 环境安装部署

这里以百度的Doris发行版 Palo-0.14.13版本为例进行演示编译安装部署

## 1. Doris编译

### 1.1 docker 镜像下载

这里我们使用的最新镜像

Apache doris 0.14.0及百度发布的Palo-0.14.7及之前的版本都是要在Docker 1.2版本下编译，之后的在Docker 1.3.1下编译

1.3.1 版本 Docker 镜像下载

```shell
$ docker pull apache/incubator-doris:build-env-1.3.1
```

1.2 版本Docker镜像下载

```shell
$ docker pull apache/incubator-doris:build-env-1.2
```

### 1.2 Doris源码下载编译

这里我们使用的是百度最新发行版的代码0.14.13，（Apache doris和百度Palo发行版源码是一致的，不过因为Apache发版周期比较长，百度doris团队会发布三位版本的doris，主要是bugfix及一些新功能迭代）

Palo源码下载地址：https://github.com/baidu/palo/releases

Palo-0.14.13: https://github.com/baidu/palo/archive/refs/tags/PALO-0.14.13-release.tar.gz

我们将Doris的源码下载以后解压到指定目录，例如我这边是放到了/root/doris目录

我这里是解压以后将目录名称重命名成doris-0.14.13了

![image-20210916141702543](/images/install/image-20210916141702543.png)

#### 1.2.1 运行Docker镜像

关于Docker的安装运行在这里我就不在讲解，不知道的可以去百度一下。

建议以挂载本地 Doris 源码目录的方式运行镜像，这样编译的产出二进制文件会存储在宿主机中，不会因为镜像退出而消失。

同时，建议同时将镜像中 maven 的 `.m2` 目录挂载到宿主机目录，以防止每次启动镜像编译时，重复下载 maven 的依赖库。

我的运行命令如下：

```shell
docker run -it --name doris-build-1.3.1 -v /root/.m2:/root/.m2 -v /root/doris/:/root/doris/ apache/incubator-doris:build-env-1.3.1
```

运行以后就会直接进入到Docker容器

#### 1.2.2 编译Doris FE，BE

进入到你的doris源码目录：

```shell
# cd /root/doris/doris-0.14.13
# sh build.sh
```

等待编译完成，看到下面界面就说明编译完成

![image-20210916142532937](/images/install/image-20210916142532937.png)

编译好的安装包在源码根目录：output目录下，拷贝出来就是可以安装了

![image-20210916142645653](/images/install/image-20210916142645653.png)

#### 1.2.3 编译Doris Broker

```shell
# cd fs_brokers/apache_hdfs_broker
# sh build.sj
```

等待编译完成，可以在output目录下看到编译好的apache_hdfs_broker拷贝出来即可

![image-20210916143342647](/images/install/image-20210916143342647.png)

#### 1.2.4 doris扩展的编译

doris扩展包的编译，可以参照官网扩展功能里的编译及使用说明，这里不做介绍

## 2.Doris 安装

### 2.1 操作系统及环境要求

这里我们使用的操作系统是`CentOS 7.8`,不支持低于7的版本。

在部署之前，要准备的工作：

1. 关闭操作系统的交换分区
2. 关闭防火墙
3. 操作系统的文件系统Ext4
4. 设置操作系统最大打开文件数
5. 禁用Selinux
6. 在要安装FE，Broker的节点上提前安装JDK环境，版本最低值1.8 及以上，BE节点如果不安装Broker可以不安装JDK环境
7. 所有机器做时钟同步

### 2.2 安装Doris注意项

1. FE 的磁盘空间主要用于存储元数据，包括日志和 image。通常从几百 MB 到几个 GB 不等。
2. BE 的磁盘空间主要用于存放用户数据，总磁盘空间按用户总数据量 * 3（3副本）计算，然后再预留额外 40% 的空间用作后台 compaction 以及一些中间数据的存放。
3. 一台机器上可以部署多个 BE 实例，但是**只能部署一个 FE**。如果需要 3 副本数据，那么至少需要 3 台机器各部署一个 BE 实例（而不是1台机器部署3个BE实例）。**多个FE所在服务器的时钟必须保持一致（允许最多5秒的时钟偏差）**
4. 测试环境也可以仅适用一个 BE 进行测试。实际生产环境，BE 实例数量直接决定了整体查询延迟。

### 2.3 关于FE节点数量

1. FE 角色分为 Follower 和 Observer，（Leader 为 Follower 组中选举出来的一种角色，以下统称 Follower）。
2. FE 节点数据至少为1（1 个 Follower）。当部署 1 个 Follower 和 1 个 Observer 时，可以实现读高可用。当部署 3 个 Follower 时，可以实现读写高可用（HA）。
3. Follower 的数量**必须**为奇数，Observer 数量随意。
4. 根据以往经验，当集群可用性要求很高时（比如提供在线业务），可以部署 3 个 Follower 和 1-3 个 Observer。如果是离线业务，建议部署 1 个 Follower 和 1-3 个 Observer。

### 2.4 开始安装Doris

- 通常我们建议 10 ~ 100 台左右的机器，来充分发挥 Doris 的性能（其中 3 台部署 FE（HA），剩余的部署 BE
- 当然，Doris的性能与节点数量及配置正相关。在最少4台机器（一台 FE，三台 BE，其中一台 BE 混部一个 Observer FE 提供元数据备份），以及较低配置的情况下，依然可以平稳的运行 Doris。
- 如果 FE 和 BE 混部，需注意资源竞争问题，并保证元数据目录和数据目录分属不同磁盘。

这里我们使用3个FE，5个BE节点，来搭建一个完整的支持高可用的Doris集群,部署角色如下

| 节点IP       | 节点名称    | 部署角色        |
| ------------ | ----------- | --------------- |
| 192.168.1.10 | doris-fe-01 | Follower,Broker |
| 192.168.1.11 | doris-fe-02 | Follower,Broker |
| 192.168.1.12 | doris-fe-03 | Follower,Broker |
| 192.168.1.13 | doris-be-01 | BE              |
| 192.168.1.14 | doris-be-02 | BE              |
| 192.168.1.15 | doris-be-03 | BE              |
| 192.168.1.16 | doris-be-04 | BE              |
| 192.168.1.17 | doris-be-05 | BE              |

#### 2.4.1 Broker 部署

Broker 是用于访问外部数据源（如 hdfs）的进程。通常，我们只在FE机器上部署 broker 实例。

Broker 以插件的形式，独立于 Doris 部署。如果需要从第三方存储系统导入数据，需要部署相应的 Broker，默认提供了读取 HDFS 和百度云 BOS 的 fs_broker。fs_broker 是无状态的，建议每一个 FE 和 BE 节点都部署一个 Broker。

- 拷贝源码 fs_broker 的 output 目录下的相应 Broker 目录到需要部署的所有节点上。建议和 BE 或者 FE 目录保持同级。

- 修改相应 Broker 配置

  在相应 broker/conf/ 目录下对应的配置文件中，可以修改相应配置。

- 启动 Broker

  `sh bin/start_broker.sh --daemon` 启动 Broker。

- 添加 Broker

  要让 Doris 的 FE 和 BE 知道 Broker 在哪些节点上，通过 sql 命令添加 Broker 节点列表。

  使用 mysql-client 连接启动的 FE，执行以下命令：

  `ALTER SYSTEM ADD BROKER broker_name "host1:port1","host2:port2",...;`

  其中 host 为 Broker 所在节点 ip；port 为 Broker 配置文件中的 broker_ipc_port。

- 查看 Broker 状态

  使用 mysql-client 连接任一已启动的 FE，执行以下命令查看 Broker 状态：`SHOW PROC "/brokers";`

  ```sql
  #mysql -u root  -h  192.168.1.10 -P  9030
  
  mysql> ALTER SYSTEM ADD BROKER broker_01 "192.168.1.10:8000";
  mysql> ALTER SYSTEM ADD BROKER broker_01 "192.168.1.11:8000";
  mysql> ALTER SYSTEM ADD BROKER broker_01 "192.168.1.12:8000";
  
  mysql> show proc '/frontends';
  +---------------+---------------+----------+------+-------+---------------------+---------------------+--------+
  | Name          | IP            | HostName | Port | Alive | LastStartTime       | LastUpdateTime      | ErrMsg |
  +---------------+---------------+----------+------+-------+---------------------+---------------------+--------+
  | broker_01 | 192.168.1.10 | doris-fe-01  | 8000 | true  | 2021-09-16 10:21:31 | 2021-09-16 15:58:55 |        |
  | broker_02 | 192.168.1.11 | doris-fe-02  | 8000 | true  | 2021-09-16 10:21:31 | 2021-09-16 15:58:55 |        |
  | broker_03 | 192.168.1.12 | doris-fe-03  | 8000 | true  | 2021-09-16 10:21:31 | 2021-09-16 15:58:55 |        |
  +---------------+---------------+----------+------+-------+---------------------+---------------------+--------+
  3 rows in set (0.01 sec)
  
  ```

  

#### 2.4.2 部署Doris FE

- 拷贝编译好的 FE 部署文件到指定节点

  将源码编译生成的 output 下的 fe 文件夹拷贝到 FE 的节点指定部署路径下。

- 配置 FE

  1. 配置文件为 conf/fe.conf。其中注意：`meta_dir`：元数据存放位置。默认在 fe/doris-meta/ 下。如果，目录不存在需**手动创建**该目录。

     **注意：生产环境强烈建议单独指定目录不要放在Doris安装目录下，最好是单独的磁盘（如果有SSD最好），测试开发环境可以使用默认配置**

  2. fe.conf 中 JAVA_OPTS 默认 java 最大堆内存为 4GB，**建议生产环境调整至 8G 以上**。

  3. priority_networks配置

     因为有多网卡的存在，或因为安装过 docker 等环境导致的虚拟网卡的存在，同一个主机可能存在多个不同的 ip。当前 Doris 并不能自动识别可用 IP。所以当遇到部署主机上有多个 IP 时，必须通过 priority_networks 配置项来强制指定正确的 IP。

     priority_networks 是 FE 和 BE 都有的一个配置，配置项需写在 fe.conf 和 be.conf 中。该配置项用于在 FE 或 BE 启动时，告诉进程应该绑定哪个IP。示例如下：

     ```
     priority_networks=192.168.1.0/24
     ```

     这是一种 [CIDR (opens new window)](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)的表示方法。FE 或 BE 会根据这个配置项来寻找匹配的IP，作为自己的 localIP

- 启动 FE

  ```shell
  sh bin/start_fe.sh --daemon
  ```
  
  FE进程启动进入后台执行。日志默认存放在 fe/log/ 目录下。如启动失败，可以通过查看 fe/log/fe.log 或者 fe/log/fe.out 查看错误信息
  
  **默认第一个启动的 FE 就是 Master，也就是Follower（Leader）**
  
  这里我们先不安装其他 FE ，先完成 BE 的安装
  
- 检查安装的 FE 是否正常

  或者通过页面访问页面: htttp://192.168.1.10:8030 , 如果能正常访问就说明正常，或者通过下面的方式

  使用Mysql client连接

  ```sql
  #mysql -u root  -h  192.168.1.10 -P  9030
  
  mysql> show proc '/frontends';
  +----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
  | Name                             | IP            | HostName | EditLogPort | HttpPort | QueryPort | RpcPort | Role     | IsMaster | ClusterId  | Join | Alive | ReplayedJournalId | LastHeartbeat       | IsHelper | ErrMsg | Version         |
  +----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
  | 192.168.1.10_9010_1605850067231 | 10.220.146.10 | doris-be--fe-01  | 9010        | 8030     | 9030      | 9020    | FOLLOWER | true     | 2113522669 | true | true  | 29778512          | 2021-09-16 14:58:44 | true     |        | 0.14.13-Unknown |
  +----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
  1 row in set (0.04 sec)
  ```
  
#### 2.4.3 安装 Doris BE

  - 拷贝 BE 部署文件到所有要部署 BE 的节点
  
    将源码编译生成的 output 下的 be 文件夹拷贝到 BE 的节点的指定部署路径下。
  
  - 修改 BE 网络配置
  
    priority_networks 配置,这块参考上面 FE 的说明
  
  - 修改所有 BE 的配置

    修改 be/conf/be.conf。主要是配置 `storage_root_path`：数据存放目录。默认在be/storage下，需要**手动创建**该目录。多个路径之间使用英文状态的分号 `;` 分隔（**最后一个目录后不要加 `;`**）。可以通过路径区别存储目录的介质，HDD或SSD。可以添加容量限制在每个路径的末尾，通过英文状态逗号`,`隔开。

    示例1如下：

    **注意：如果是SSD磁盘要在目录后面加上`.SSD`,HDD磁盘在目录后面加`.HDD`**

    `storage_root_path=/home/disk1/doris.HDD,50;/home/disk2/doris.SSD,10;/home/disk2/doris`

    **说明**

    - /home/disk1/doris.HDD, 50，表示存储限制为50GB, HDD;
  - /home/disk2/doris.SSD 10， 存储限制为10GB，SSD；
    - /home/disk2/doris，存储限制为磁盘最大容量，默认为HDD

    示例2如下：

    **注意：不论HHD磁盘目录还是SSD磁盘目录，都无需添加后缀，storage_root_path参数里指定medium即可**

    `storage_root_path=/home/disk1/doris,medium:hdd,capacity:50;/home/disk2/doris,medium:ssd,capacity:50`

    **说明**

    - /home/disk1/doris,medium:hdd,capacity:10，表示存储限制为10GB, HHD;
    - /home/disk2/doris,medium:ssd,capacity:50，表示存储限制为50GB, SSD;
  
- 在 FE 中添加所有 BE 节点
  

BE 节点需要先在 FE 中添加，才可加入集群。可以使用 mysql-client(下载MySQL 5.7 (opens new window)) 连接到 FE：

```sql
#mysql -u root  -h  192.168.1.10 -P  9030
mysql> ALTER SYSTEM ADD BACKEND "192.168.1.13:9050";
mysql> ALTER SYSTEM ADD BACKEND "192.168.1.14:9050";
mysql> ALTER SYSTEM ADD BACKEND "192.168.1.15:9050";
mysql> ALTER SYSTEM ADD BACKEND "192.168.1.16:9050";
mysql> ALTER SYSTEM ADD BACKEND "192.168.1.17:9050";
```

其中 host 为 FE 所在节点 ip；port 为 fe/conf/fe.conf 中的 query_port；默认使用 root 账户，无密码登录。

登录后，执行以下命令来添加每一个 BE：

ALTER SYSTEM ADD BACKEND "host:port";

	其中 host 为 BE 所在节点 ip；port 为 be/conf/be.conf 中的 heartbeat_service_port。

  - 启动 BE

    在每台机器的BE安装目录下执行下面的命令启动BE

```shell
sh bin/start_be.sh --daemon
```

BE 进程将启动并进入后台执行。日志默认存放在 be/log/ 目录下。如启动失败，可以通过查看 be/log/be.log 或者 be/log/be.out 查看错误信息。

  - 查看BE状态

使用 mysql-client 连接到 FE，并执行 SHOW PROC '/backends'; 查看 BE 运行情况。如一切正常，isAlive 列应为 true。

```sql
#mysql -u root  -h  192.168.1.10 -P  9030

mysql> show proc '/backends';
+-----------+-----------------+---------------+----------+---------------+--------+----------+----------+---------------------+---------------------+-------+----------------------+--------------------
| BackendId | Cluster         | IP            | HostName | HeartbeatPort | BePort | HttpPort | BrpcPort | LastStartTime       | LastHeartbeat       | Alive | SystemDecommissioned | ClusterDecommission
+-----------+-----------------+---------------+----------+---------------+--------+----------+----------+---------------------+---------------------+-------+----------------------+--------------------
| 36728047  | default_cluster | 192.168.1.13 | doris-be-01  | 9050          | 9060   | 8040     | 8060     | 2021-07-15 10:21:42 | 2021-09-16 15:54:29 | true  | false                | false              
| 36728048  | default_cluster | 192.168.1.14 | doris-be-02  | 9050          | 9060   | 8040     | 8060     | 2021-07-15 10:22:44 | 2021-09-16 15:54:29 | true  | false                | false              
| 36728049  | default_cluster | 192.168.1.15 | doris-be-03  | 9050          | 9060   | 8040     | 8060     | 2021-07-15 10:23:32 | 2021-09-16 15:54:29 | true  | false                | false              
| 36728050  | default_cluster | 192.168.1.16 | doris-be-04  | 9050          | 9060   | 8040     | 8060     | 2021-07-15 10:24:12 | 2021-09-16 15:54:29 | true  | false                | false              
| 36728051  | default_cluster | 192.168.1.17 | doris-be-05  | 9050          | 9060   | 8040     | 8060     | 2021-07-15 10:25:22 | 2021-09-16 15:54:29 | true  | false                | false                         
+-----------+-----------------+---------------+----------+---------------+--------+----------+----------+---------------------+---------------------+-------+----------------------+--------------------
5 rows in set (0.00 sec)

```

#### 2.4.4 Doris FE 高可用配置

可以通过将 FE 扩容至 3 个以上节点来实现 FE 的高可用。

用户可以通过 mysql 客户端登陆 Master FE。通过:

```
SHOW PROC '/frontends';
```

来查看当前 FE 的节点情况。

FE 节点的扩容和缩容过程，不影响当前系统运行。

**增加 FE 节点**

FE 分为 Leader，Follower 和 Observer 三种角色。 默认一个集群，只能有一个 Leader，可以有多个 Follower 和 Observer。其中 Leader 和 Follower 组成一个 Paxos 选择组，如果 Leader 宕机，则剩下的 Follower 会自动选出新的 Leader，保证写入高可用。Observer 同步 Leader 的数据，但是不参加选举。如果只部署一个 FE，则 FE 默认就是 Leader。

第一个启动的 FE 自动成为 Leader。在此基础上，可以添加若干 Follower 和 Observer。

添加 Follower 或 Observer。使用 mysql-client 连接到已启动的 FE，并执行：

```
ALTER SYSTEM ADD FOLLOWER "host:port";
```

或

```
ALTER SYSTEM ADD OBSERVER "host:port";
```

这里我们全部使用的FOLLOWER角色

```sql
#mysql -u root  -h  192.168.1.10 -P  9030

mysql> SHOW PROC "/frontends";
+----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
| Name                             | IP            | HostName | EditLogPort | HttpPort | QueryPort | RpcPort | Role     | IsMaster | ClusterId  | Join | Alive | ReplayedJournalId | LastHeartbeat       | IsHelper | ErrMsg | Version         |
+----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
| 192.168.1.10_9010_1605850067231 | 192.168.1.10 | doris-fe-01  | 9010        | 8030 | 9030      | 9020    | FOLLOWER | true     | 2113522669 | true | true  | 29781119          | 2021-09-16 16:09:59 | true     |        | 0.14.13-Unknown |
+----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
1 row in set (0.04 sec)
mysql> ALTER SYSTEM DROP FOLLOWER "192.168.1.11:9010";
mysql> ALTER SYSTEM DROP FOLLOWER "192.168.1.12:9010";

```

其中 host 为 Follower 或 Observer 所在节点 ip，port 为其配置文件 fe.conf 中的 edit_log_port。

配置及启动 Follower 或 Observer。Follower 和 Observer 的配置同 Leader 的配置。第一次启动时，需执行以下命令：

```
./bin/start_fe.sh --helper host:port --daemon
```

其中 host 为 Leader 所在节点 ip, port 为 Leader 的配置文件 fe.conf 中的 edit_log_port。--helper 参数仅在 follower 和 observer 第一次启动时才需要。

查看 Follower 或 Observer 运行状态。使用 mysql-client 连接到任一已启动的 FE，并执行：

```sql
mysql> SHOW PROC '/frontends'; 
+----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
| Name                             | IP            | HostName | EditLogPort | HttpPort | QueryPort | RpcPort | Role     | IsMaster | ClusterId  | Join | Alive | ReplayedJournalId | LastHeartbeat       | IsHelper | ErrMsg | Version         |
+----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
| 192.168.1.10_9010_1605850067231 | 192.168.1.10 | doris-fe-01  | 9010        | 8030 | 9030      | 9020    | FOLLOWER | true     | 2113522669 | true | true  | 29781119          | 2021-09-16 16:09:59 | true     |        | 0.14.13-Unknown |
| 192.168.1.11_9010_1605850067231 | 192.168.1.11 | doris-fe-02  | 9010        | 8030 | 9030      | 9020    | FOLLOWER | true     | 2113522669 | true | true  | 29781119          | 2021-09-16 16:09:59 | true     |        | 0.14.13-Unknown |
| 192.168.1.12_9010_1605850067231 | 192.168.1.12 | doris-fe-03  | 9010        | 8030 | 9030      | 9020    | FOLLOWER | true     | 2113522669 | true | true  | 29781119          | 2021-09-16 16:09:59 | true     |        | 0.14.13-Unknown |
+----------------------------------+---------------+----------+-------------+----------+-----------+---------+----------+----------+------------+------+-------+-------------------+---------------------+----------+--------+-----------------+
```

可以查看当前已加入集群的 FE 及其对应角色

#### 2.4.5 Doris FE缩容

1. 停止对应节点上FE服务，
2. 使用以下命令删除对应的 FE 节点：

```
ALTER SYSTEM DROP FOLLOWER[OBSERVER] "fe_host:edit_log_port";
```

> FE 缩容注意事项：
>
> 1. 删除 Follower FE 时，确保最终剩余的 Follower（包括 Leader）节点为奇数。

## 3.安装完成	

这样整个集群就安装部署完成了

![image-20210916162201639](/images/install/image-20210916162201639.png)



![image-20210916162131985](/images/install/image-20210916162131985.png)



![image-20210916162050287](/images/install/image-20210916162050287.png)
