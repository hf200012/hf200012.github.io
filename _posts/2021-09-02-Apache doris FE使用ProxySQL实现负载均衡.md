---
layout: post
title: "Apache Doris FE使用ProxySQL实现负载均衡"
date: 2021-09-02
description: "Apache Doris FE使用ProxySQL实现负载均衡"
tag: Apache Doris
---
ProxySQL是灵活强大的MySQL代理层, 是一个能实实在在用在生产环境的MySQL中间件，可以实现读写分离，支持 Query 路由功能，支持动态指定某个 SQL 进行 cache，支持动态加载配置、故障切换和一些 SQL的过滤功能。

ProxySQL的优缺点，这里我就不说了，我只介绍怎么安装使用

## ProxySQL安装（yum方式）

```text
[root@mysql-proxy ~]# vim /etc/yum.repos.d/proxysql.repo
[proxysql_repo]
name= ProxySQL YUM repository
baseurl=http://repo.proxysql.com/ProxySQL/proxysql-1.4.x/centos/\$releasever
gpgcheck=1
gpgkey=http://repo.proxysql.com/ProxySQL/repo_pub_key
 
执行安装
[root@mysql-proxy ~]# yum clean all
[root@mysql-proxy ~]# yum makecache
[root@mysql-proxy ~]# yum -y install proxysql
  
[root@mysql-proxy ~]# proxysql --version
ProxySQL version 1.4.13-15-g69d4207, codename Truls
设置开机自启动
[root@mysql-proxy ~]# systemctl enable proxysql
[root@mysql-proxy ~]# systemctl start proxysql      
[root@mysql-proxy ~]# systemctl status proxysql
启动后会监听两个端口，
默认为6032和6033。6032端口是ProxySQL的管理端口，6033是ProxySQL对外提供服务的端口 (即连接到转发后端的真正数据库的转发端口)。
[root@mysql-proxy ~]# netstat -tunlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name  
tcp        0      0 0.0.0.0:6032            0.0.0.0:*               LISTEN      23940/proxysql    
tcp        0      0 0.0.0.0:6033            0.0.0.0:*               LISTEN
```

## **ProxySQL配置**

ProxySQL有配置文件/etc/proxysql.cnf和配置数据库文件/var/lib/proxysql/proxysql.db。**这里需要特别注意**：如果存在如果存在"proxysql.db"文件(在/var/lib/proxysql目录下)，则ProxySQL服务只有在第一次启动时才会去读取proxysql.cnf文件并解析；后面启动会就不会读取proxysql.cnf文件了！如果想要让proxysql.cnf文件里的配置在重启proxysql服务后生效(即想要让proxysql重启时读取并解析proxysql.cnf配置文件)，则需要先删除/var/lib/proxysql/proxysql.db数据库文件，然后再重启proxysql服务。这样就相当于初始化启动proxysql服务了，会再次生产一个纯净的proxysql.db数据库文件(如果之前配置了proxysql相关路由规则等，则就会被抹掉)

```text
[root@mysql-proxy ~]# egrep -v "^#|^$" /etc/proxysql.cnf
datadir="/var/lib/proxysql"                                   #数据目录
admin_variables=
{
        admin_credentials="admin:admin"  #连接管理端的用户名与密码
        mysql_ifaces="0.0.0.0:6032"    #管理端口，用来连接proxysql的管理数据库
}
mysql_variables=
{
        threads=4                                             #指定转发端口开启的线程数量
        max_connections=2048
        default_query_delay=0
        default_query_timeout=36000000
        have_compress=true
        poll_timeout=2000
        interfaces="0.0.0.0:6033"       #指定转发端口，用于连接后端mysql数据库的，相当于代理作用
        default_schema="information_schema"
        stacksize=1048576
        server_version="5.5.30"                               #指定后端mysql的版本
        connect_timeout_server=3000
        monitor_username="monitor"
        monitor_password="monitor"
        monitor_history=600000
        monitor_connect_interval=60000
        monitor_ping_interval=10000
        monitor_read_only_interval=1500
        monitor_read_only_timeout=500
        ping_interval_server_msec=120000
        ping_timeout_server=500
        commands_stats=true
        sessions_sort=true
        connect_retries_on_failure=10
}
mysql_servers =
(
)
mysql_users:
(
)
mysql_query_rules:
(
)
scheduler=
(
)
mysql_replication_hostgroups=
(
)
连接ProxySQL管理端口
[root@mysql-proxy ~]# mysql -uadmin -padmin -P6032 -hdoris01
查看main库（默认登陆后即在此库）的global_variables表信息
MySQL [(none)]> show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |                                     |
| 2   | disk          | /var/lib/proxysql/proxysql.db       |
| 3   | stats         |                                     |
| 4   | monitor       |                                     |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db |
+-----+---------------+-------------------------------------+
5 rows in set (0.000 sec)
MySQL [(none)]> use main;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A
 
Database changed
MySQL [main]> show tables;
+--------------------------------------------+
| tables                                     |
+--------------------------------------------+
| global_variables                           |
| mysql_collations                           |
| mysql_group_replication_hostgroups         |
| mysql_query_rules                          |
| mysql_query_rules_fast_routing             |
| mysql_replication_hostgroups               |
| mysql_servers                              |
| mysql_users                                |
| proxysql_servers                           |
| runtime_checksums_values                   |
| runtime_global_variables                   |
| runtime_mysql_group_replication_hostgroups |
| runtime_mysql_query_rules                  |
| runtime_mysql_query_rules_fast_routing     |
| runtime_mysql_replication_hostgroups       |
| runtime_mysql_servers                      |
| runtime_mysql_users                        |
| runtime_proxysql_servers                   |
| runtime_scheduler                          |
| scheduler                                  |
+--------------------------------------------+
20 rows in set (0.000 sec)
这些表的含义及作用大家可以在网上搜索
```

## **ProxySQL配置后端Doris FE**

```text
使用insert语句添加主机到mysql_servers表中，其中：hostgroup_id 为10表示写组，为20表示读组，我们这里不需要读写分许，无所谓随便设置哪一个都可以，后面我会讲出现的问题及解决办法。
  
[root@mysql-proxy ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
............
MySQL [(none)]> insert into mysql_servers(hostgroup_id,hostname,port) values(10,'192.168.9.211',9030);
Query OK, 1 row affected (0.000 sec)
  
MySQL [(none)]> insert into mysql_servers(hostgroup_id,hostname,port) values(10,'192.168.9.212',9030);
Query OK, 1 row affected (0.000 sec)
  
MySQL [(none)]> insert into mysql_servers(hostgroup_id,hostname,port) values(10,'192.168.9.213',9030);
Query OK, 1 row affected (0.000 sec)
 
如果在插入过程中，出现报错：
ERROR 1045 (#2800): UNIQUE constraint failed: mysql_servers.hostgroup_id, mysql_servers.hostname, mysql_servers.port
 
说明可能之前就已经定义了其他配置，可以清空这张表 或者 删除对应host的配置
MySQL [(none)]> select * from mysql_servers;
MySQL [(none)]> delete from mysql_servers;
Query OK, 6 rows affected (0.000 sec)

查看这3个节点是否插入成功，以及它们的状态。
MySQL [(none)]> select * from mysql_servers\G;
*************************** 1. row ***************************
       hostgroup_id: 10
           hostname: 192.168.9.211
               port: 9030
             status: ONLINE
             weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
     max_latency_ms: 0
            comment:
*************************** 2. row ***************************
       hostgroup_id: 10
           hostname: 192.168.9.212
               port: 9030
             status: ONLINE
             weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
     max_latency_ms: 0
            comment:
*************************** 3. row ***************************
       hostgroup_id: 10
           hostname: 192.168.9.213
               port: 9030
             status: ONLINE
             weight: 1
        compression: 0
    max_connections: 1000
max_replication_lag: 0
            use_ssl: 0
     max_latency_ms: 0
            comment:
6 rows in set (0.000 sec)
  
ERROR: No query specified
  
如上修改后，加载到RUNTIME，并保存到disk
MySQL [(none)]> load mysql servers to runtime;
Query OK, 0 rows affected (0.006 sec)
  
MySQL [(none)]> save mysql servers to disk;
Query OK, 0 rows affected (0.348 sec)
```

## **监控后端doris节点**

添doris fe节点之后，还需要监控这些后端节点。对于后端多个FE高可用负载均衡环境来说，这是必须的，因为ProxySQL需要通过每个节点的read_only值来自动调整

它们是属于读组还是写组。

首先在后端master主数据节点上创建一个用于监控的用户名

```text
在doris fe master主数据库节点行执行：
[root@doris01 ~]# mysql -P9030 -uroot -p 
mysql> create user monitor@'192.168.9.%' identified by 'P@ssword1!';
Query OK, 0 rows affected (0.03 sec)
mysql> grant ADMIN_PRIV on *.* to monitor@'192.168.9.%';
Query OK, 0 rows affected (0.02 sec)
 
然后回到mysql-proxy代理层节点上配置监控
[root@mysql-proxy ~]# mysql -uadmin -padmin -P6032 -h127.0.0.1
MySQL [(none)]> set mysql-monitor_username='monitor';
Query OK, 1 row affected (0.000 sec)
 
MySQL [(none)]> set mysql-monitor_password='P@ssword1!';
Query OK, 1 row affected (0.000 sec)
 
修改后，加载到RUNTIME，并保存到disk
MySQL [(none)]> load mysql variables to runtime;
Query OK, 0 rows affected (0.001 sec)
 
MySQL [(none)]> save mysql variables to disk;
Query OK, 94 rows affected (0.079 sec)
 
验证监控结果：ProxySQL监控模块的指标都保存在monitor库的log表中。
以下是连接是否正常的监控(对connect指标的监控)：
注意：可能会有很多connect_error，这是因为没有配置监控信息时的错误，配置后如果connect_error的结果为NULL则表示正常。
MySQL [(none)]> select * from mysql_server_connect_log;
+---------------+------+------------------+-------------------------+---------------+
| hostname      | port | time_start_us    | connect_success_time_us | connect_error |
+---------------+------+------------------+-------------------------+---------------+
| 192.168.9.211 | 9030 | 1548665195883957 | 762                     | NULL          |
| 192.168.9.212 | 9030 | 1548665195894099 | 399                     | NULL          |
| 192.168.9.213 | 9030 | 1548665195904266 | 483                     | NULL          |
| 192.168.9.211 | 9030 | 1548665255883715 | 824                     | NULL          |
| 192.168.9.212 | 9030 | 1548665255893942 | 656                     | NULL          |
| 192.168.9.211 | 9030 | 1548665495884125 | 615                     | NULL          |
| 192.168.9.212 | 9030  | 1548665495894254 | 441                     | NULL          |
| 192.168.9.213 | 9030 | 1548665495904479 | 638                     | NULL          |
| 192.168.9.211 | 9030 | 1548665512917846 | 487                     | NULL          |
| 192.168.9.212 | 9030 | 1548665512928071 | 994                     | NULL          |
| 192.168.9.213 | 9030 | 1548665512938268 | 613                     | NULL          |
+---------------+------+------------------+-------------------------+---------------+
20 rows in set (0.000 sec)
以下是对心跳信息的监控(对ping指标的监控)
MySQL [(none)]> select * from mysql_server_ping_log;
+---------------+------+------------------+----------------------+------------+
| hostname      | port | time_start_us    | ping_success_time_us | ping_error |
+---------------+------+------------------+----------------------+------------+
| 192.168.9.211 | 9030 | 1548665195883407 | 98                   | NULL       |
| 192.168.9.212 | 9030 | 1548665195885128 | 119                  | NULL       |
...........
| 192.168.9.213 | 9030 | 1548665415889362 | 106                  | NULL       |
| 192.168.9.213 | 9030 | 1548665562898295 | 97                   | NULL       |
+---------------+------+------------------+----------------------+------------+
110 rows in set (0.001 sec)
 
read_only日志此时也为空(正常来说，新环境配置时，这个只读日志是为空的)
MySQL [(none)]> select * from mysql_server_read_only_log;
Empty set (0.000 sec)

3个节点都在hostgroup_id=10的组中。
现在，将刚才mysql_replication_hostgroups表的修改加载到RUNTIME生效。
MySQL [(none)]> load mysql servers to runtime;
Query OK, 0 rows affected (0.003 sec)
 
MySQL [(none)]> save mysql servers to disk;
Query OK, 0 rows affected (0.361 sec)

现在看结果
MySQL [(none)]> select hostgroup_id,hostname,port,status,weight from mysql_servers;
+--------------+---------------+------+--------+--------+
| hostgroup_id | hostname      | port | status | weight |
+--------------+---------------+------+--------+--------+
| 10           | 192.168.9.211 | 9030 | ONLINE | 1      |
| 20           | 192.168.9.212 | 9030 | ONLINE | 1      |
| 20           | 192.168.9.213 | 9030 | ONLINE | 1      |
+--------------+---------------+------+--------+--------+
3 rows in set (0.000 sec)
```

**配置mysql_users**

上面的所有配置都是关于后端MySQL节点的，现在可以配置关于SQL语句的，包括：发送SQL语句的用户、SQL语句的路由规则、SQL查询的缓存、SQL语句的重写等等。本小节是SQL请求所使用的用户配置，例如root用户。这要求我们需要先在后端Doris FE节点添加好相关用户。这里以root和doris两个用户名为例.

```text
首先，在Doris FE master主数据库节点上执行：[root@doris01 ~]# mysql -P9030 -uroot -p
.........
root用户已经存在，我们直接创建doris用户
mysql> create user doris@'%' identified by 'P@ssword1!';
Query OK, 0 rows affected, 1 warning (0.04 sec)
 
mysql> grant ADMIN_PRIV on *.* to doris@'%';
Query OK, 0 rows affected, 1 warning (0.03 sec)
 
 
然后回到mysql-proxy代理层节点，配置mysql_users表，将刚才的两个用户添加到该表中。
admin> insert into mysql_users(username,password,default_hostgroup) values('root','',10);
Query OK, 1 row affected (0.001 sec)
  
admin> insert into mysql_users(username,password,default_hostgroup) values('doris','P@ssword1!',10);
Query OK, 1 row affected (0.000 sec)
  
admin> load mysql users to runtime;
Query OK, 0 rows affected (0.001 sec)
  
admin> save mysql users to disk;
Query OK, 0 rows affected (0.108 sec)
  
mysql_users表有不少字段，最主要的三个字段为username、password和default_hostgroup：
-  username：前端连接ProxySQL，以及ProxySQL将SQL语句路由给MySQL所使用的用户名。
-  password：用户名对应的密码。可以是明文密码，也可以是hash密码。如果想使用hash密码，可以先在某个MySQL节点上执行
   select password(PASSWORD)，然后将加密结果复制到该字段。
-  default_hostgroup：该用户名默认的路由目标。例如，指定root用户的该字段值为10时，则使用root用户发送的SQL语句默认
   情况下将路由到hostgroup_id=10组中的某个节点。
 
admin> select * from mysql_users\G
*************************** 1. row ***************************
              username: root
              password: 
                active: 1
               use_ssl: 0
     default_hostgroup: 10
        default_schema: NULL
         schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
               backend: 1
              frontend: 1
       max_connections: 10000
*************************** 2. row ***************************
              username: doris
              password: P@ssword1!
                active: 1
               use_ssl: 0
     default_hostgroup: 10
        default_schema: NULL
         schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
               backend: 1
              frontend: 1
       max_connections: 10000
2 rows in set (0.000 sec)
  
虽然这里没有详细介绍mysql_users表，但上面标注了"注意本行"的两个字段必须要引起注意。只有active=1的用户才是有效的用户。
至于transaction_persistent字段，当它的值为1时，表示事务持久化：当某连接使用该用户开启了一个事务后，那么在事务提交/回滚之前，
所有的语句都路由到同一个组中，避免语句分散到不同组。在以前的版本中，默认值为0，不知道从哪个版本开始，它的默认值为1。
我们期望的值为1，所以在继续下面的步骤之前，先查看下这个值，如果为0，则执行下面的语句修改为1。
 
MySQL [(none)]> update mysql_users set transaction_persistent=1 where username='root';
Query OK, 1 row affected (0.000 sec)
 
MySQL [(none)]> update mysql_users set transaction_persistent=1 where username='sqlsender';
Query OK, 1 row affected (0.000 sec)
 
MySQL [(none)]> load mysql users to runtime;
Query OK, 0 rows affected (0.001 sec)
 
MySQL [(none)]> save mysql users to disk;
Query OK, 0 rows affected (0.123 sec)

这样就可以通过sql客户端，使用doris的用户名密码去连接了ProxySQL了
```

## 通过ProxySQL连接Doris测试

下面，分别使用root用户和doris用户测试下它们是否能路由到默认的hostgroup_id=10(它是一个写组)读数据。下面是通过转发端口6033连接的，连接的是转发到后端真正的数据库!

```text
[root@mysql-master ~]#mysql -uroot -p -P6033 -hdoris01 -e "show databases;"
Enter password: 
ERROR 9001 (HY000) at line 1: Max connect timeout reached while reaching hostgroup 10 after 10000ms
这个时候发现出错，并没有转发到后端真正的doris fe上
通过日志看到有set autocommit=0这样开启事务
检查配置发现：
mysql-forward_autocommit=false
mysql-autocommit_false_is_transaction=false
我们这里不需要读写分离，只需要将这两个参数通过下面语句直接搞成true就可以了
mysql> UPDATE global_variables SET variable_value='true' WHERE variable_name='mysql-forward_autocommit';
Query OK, 1 row affected (0.00 sec)

mysql> UPDATE global_variables SET variable_value='true' WHERE variable_name='mysql-autocommit_false_is_transaction';
Query OK, 1 row affected (0.01 sec)

mysql>  LOAD MYSQL VARIABLES TO RUNTIME;
Query OK, 0 rows affected (0.00 sec)

mysql> SAVE MYSQL VARIABLES TO DISK;
Query OK, 98 rows affected (0.12 sec)

然后我们在重新试一下，显示成功
[root@doris01 ~]# mysql -udoris -pP@ssword1! -P6033 -h192.168.9.211  -e "show databases;"
Warning: Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| doris_audit_db     |
| information_schema |
| retail             |
+--------------------+
```

OK，到此就结束了，你就可以用Mysql客户端，JDBC等任何连接mysql的方式连接ProxySQL去操作你的doris了

**备注：在使用jdbc连接池的时候，mysql的驱动请使用8.0.15，我试过好几个版本都会出错，这个版本没有问题，出错信息如下**

**java.sql.SQLException: Unknown system variable 'performance_schema''**