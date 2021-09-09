---
layout: post
title: "Apache Doris ODBC mysql外表注意事项"
date: 2021-09-09 
description: "Apache Doris ODBC mysql外表注意事项"
tag: Apache Doris
---

 前面一篇文章介绍了[Apache doris ODBC外表使用方式](https://hf200012.github.io/2021/09/Apache-doris-ODBC外表使用方式/)，这里要说的是在使用Mysql的ODBC外表的时候要注意事项：

1. mysql数据库及表的字符集一定要是用UTF8，不要使用UTF8mb4，目前doris ODBC外表只支持UTF8编码
2. 在doris BE节点配置conf/odbcinst.ini，这里配置

```
[MySQL Driver]
Description     = ODBC for MySQL
Driver          = /usr/lib/libmyodbc8w.so
FileUsage       = 1
```

这里一定不要使用libmyodbc8a.so  要使用 libmyodbc8w.so 否则会出现中文查询不到结果的问题，这里是因为：

```
当前ODBC支持ANSI 与 Unicode 两种Driver形式，当前Doris只支持Unicode Driver。
如果强行使用ANSI Driver可能会导致查询结果出错。
```

如果使用了libmyodbc8a.so ,BE转发过去的sql语句会出现乱码，这个可以通过Mysql的general log 看到。

下图是个示例，本来我传入的是一个中文城市名称，但是因为使用了libmyodbc8a.so ,BE转发到Mysql大的语句做了转码，就出现了中文乱码问题，改成libmyodbc8w.so 后一切问题解决，修改这个不需要重启BE

![img](https://pic1.zhimg.com/v2-c37be6873570ba4550108556464879d8_b.png)
