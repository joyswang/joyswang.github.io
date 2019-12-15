---
layout: post
title: docker简单mysql主备
date: 2019-12-10 23:00:00
categories:
- mysql
- docker
tags:
- docker
- mysql
---

以下例子的mysql版本为5.7

docker官网下载docker for mac然后直接安装。下载地址自行百度\google

#### 拉取mysql5.7

```
docker pull mysql:5.7

```

#### 配置mysql的master、slave的my.cnf

##### 配置目录

* master
    * conf
        * my.cnf
    * logs
    * data
* slave
    * conf
        * my.cnf
    * logs
    * data

##### my.cnf

master的my.cnf

```
[mysqld]
#skip-grant-tables
character_set_server=utf8
server_id=1
log_bin=mysql-bin
```

slave的my.cnf

```
[mysqld]
#skip-grant-tables
character_set_server=utf8
server_id=2
log_bin=mysql-slave-bin
relay_log=edu-mysql-relay-bin
```

#### 启动mysql

```
//master目录下
docker run -p 3306:3306 --name mysql_master -v $PWD/conf/my.cnf:/etc/mysql/my.cnf -v $PWD/logs:/logs -v $PWD/data:/mysql_data -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7


//slave目录下
docker run -p 3307:3306 --name mysql_slave -v $PWD/conf/my.cnf:/etc/mysql/my.cnf -v $PWD/logs:/logs -v $PWD/data:/mysql_data -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

```

#### master配置

新建一个主备账户，并且记录当前master的status

```
grant replication slave on *.* to 'testSlave'@'%' identified by 'testSlave';
flush privileges;

show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      594 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
//记录File和Positoin

```

#### slave配置

获取master容器的ip,假如是：172.17.0.2

```
docker inspect mysql_master | grep IPAddress

```

配置slave

```

CHANGE MASTER TO
MASTER_HOST='172.17.0.2',
MASTER_USER='testSlave',
MASTER_PASSWORD='testSlave',
MASTER_LOG_FILE='mysql-bin.000003',
MASTER_LOG_POS=594;

start slave;

```
查看slave的status

```
show slave status\G;

```

如下显示，则表示配置成功

```
Slave_IO_State: Waiting for master to send event
.
.
.
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
.
.
.

```

#### 测试

进入master容器，新建数据库

```
docker exec -it mysql_master bash;
mysql -uroot -p123456;
create database canal_sync;

```

查看slave容器内的数据库，可以看到已经有了canal_sync数据库

