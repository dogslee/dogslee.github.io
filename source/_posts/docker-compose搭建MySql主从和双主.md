---
title: docker-compose搭建MySql主从和双主
date: 2021-08-23 16:03:58
index_img: /img/mysql/mysql.png
banner_img: /img/mysql/banner-1.jpg
categories:
- 存储
tags:
- mysql
- 高可用
- 数据同步
---

<p class="note note-primary">相信mysql的binlog都不陌生</br>binlog的主要作用就是进行数据同步，今天我们从数据同步的角度搭一下mysql的主从/双主。</p>

## base

以下基于mysql5.7

## 简单了解下binlog

- binlog是记录所有数据库表结构变更（例如CREATE、ALTER TABLE…）以及表数据修改（INSERT、UPDATE、DELETE…）的二进制日志。

- binlog日志包括两类文件：二进制日志索引文件（文件名后缀为.index）用于记录所有的二进制文件，二进制日志文件（文件名后缀为.00000*）记录数据库所有的DDL和DML(除了数据查询语句)语句事件。

- binlog有三种格式：statement基于sql语句复制、row基于行数据变更的复制、mixed混合前两种格式的复制。

## 搭建主从结构

![mysql主从架构](/img/mysql/mysql-m-s.png)

### 创建docker-compose

```yml
version: '3.8'
services:
  mysql-master:
    container_name: mysql-master 
    image: mysql:5.7.31
    restart: always
    ports:
      - 13306:3306 
    privileged: true
    volumes:
      # 这个只对mysql主从进行验证 没有持久化配置以及日志相关的映射
      - $PWD/master/conf/my.cnf:/etc/mysql/my.cnf
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - myweb
      
  mysql-slave:
    container_name: mysql-slave 
    image: mysql:5.7.31
    restart: always
    ports:
      - 23306:3306 
    privileged: true
    volumes:
      # 这个只对mysql主从进行验证 没有持久化配置以及日志相关的映射
      - $PWD/slave/conf/my.cnf:/etc/mysql/my.cnf
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - myweb    

networks:
  myweb:
    driver: bridge
```

### 创建配置文件

#### 配置文件目录结构

```bash
[root@xxx MySQLM-S]# tree
.
├── docker-compose.yaml
├── master
│   └── conf
│       └── my.cnf
└── slave
    └── conf
        └── my.cnf
```

#### master/conf/my.cnf 配置文件

```bash
[mysqld]
# [必须]服务器唯一ID，默认是1，一般取IP最后一段
server-id=1


# ###################################################
# 如果当前实例既做主库又做从库次选线必须开启
# log-slave-updates = true 

# 自增长ID
# 特殊说明 当该实例为双主的架构时要特殊配置 以避免自增id冲突的问题
# auto_increment_offset = 1
# auto_increment_increment = 2  
# ####################################################


# [必须]启用二进制日志
log-bin=mysql-bin 

# 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql

# 确保binlog日志写入后与硬盘同步
sync_binlog = 1

# 设置需要同步的数据库 binlog_do_db = 数据库名； 
# 如果是多个同步库，就以此格式另写几行即可。
# 如果不指明对某个具体库同步，表示同步所有库。除了binlog-ignore-db设置的忽略的库
# binlog_do_db = test #需要同步test数据库。

# 设置需要同步的数据库，主服务器上不限定数据库，在从服务器上限定replicate-do-db = 数据库名；
# 如果不指明同步哪些库，就去掉这行，表示所有库的同步（除了ignore忽略的库）。
# replicate-do-db = test；

# 跳过所有的错误，继续执行复制操作
slave-skip-errors = all  
```

#### slave/conf/my.cnf 配置文件

```bash
[mysqld]
# [必须]服务器唯一ID，默认是1，一般取IP最后一段  
server-id=2


# ###################################################
# 如果当前实例既做主库又做从库次选线必须开启
# log-slave-updates = true 

# 自增长ID
# 特殊说明 当该实例为双主的架构时要特殊配置 以避免自增id冲突的问题
# auto_increment_offset = 2
# auto_increment_increment = 2  
# ####################################################

# [必须]启用二进制日志
log-bin=mysql-bin 

# 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql

# 确保binlog日志写入后与硬盘同步
# sync_binlog = 1

# 设置需要同步的数据库 binlog_do_db = 数据库名； 
# 如果是多个同步库，就以此格式另写几行即可。
# 如果不指明对某个具体库同步，表示同步所有库。除了binlog-ignore-db设置的忽略的库
# binlog_do_db = test #需要同步test数据库。

# 设置需要同步的数据库，主服务器上不限定数据库，在从服务器上限定replicate-do-db = 数据库名；
# 如果不指明同步哪些库，就去掉这行，表示所有库的同步（除了ignore忽略的库）。
# replicate-do-db = test；

# 跳过所有的错误，继续执行复制操作
slave-skip-errors = all 
```

### 启动docker-compose并配置主从关系

#### 启动

```bash
docker-compose up -d 
```

#### 进入master配置同步账号和权限

```bash
docker-compose exec mysql-slave bash

mysql -uroot -p123455

# 查看配置的服务ID
mysql> show variables like '%server_id%';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| server_id      | 1     |
| server_id_bits | 32    |
+----------------+-------+

# 看master信息 File 和 Position 从服务上要用
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |      154 |              | mysql            |                   |
+------------------+----------+--------------+------------------+-------------------+

# 创建同步账户并开启权限
mysql> grant replication slave,replication client on *.* to 'slave'@'%' identified by "123456";
mysql> flush privileges;
```

#### 进入slave服务配置

```bash
docker-compose exec docker-slave bash

mysql -uroot -p123456

#查看server_id是否生效
mysql> show variables like '%server_id%';
+----------------+-------+
| Variable_name  | Value |
+----------------+-------+
| server_id      | 2     |
| server_id_bits | 32    |
+----------------+-------+

# 连接主mysql服务 master_log_file 和 master_log_pos的值要填写主master里查出来的值 注意这里使用的docker-compose 内部服务的端口和ip
mysql> change master to master_host='mysql-master',master_user='slave',master_password='123456',master_port=3306,master_log_file='mysql-bin.000004', master_log_pos=154,master_connect_retry=30;


# 开启slave

mysql> start slave;

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-master
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 778
               Relay_Log_File: af5556aff9be-relay-bin.000002
                Relay_Log_Pos: 944
        Relay_Master_Log_File: mysql-bin.000004
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: mysql
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 778
              Relay_Log_Space: 1158
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 466c4a60-03f4-11ec-a1a1-0242ac160002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 

```

<p class="note note-success"> 上面看到 Slave_IO_Running: Yes，Slave_SQL_Running: Yes 表示已经成功开启主从</p>

连接主mysql参数说明：

**master_port**：Master的端口号，指的是容器的端口号

**master_user**：用于数据同步的用户

**master_password**：用于同步的用户的密码

**master_log_file**：指定 Slave 从哪个日志文件开始复制数据，即上文中提到的 File 字段的值

**master_log_pos**：从哪个 Position 开始读，即上文中提到的 Position 字段的值

**master_connect_retry**：如果连接失败，重试的时间间隔，单位是秒，默认是60秒

## 搭建双主结构

通过上面主从结构我们可以我们可以大胆设想，要是两个数据库的实例都配置对方为master不就实现的双主么？事实却是如此：

双主结构只需要将双方的配置文件注释掉的地方取消注释掉分别在两台服务器上创同步账号和配置

### 创建docker-compose文件（双主）

这里只更改了docker-compose中服务的名称

```yaml
version: '3.8'
services:
  mysql-m1:
    container_name: mysql-m1
    image: mysql:5.7.31
    restart: always
    ports:
      - 13306:3306 
    privileged: true
    volumes:
      # 这个只对mysql主从进行验证 没有持久化配置以及日志相关的映射
      - $PWD/m1/conf/my.cnf:/etc/mysql/my.cnf
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - myweb
      
  mysql-m2:
    container_name: mysql-m2 
    image: mysql:5.7.31
    restart: always
    ports:
      - 23306:3306 
    privileged: true
    volumes:
      # 这个只对mysql主从进行验证 没有持久化配置以及日志相关的映射
      - $PWD/m2/conf/my.cnf:/etc/mysql/my.cnf
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    command: [
        '--character-set-server=utf8mb4',
        '--collation-server=utf8mb4_general_ci',
        '--max_connections=3000'
    ]
    networks:
      - myweb    

networks:
  myweb:
    driver: bridge
```

### 创建配置文件(双主)

这里只是将主从中的配置中注释掉的服务添加上

#### 配置文件目录结构（双主）

```bash
[root@xxx MySQLM-M]# tree
.
├── docker-compose.yaml
├── m1
│   └── conf
│       └── my.cnf
└── m2
    └── conf
        └── my.cnf
```

#### m1/conf/my.cnf

```bash
[mysqld]
# [必须]服务器唯一ID，默认是1，一般取IP最后一段
server-id=1


# ###################################################
# 如果当前实例既做主库又做从库次选线必须开启
log-slave-updates = true 

# 自增长ID
# 特殊说明 当该实例为双主的架构时要特殊配置 以避免自增id冲突的问题
auto_increment_offset = 1
auto_increment_increment = 2  
# ####################################################


# [必须]启用二进制日志
log-bin=mysql-bin 

# 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql

# 确保binlog日志写入后与硬盘同步
sync_binlog = 1

# 设置需要同步的数据库 binlog_do_db = 数据库名； 
# 如果是多个同步库，就以此格式另写几行即可。
# 如果不指明对某个具体库同步，表示同步所有库。除了binlog-ignore-db设置的忽略的库
# binlog_do_db = test #需要同步test数据库。

# 设置需要同步的数据库，主服务器上不限定数据库，在从服务器上限定replicate-do-db = 数据库名；
# 如果不指明同步哪些库，就去掉这行，表示所有库的同步（除了ignore忽略的库）。
# replicate-do-db = test；

# 跳过所有的错误，继续执行复制操作
slave-skip-errors = all  
```

#### m2/conf/my.cnf

```bash
[mysqld]
# [必须]服务器唯一ID，默认是1，一般取IP最后一段  
server-id=2


# ###################################################
# 如果当前实例既做主库又做从库次选线必须开启
log-slave-updates = true 

# 自增长ID
# 特殊说明 当该实例为双主的架构时要特殊配置 以避免自增id冲突的问题
auto_increment_offset = 2
auto_increment_increment = 2  
# ####################################################

# [必须]启用二进制日志
log-bin=mysql-bin 

# 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql

# 确保binlog日志写入后与硬盘同步
sync_binlog = 1

# 设置需要同步的数据库 binlog_do_db = 数据库名； 
# 如果是多个同步库，就以此格式另写几行即可。
# 如果不指明对某个具体库同步，表示同步所有库。除了binlog-ignore-db设置的忽略的库
# binlog_do_db = test #需要同步test数据库。

# 设置需要同步的数据库，主服务器上不限定数据库，在从服务器上限定replicate-do-db = 数据库名；
# 如果不指明同步哪些库，就去掉这行，表示所有库的同步（除了ignore忽略的库）。
# replicate-do-db = test；

# 跳过所有的错误，继续执行复制操作
slave-skip-errors = all 
```

### 启动docker-compose并配置m1和m2的双主

#### 启动（双主）

```bash
docker-compose up -d 
```

进入m1和m2下执行下列命令来获取各自的master status 和同步账号

m1:

```bash
docker-compose exec mysql-m1 bash

mysql -uroot -p123456

# 查看m1 File和Position
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      154 |              | mysql            |                   |
+------------------+----------+--------------+------------------+-------------------+

# 创建同步账号
mysql> grant replication slave,replication client on *.* to 'slave'@'%' identified by "123456";
mysql> flush privileges;

```

m2:

```bash
docker-compose exec mysql-m1 bash

mysql -uroot -p123456

# 查看m2 File和Position
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000005 |      154 |              | mysql            |                   |
+------------------+----------+--------------+------------------+-------------------+

# 创建同步账号
mysql> grant replication slave,replication client on *.* to 'slave'@'%' identified by "123456";
mysql> flush privileges;
```

m1:

```bash
mysql> change master to master_host='mysql-m2',master_user='slave',master_password='123456',master_port=3306,master_log_file='mysql-bin.000005', master_log_pos=154,master_connect_retry=30;

mysql> start slave;

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-m2
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 620
               Relay_Log_File: de7a84f1b7f1-relay-bin.000002
                Relay_Log_Pos: 786
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 620
              Relay_Log_Space: 1000
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 2
                  Master_UUID: de8af5ce-0410-11ec-ab6d-0242ac170003
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:

```

m2:

```bash
mysql> change master to master_host='mysql-m1',master_user='slave',master_password='123456',master_port=3306,master_log_file='mysql-bin.000005', master_log_pos=154,master_connect_retry=30;

mysql> start slave;

mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-m1
                  Master_User: slave
                  Master_Port: 3306
                Connect_Retry: 30
              Master_Log_File: mysql-bin.000005
          Read_Master_Log_Pos: 1086
               Relay_Log_File: 65322be4d8a9-relay-bin.000002
                Relay_Log_Pos: 786
        Relay_Master_Log_File: mysql-bin.000005
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1086
              Relay_Log_Space: 1000
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: de898a82-0410-11ec-9fee-0242ac170002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
```

<p class="noet note-success">至此双主节点设置完成</p>

## 总结

主从模式多用来进行读写分离

双主模式多用来进行高可用

更复杂的部署可能会部署多master和多slave并用keeplive保证统一访问的模式，这里没探究
