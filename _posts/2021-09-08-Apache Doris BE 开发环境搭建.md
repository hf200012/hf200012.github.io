---
layout: post
title: "Apache Doris BE 开发环境搭建"
date: 2021-09-08
description: "Apache Doris BE 开发环境搭建"
tag: Apache Doris
---
# Apache Doris BE 开发环境搭建

##  前期准备工作

**本教程是在Ubuntu 20.04下进行的**

1. 下载doris源代码

   下载地址为：[apache/incubator-doris: Apache Doris (Incubating) (github.com)](https://github.com/apache/incubator-doris)

2. 安装GCC 8.3.1+，Oracle JDK 1.8+，Python 2.7+，确认 gcc, java, python 命令指向正确版本, 设置 JAVA_HOME 环境变量

1. 安装其他依赖包

```html
sudo apt install build-essential openjdk-8-jdk maven cmake byacc flex automake libtool-bin bison binutils-dev libiberty-dev zip unzip libncurses5-dev curl git ninja-build python brotli
sudo add-apt-repository ppa:ubuntu-toolchain-r/ppa
sudo apt update
sudo apt install gcc-10 g++-10 
sudo apt-get install autoconf automake libtool autopoint
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

安装openssl-devel:

```html
sudo apt install -y openssl-devel
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 编译

以下操作步骤在/home/zhangfeng目录下进行

1. 下载源码

```html
git clone https://github.com/apache/incubator-doris.git 
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

1. 编译第三方依赖包

```html
 cd /home/zhangfeng/incubator-doris/thirdparty
 ./build-thirdparty.sh
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

1. 编译doris产品代码

```html
cd /home/zhangfeng/incubator-doris
./build.sh
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

注意：这个编译有以下几条指令：

```html
./build.sh  #同时编译be 和fe
./build.sh  --be #只编译be
./build.sh  --fe #只编译fe
./build.sh  --fe --be#同时编译be fe
./build.sh  --fe --be --clean#删除并同时编译be fe
./build.sh  --fe  --clean#删除并编译fe
./build.sh  --be  --clean#删除并编译be
./build.sh  --be --fe  --clean#删除并同时编译be fe
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

如果不出意外，应该会编译成功，最终的部署文件将产出到 /home/zhangfeng/incubator-doris/output/ 目录下。如果还遇到其他问题，可以参照doris的安装文档http://doris.apache.org。

## 部署调试

1. 给be编译结果文件授权

```html
chmod  /home/zhangfeng/incubator-doris/output/be/lib/palo_be
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

注意： /home/zhangfeng/incubator-doris/output/be/lib/palo_be为be的执行文件。

1. 创建数据存放目录

通过查看/home/zhangfeng/incubator-doris/output/be/conf/be.conf

```html
# INFO, WARNING, ERROR, FATAL
sys_log_level = INFO
be_port = 9060
be_rpc_port = 9070
webserver_port = 8040
heartbeat_service_port = 9050
brpc_port = 8060

# Note that there should at most one ip match this list.
# If no ip match this rule, will choose one randomly.
# use CIDR format, e.g. 10.10.10.0/
# Default value is empty.
priority_networks = 192.168.59.0/24 # data root path, seperate by ';'
storage_root_path = /soft/be/storage 
# sys_log_dir = ${PALO_HOME}/log
# sys_log_roll_mode = SIZE-MB-
# sys_log_roll_num =
# sys_log_verbose_modules =
# log_buffer_level = -
# palo_cgroups 
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

需要创建这两个文件夹，这是be数据存放的地方

```html
mkdir -p /soft/be/storage
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

1. 打开vscode，并打开be源码所在目录，在本案例中打开目录为**/home/workspace/incubator-doris/**
2. 安装vscode ms c++调试插件

![img](https://img-blog.csdnimg.cn/20210618105205864.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hmMjAwMDEy,size_16,color_FFFFFF,t_70)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

1. 创建launch.json文件，文件内容如下：

```html
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "/home/workspace/incubator-doris/output/be/lib/palo_be",
            "args": [],
            "stopAtEntry": false,
            "cwd": "/home/workspace/incubator-doris/",
            "environment": [{"name":"PALO_HOME","value":"/home/zhangfeng/incubator-doris/output/be/"},
                            {"name":"UDF_RUNTIME_DIR","value":"/home/zhangfeng/incubator-doris/output/be/lib/udf-runtime"},
                            {"name":"LOG_DIR","value":"/home/zhangfeng/incubator-doris/output/be/log"},
                            {"name":"PID_DIR","value":"/home/zhangfeng/incubator-doris/output/be/bin"}
                           ],
            "externalConsole": true,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

其中，environment定义了几个环境变量DORIS_HOME UDF_RUNTIME_DIR LOG_DIR PID_DIR，这是palo_be运行时需要的环境变量，如果没有设置，启动会失败。

**注意：如果希望是attach(附加进程）调试，配置代码如下：**

```html
{
    "version": "0.2.0",
    "configurations": [
        {
          "name": "(gdb) Launch",
          "type": "cppdbg",
          "request": "attach",
          "program": "/home/zhangfeng/incubator-doris/output/lib/palo_be",
          "processId":,
          "MIMode": "gdb",
          "internalConsoleOptions":"openOnSessionStart",
          "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

配置中 **"request": "attach",** **"processId":17016**，这两个配置节是重点： 分别设置gdb的调试模式为attach，附加进程的processId，否则会失败。如何查找进程id，可以在命令行中输入以下命令：

```html
ps -ef | grep palo*
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

如图：



其中的15200即为当前运行的be的进程id.

一个完整的lainch.json的例子如下：

```html
 {
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(gdb) Attach",
            "type": "cppdbg",
            "request": "attach",
            "program": "/home/zhangfeng/incubator-doris/output/be/lib/palo_be",
            "processId": 17016,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        },
        {
            "name": "(gdb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "program": "/home/zhangfeng/incubator-doris/output/be/lib/palo_be",
            "args": [],
            "stopAtEntry": false,
            "cwd": "/home/zhangfeng/incubator-doris/output/be",
            "environment": [
                {
                    "name": "DORIS_HOME",
                    "value": "/home/zhangfeng/incubator-doris/output/be"
                },
                {
                    "name": "UDF_RUNTIME_DIR",
                    "value": "/home/zhangfeng/incubator-doris/output/be/lib/udf-runtime"
                },
                {
                    "name": "LOG_DIR",
                    "value": "/home/zhangfeng/incubator-doris/output/be/log"
                },
                {
                    "name": "PID_DIR",
                    "value": "/home/zhangfeng/incubator-doris/output/be/bin"
                }
            ],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

6.点击调试即可

![img](https://img-blog.csdnimg.cn/20210618100622505.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2hmMjAwMDEy,size_16,color_FFFFFF,t_70)

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
