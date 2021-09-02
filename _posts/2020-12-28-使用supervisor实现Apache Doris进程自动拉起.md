---
layout: post
title: "使用supervisor实现Apache Doris进程自动拉起"
date: 2020-12-28 
description: "使用supervisor实现Apache Doris进程自动拉起"
tag: 系统运维
---

 **supervisor安装**

```text
1.使用yum命令安装(推荐)
 yum install epel-release
 yum install -y supervisor
 systemctl enable supervisord # 开机自启动
 systemctl start supervisord # 启动supervisord服务
 systemctl status supervisord # 查看supervisord服务状态
 ps -ef|grep supervisord # 查看是否存在supervisord进程
2.使用pip手工安装配置(不推荐)
确认9001端口未被占用，如果9001端口被占用，请将后面提到的supervisord.conf文件中的9001替换为可用端口号，如7001。
 2.1切换为root用户
 sudo su
 2.2为python2.7安装pip（supervisor只支持python2.7）
   下载pip:# wget https://bootstrap.pypa.io/get-pip.py --no-check-certificate
   安装pip:# python2.7 get-pip.py如果遇到psutil相关的报错，参考https://www.cnblogs.com/chentq/p/4954135.html
 2.3安装supervisor
 pip2 install supervisor
#可能遇到的问题及解决办法：
#如果遇到NameError: name 'sys_platform' is not defined 
运行如下命令即可（但是也会报错，不用理会）
 pip2 install --upgrade distribute,
#然后再卸载supervisor，
 pip2 uninstall supervisor，
#重装supervisor,
 pip2 install supervisor。
# #参考http://www.yuchaoshui.com/post/CentOS6-Python-installation
2.4创建supervisor所需目录
 mkdir /etc/supervisord.d/
2.5创建supervisor配置文件
 echo_supervisord_conf > /etc/supervisord.conf
2.6编辑supervisord.conf文件
 vim /etc/supervisord.conf 添加如下内容
; Sample supervisor config file.
;
; For more information on the config file, please see:
; http://supervisord.org/configuration.html
;
; Notes:
;  - Shell expansion ("~" or "$HOME") is not supported.  Environment
;    variables can be expanded using this syntax: "%(ENV_HOME)s".
;  - Quotes around values are not supported, except in the case of
;    the environment= options as shown below.
;  - Comments must have a leading space: "a=b ;comment" not "a=b;comment".
;  - Command will be truncated if it looks like a config file comment, e.g.
;    "command=bash -c 'foo ; bar'" will truncate to "command=bash -c 'foo ".

[unix_http_server]
file=/var/run/supervisor.sock   ; the path to the socket file
;chmod=0700                 ; socket file mode (default 0700)
;chown=nobody:nogroup       ; socket file uid:gid owner
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

[inet_http_server]         ; inet (TCP) server disabled by default
port=127.0.0.1:9001        ; ip_address:port specifier, *:port for all iface
;username=user              ; default is no username (open server)
;password=123               ; default is no password (open server)

[supervisord]
logfile=/var/log/supervisord.log ; main log file; default $CWD/supervisord.log
logfile_maxbytes=50MB        ; max main logfile bytes b4 rotation; default 50MB
logfile_backups=10           ; # of main logfile backups; 0 means none, default 10
loglevel=info                ; log level; default info; others: debug,warn,trace
pidfile=/var/run/supervisord.pid ; supervisord pidfile; default supervisord.pid
nodaemon=false               ; start in foreground if true; default false
minfds=1024                  ; min. avail startup file descriptors; default 1024
minprocs=200                 ; min. avail process descriptors;default 200
;umask=022                   ; process file creation umask; default 022
;user=chrism                 ; default is current user, required if root
;identifier=supervisor       ; supervisord identifier, default is 'supervisor'
;directory=/tmp              ; default is not to cd during start
;nocleanup=true              ; don't clean up tempfiles at start; default false
;childlogdir=/tmp            ; 'AUTO' child log dir, default $TEMP
;environment=KEY="value"     ; key value pairs to add to environment
;strip_ansi=false            ; strip ansi escape codes in logs; def. false

; The rpcinterface:supervisor section must remain in the config file for
; RPC (supervisorctl/web interface) to work.  Additional interfaces may be
; added by defining them in separate [rpcinterface:x] sections.

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

; The supervisorctl section configures how supervisorctl will connect to
; supervisord.  configure it match the settings in either the unix_http_server
; or inet_http_server section.

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket
;serverurl=http://127.0.0.1:9001 ; use an http:// url to specify an inet socket
;username=chris              ; should be same as in [*_http_server] if set
;password=123                ; should be same as in [*_http_server] if set
;prompt=mysupervisor         ; cmd line prompt (default "supervisor")
;history_file=~/.sc_history  ; use readline history if available

; The sample program section below shows all possible program subsection values.
; Create one or more 'real' program: sections to be able to control them under
; supervisor.

;[program:theprogramname]
;command=/bin/cat              ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=999                  ; the relative start priority (default 999)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; when to restart if exited after running (def: unexpected)
;exitcodes=0,2                 ; 'expected' exit codes used with autorestart (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=true          ; redirect proc stderr to stdout (default false)
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
;stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
;stderr_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions (def no adds)
;serverurl=AUTO                ; override serverurl computation (childutils)

; The sample eventlistener section below shows all possible eventlistener
; subsection values.  Create one or more 'real' eventlistener: sections to be
; able to handle event notifications sent by supervisord.

;[eventlistener:theeventlistenername]
;command=/bin/eventlistener    ; the program (relative uses PATH, can take args)
;process_name=%(program_name)s ; process_name expr (default %(program_name)s)
;numprocs=1                    ; number of processes copies to start (def 1)
;events=EVENT                  ; event notif. types to subscribe to (req'd)
;buffer_size=10                ; event buffer queue size (default 10)
;directory=/tmp                ; directory to cwd to before exec (def no cwd)
;umask=022                     ; umask for process (default None)
;priority=-1                   ; the relative start priority (default -1)
;autostart=true                ; start at supervisord start (default: true)
;startsecs=1                   ; # of secs prog must stay up to be running (def. 1)
;startretries=3                ; max # of serial start failures when starting (default 3)
;autorestart=unexpected        ; autorestart if exited after running (def: unexpected)
;exitcodes=0,2                 ; 'expected' exit codes used with autorestart (default 0,2)
;stopsignal=QUIT               ; signal used to kill process (default TERM)
;stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
;stopasgroup=false             ; send stop signal to the UNIX process group (default false)
;killasgroup=false             ; SIGKILL the UNIX process group (def false)
;user=chrism                   ; setuid to this UNIX account to run the program
;redirect_stderr=false         ; redirect_stderr=true is not allowed for eventlisteners
;stdout_logfile=/a/path        ; stdout log path, NONE for none; default AUTO
;stdout_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stdout_logfile_backups=10     ; # of stdout logfile backups (0 means none, default 10)
;stdout_events_enabled=false   ; emit events on stdout writes (default false)
;stderr_logfile=/a/path        ; stderr log path, NONE for none; default AUTO
;stderr_logfile_maxbytes=1MB   ; max # logfile bytes b4 rotation (default 50MB)
;stderr_logfile_backups=10     ; # of stderr logfile backups (0 means none, default 10)
;stderr_events_enabled=false   ; emit events on stderr writes (default false)
;environment=A="1",B="2"       ; process environment additions
;serverurl=AUTO                ; override serverurl computation (childutils)

; The sample group section below shows all possible group values.  Create one
; or more 'real' group: sections to create "heterogeneous" process groups.

;[group:thegroupname]
;programs=progname1,progname2  ; each refers to 'x' in [program:x] definitions
;priority=999                  ; the relative start priority (default 999)

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisord.d/*.ini
2.7启动supervisor
supervisord -c /etc/supervisord.conf
2.8查看supervisor是否启动成功
ps -ef|grep supervisord
2.9将supervisor配置为开机自启动服务
vim /usr/lib/systemd/system/supervisord.service
[Unit]
Description=Supervisor daemon
[Service]
Type=forking
PIDFile=/var/run/supervisord.pid
ExecStart=/bin/supervisord -c /etc/supervisord.conf
ExecStop=/bin/supervisorctl shutdown
ExecReload=/bin/supervisorctl reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
2.10启动服务
systemctl enable supervisord
3.11查看是否启动
systemctl is-enabled supervisord
enabled
2.12.1意
因为我们的supervisor使用的是root安装，所以，对于非root用户，如果在执行 
$ supervisord -c /etc/supervisord.conf 
$ supervisorctl
命令时，会遇到访问被拒（Permission denied）的问题。
在命令最前面加上sudo即可
2.12.2 如果更改了/etc/supervisord.conf中的端口号，原来的简写命令:supervisorctl就需要在后面指定supervsor配置文件位置，或指定supervisor服务运行的端口号
# supervisorctl -c /etc/supervisord.conf 
# supervisorctl -s http://localhost:7001
否则会报连接拒绝
```

### **doris通过supervsor进行进程管理配置**

```text
1.配置palo be 进程管理
1.1修改各个 start_be.sh 脚本，去掉最后的 & 符号
vim /soft/doris-be/bin/start_be.sh
99行 nohup $LIMIT ${DORIS_HOME}/lib/palo_be "$@" >> $LOG_DIR/be.out 2>&1 </dev/null &
修改成nohup $LIMIT ${DORIS_HOME}/lib/palo_be "$@" >> $LOG_DIR/be.out 2>&1 </dev/null
wq保存退出
1.2创建be supervisor进程管理配置文件
vim /etc/supervisor/config.d/palo_be.ini

[program:palo_be]      
process_name=%(program_name)s      
directory=/soft/doris-be
command=sh /soft/doris-be/bin/start_be.sh
autostart=true
autorestart=true
user=root
numprocs=1
startretries=3
stopasgroup=true
killasgroup=true
startsecs=5
#redirect_stderr = true
#stdout_logfile_maxbytes = 20MB
#stdout_logfile_backups = 10
#stdout_logfile=/var/log/supervisor-palo_be.log 


2.配置broker进程管理
2.1 修改各个 start_broker.sh 脚本，去掉最后的 & 符号
vim //soft/apache_hdfs_broker/bin/start_broker.sh
83行修改 nohup $LIMIT $JAVA $JAVA_OPTS org.apache.doris.broker.hdfs.BrokerBootstrap "$@" >> $BROKER_LOG_DIR/apache_hdfs_broker.out 2>&1 </dev/null &
为nohup $LIMIT $JAVA $JAVA_OPTS org.apache.doris.broker.hdfs.BrokerBootstrap "$@" >> $BROKER_LOG_DIR/apache_hdfs_broker.out 2>&1 </dev/null
2.2创建broker supervisor进程管理配置文件
vim /etc/supervisor/config.d/palo_broker.ini
[program:BrokerBootstrap]
environment = JAVA_HOME="/usr/local/java"
process_name=%(program_name)s
directory=/soft/apache_hdfs_broker
command=sh /soft/apache_hdfs_broker/bin/start_broker.sh
autostart=true
autorestart=true
user=root
numprocs=1
startretries=3
stopasgroup=true
killasgroup=true
startsecs=5
#redirect_stderr=true
#stdout_logfile_maxbytes=20MB
#stdout_logfile_backups=10
#stdout_logfile=/var/log/supervisor-BrokerBootstrap.log

3 配置fe进程管理
3.1 修改各个 start_fe.sh 脚本，去掉最后的 & 符号
vim /soft/doris-fe/bin/start_fe.sh
147行nohup $LIMIT $JAVA $final_java_opt org.apache.doris.PaloFe ${HELPER} "$@" >> $LOG_DIR/fe.out 2>&1 </dev/null &
修改为nohup $LIMIT $JAVA $final_java_opt org.apache.doris.PaloFe ${HELPER} "$@" >> $LOG_DIR/fe.out 2>&1 </dev/null
3.2 创建fe supervisor进程管理配置文件
 vim /etc/supervisor/config.d/palo_fe.ini
 [program:PaloFe]
environment = JAVA_HOME="/usr/local/java"
process_name=PaloFe
directory=/soft/doris-fe
command=sh /soft/doris-fe/bin/start_fe.sh
autostart=true
autorestart=true
user=root
numprocs=1
startretries=3
stopasgroup=true
killasgroup=true
startsecs=10
#redirect_stderr=true
#stdout_logfile_maxbytes=20MB
#stdout_logfile_backups=10
#stdout_logfile=/var/log/supervisor-PaloFe.log

4. 验证
4.1 先确保没有palo fe，be，broker进程在运行，如果有则使用kill -9  [processid]杀死掉
[root@doris01 soft]# jps
50258 DataNode
60387 Jps
59908 PaloFe
50109 NameNode
40318 BrokerBootstrap
[root@doris01 soft]# kill -9 59908
[root@doris01 soft]# kill -9 40318
说明： BrokerBootstrap为broker的进程名称，PaloFe为fe的进程名称
4.2停止掉be
[root@doris01 soft]# ps -e | grep palo
root     14151 13619 99 12月21 ?      13:50:03 /soft/doris-be/lib/palo_be
[root@doris01 soft]# kill -9 14151
4.3 启动supervisor,验证fe，be，broker是否启动
启动supervisor
supervisord -c /etc/supervisord.conf
查看状态：
[root@doris01 soft]# supervisorctl status
BrokerBootstrap                  RUNNING   pid 64312, uptime 0:00:16
PaloFe                           RUNNING   pid 64314, uptime 0:00:16
palo_be                          RUNNING   pid 64313, uptime 0:00:16
验证fe,be,broker进程是否启动
[root@doris01 soft]# jps
50258 DataNode
63846 Jps
61548 BrokerBootstrap
50109 NameNode
60734 PaloFe
[root@doris01 soft]# ps -e | grep palo
61118 ?        00:00:01 palo_be
4.4 通过supervisorctl stop后，进程是否停止
[root@doris01 soft]# supervisorctl stop palo_be
palo_be: stopped
[root@doris01 soft]# supervisorctl stop PaloFe
PaloFe: stopped
[root@doris01 soft]# supervisorctl stop BrokerBootstrap
BrokerBootstrap: stopped
4.5 通过supervisorctl start可以开启进程
[root@doris01 soft]# supervisorctl start all
palo_be: started
PaloFe: started
BrokerBootstrap: started
[root@doris01 soft]# supervisorctl status
BrokerBootstrap                  RUNNING   pid 65421, uptime 0:00:21
PaloFe                           RUNNING   pid 498, uptime 0:00:21
palo_be                          RUNNING   pid 65422, uptime 0:00:21
结果显示启动控制成功。
4.6验证在fe，be，broker崩溃后supervisor能够自动重启进程
 输入命令ps xuf 查看进程间的父子关系
 ps xuf
 root     13617  0.0  0.0 243272 13056 ?        Ss   12月21   0:15 /usr/bin/python /bin/supervisord -c /etc/supervisord.conf
root     13618  0.0  0.0 113304  1572 ?        S    12月21   0:00  \_ sh /soft/apache_hdfs_broker/bin/start_broker.sh
root     13907  0.0  0.0 5259032 235864 ?      Sl   12月21   0:34  |   \_ /usr/local/java/bin/java -Xmx1024m -Dfile.encoding=UTF-8 org.apache.doris.broker.hdfs.BrokerBootstrap
root     13619  0.0  0.0 113176  1512 ?        S    12月21   0:00  \_ sh /soft/doris-be/bin/start_be.sh
root     14151  102  0.7 7065568 4194076 ?     Sl   12月21 836:43      \_ /soft/doris-be/lib/palo_be
执行命令 kill -9 14151  14151是supervisor启动be的pid
输入命令，看是否重启
ps xuf
root     13617  0.0  0.0 243528 13388 ?        Ss   12月21   0:15 /usr/bin/python /bin/supervisord -c /etc/supervisord.conf
root     13618  0.0  0.0 113304  1572 ?        S    12月21   0:00  \_ sh /soft/apache_hdfs_broker/bin/start_broker.sh
root     13907  0.0  0.0 5259032 235864 ?      Sl   12月21   0:34  |   \_ /usr/local/java/bin/java -Xmx1024m -Dfile.encoding=UTF-8 org.apache.doris.broker.hdfs.BrokerBootstrap
root     24209  0.5  0.0 113384  1624 ?        S    10:02   0:00  \_ sh /soft/doris-be/bin/start_be.sh
root     24568  113  0.5 5640592 2756220 ?     Sl   10:02   0:27      \_ /soft/doris-be/lib/palo_be
重启后的pid是24568
```

### **查看be配置**

```text
http://172.22.197.73:8040/varz
```

### **查看FE配置**

```text
http://172.22.197.72:8030/variable
```

------
