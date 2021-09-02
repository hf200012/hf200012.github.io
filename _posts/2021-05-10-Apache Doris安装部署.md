---
layout: post
title: "Apache Doris安装部署"
date: 2021-05-10 
description: "Apache Doris安装部署"
tag: Apache Doris
---

此文档是笔记方式，没有详细整理，仅供参考,目前社区版本是0.14.0，百度预编译发布版本是0.14.12.4

如果有疑问，可以留言或者加我微信，微信在我个人站里可以找到

### **安装部署**

```text
1.下载Doris的安装包
cd /opt
wget https://dist.apache.org/repos/dist/dev/incubator/doris/0.12.0-rc03/apache-doris-0.12.
0-incubating-src.tar.gz
解压安装
tar -zxvf apache-doris-0.12.0-incubating-src.tar.gz
cd apache-doris-0.12.0-incubating-src
sh build.sh

2.配置该节点的FE（Leader）
cd output/fe
mkdir doris-meta
mkdir log
sh bin/start_fe.sh --daemon
运行之后检查一下，是否有doris的进行，监听的端口，日志信息等等
vi log/fe.log

3.配置BE
cd output/be
mkdir storage
mkdir log

4.分发到所有需要安装的BE节点 scp -r output/be root@主机名：/

5.安装mysql客户端
1，从官网下载安装包（在Centos7上要下载 RH Linux 7 的安装包）
https://dev.mysql.com/downloads/mysql/
mysql-8.0.17-1.el7.x86_64.rpm-bundle.tar
2，清理环境
2.1 查看系统是否已经安装了mysql数据库
rpm -qa | grep mysql
2.2 将查询出的文件逐个删除，如
yum remove mysql-community-common-5.7.20-1.el6.x86_64
2.3 删除mysql的配置文件
find / -name mysql
2.4 删除配置文件
rm -rf /var/lib/mysql
2.5删除MariaDB文件
rpm -pa | grep mariadb
删除查找出的相关文件和目录，如
yum -y remove mariadb-libs.x86_64
3，安装
3.1解压
tar -xf mysql-8.0.17-1.el7.x86_64.rpm-bundle.tar
3.2安装
yum install mysql-community-{client,common,devel,embedded,libs,server}-*
等待安装成功！
4，配置
4.1 启动mysqld服务，并设为开机自动启动。命令：
systemctl start mysqld.service //这是centos7的命令
systemctl enable mysqld.service
4.2 通过如下命令可以在日志文件中找出密码：
grep "password" /var/log/mysqld.log
4.3按照日志文件中的密码，进入数据库
mysql -uroot -p
4.4设置密码（注意Mysql8密码设置规则必须是大小写字母+特殊符号+数字的类型）
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new password';

6.远程连接doris服务
mysql -uroot -h 172.22.197.72 -P 9030

7.添加所有BE
ALTER SYSTEM ADD BACKEND "172.22.197.73:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.74:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.75:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.76:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.77:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.78:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.79:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.80:9050";
ALTER SYSTEM ADD BACKEND "172.22.197.81:9050";
#删除BE节点，数据会同步到其他节点
ALTER SYSTEM DECOMMISSION BACKEND "172.22.197.73:9050";
#删除BE节点，该节点数据直接删除
ALTER SYSTEM DECOMMISSION BACKEND "172.22.197.73:9050";

8.启动BE节点
sh bin/start-be.sh --daemon

9.ui界面查看是否添加进来
http://172.22.197.72:8030/system?path=//backends

10.添加brokername
ALTER SYSTEM ADD BROKER broker_name01 "test-pro-doris-01:8000";
#删除
ALTER SYSTEM DROP BROKER broker_name "test-pro-doris-01:8000";
11.ui界面查看是否添加成功
http://172.22.197.72:8030/system?path=//brokers
```



### **doris ODBC load**

### **1.在线安装MYSQL ODBC驱动**

```text
 yum -y install unixODBC
 yum -y install mysql-connector-odbc
 
 遇到问题：yum -y install mysql-connector-odbc 安装不成功
 
 解决方法：下载jar mysql-connector-odbc-8.0.11-1.el7.x86_64.rpm进行本地安装
 
 yum localinstall mysql-connector-odbc-8.0.11-1.el7.x86_64.rpm
```



### **2.配置Mysql驱动**

```text
 cat /etc/odbc.ini #添加如下信息
 /************************************************
 [mysql-hr]
 Driver = /usr/lib64/libmyodbc8a.so #注意驱动程序的选择
 Description = MyODBC 5 Driver 
 SERVER = 192.168.235.140    #要连接的数据库信息
 PORT = 3306
 USER = root
 Password = root
 Database = hr
 OPTION = 3
 charset=UTF8
```

### **3.测试连接**

```sql
 # isql mysql-hr test root password -v ##语法：isql 数据源名称 用户名 密码 选项
 +---------------------------------------+
 | Connected! |
 | |password
 | sql-statement |
 | help [tablename] |
 | quit |
 | |
 +---------------------------------------+
 SQL>show database;
 测试成功
```

### **4.配置FE**

```text
 vim /doris-0.13.11/output/be/conf/fe.conf
 enable_odbc_table = true  必配项
```

### **5.配置BE（所有BE节点都需要配置）**

```text
 vim /doris-0.13.11/output/be/conf/odbcinst.ini 添加
 [MySQL Driver]
 Description     = ODBC for MySQL
 Driver          = /usr/lib/libmyodbc8a.so
 FileUsage       = 1
 说明：driver ODBC安装的目录
```

### **6.测试ODBC on doris**

```mysql
 推荐方式：
 
 ##### 1.通过ODBC_Resource来创建ODBC外表
 CREATE EXTERNAL RESOURCE `mysql_odbc_doris`
 PROPERTIES (
 "type" = "odbc_catalog",
 "host" = "172.22.193.65",
 "port" = "3306",
 "user" = "root",
 "password" = "password",
 "database" = "posresult",
 "odbc_type" = "mysql",
 "driver" = "MySQL Driver"
 );
 说明：
      host需要连接的数据库ip（映射库的ip）
      port端口
      user用户名
      password密码
      database数据库
      odbc_type：mysql(支持oracle, mysql, postgresql)
      driver：ODBC外表的Driver名，该名字需要和be/conf/odbcinst.ini中的Driver名一致
 #####2.创建DORIS外部表映射MYSQL表
 CREATE EXTERNAL TABLE `test_mysql` (
   `id` varchar(32) NOT NULL COMMENT 'ID',
   `table_bill_id` varchar(36) DEFAULT NULL COMMENT '菜单编号',
   `shop_id` varchar(32) DEFAULT NULL COMMENT '门店ID',
   `dish_type` int(11) DEFAULT NULL COMMENT '类型 ： 1-菜品 2-火锅 3-底料',
   `dish_id` varchar(50) DEFAULT NULL COMMENT '菜品ID（此处为菜品ID，不是菜品关联ID）',
   `dish_name` varchar(100) DEFAULT NULL COMMENT '菜品名称',
   `standard_id` varchar(32) DEFAULT NULL COMMENT '规格编码',
   `standard_code` varchar(100) DEFAULT NULL COMMENT '规格ID',
   `dish_price` varchar(16) DEFAULT NULL COMMENT '菜品单价',
   `served_quantity` int(11) DEFAULT NULL COMMENT '已上数量',
   `order_time` varchar(50) DEFAULT NULL COMMENT '点菜时间',
   `dish_abnormal_status` varchar(20) DEFAULT NULL COMMENT '[A]菜品异常状态',
   `ts` varchar(20) DEFAULT NULL COMMENT 'POS订单创建时间',
   `taste_type_id` varchar(32) DEFAULT NULL,
   `taste_name` varchar(50) DEFAULT NULL
 ) ENGINE=ODBC
 COMMENT "ODBC"
 PROPERTIES (
 "odbc_catalog_resource" = "mysql_odbc_doris_test",
 "database" = "posresult",
 "table" = "t_pro_dish_list_detail"
 );
 说明：
 odbc_catalog_resource 创建的Resource名称
 database 外表数据库数据库名称
 table 外表数据库表名
 #####3.执行DDL操作是否插入成功
 selct * from test_mysql
```

### **7.常见错误**

```text
 1.出现错误：(10001 NOT ALIVE,10002 NOT ALIVE)
 原因：编译doris的时候没有带WITH_MYSQL,Mysql_Odbc需要8.x,如果采用5.x会出现上面错误，切换版本到8.X
      编译如果带WITH_MYSQL,可以采用5.x版本
```