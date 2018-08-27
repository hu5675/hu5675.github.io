---
title: Mysql Group Replication 高可用组复制集群环境搭建
date: 2018-08-27 14:20
---

### 一、MySQL的高可用集群可分为“主从复制”和“组复制”

#### 传统主从复制

<a href="https://dev.mysql.com/doc/refman/5.7/en/group-replication-primary-secondary-replication.html" target="_blank">参照Mysql手册</a>

<font color="#dd0000"> 1、异步复制</font>

![](/images/mysql-mgr/1.png)

<font color="#dd0000"> 2、半同步复制</font>

![](/images/mysql-mgr/1.png)


#### 组复制（5.7.17新增功能）
	 
<a href="https://dev.mysql.com/doc/refman/5.7/en/group-replication-summary.html" target="_blank">参照Mysql手册</a>

![](/images/mysql-mgr/3.png)


### 二、Mysql Server服务节点环境说明

	mgr_n1		192.172.0.11
	mgr_n2		192.172.0.21
	mgr_n3		192.172.0.31

### 三、启动三个Mysql Server节点

	docker run -it --net=mha_cluster --ip=192.172.0.11 --name mgr_n1 ubuntu:mysql 

	docker run -it --net=mha_cluster --ip=192.172.0.21 --name mgr_n2 ubuntu:mysql 

	docker run -it --net=mha_cluster --ip=192.172.0.31 --name mgr_n3 ubuntu:mysql 

### 四、单主——组复制集群搭建

#### 1、配置 mgr_n1 为主节点

<font color="#dd0000">a）编辑mysql配置文件，并启动mysql服务</font>
	
	root@d74d96e3c7f5:/# vim /etc/mysql/mysql.conf.d/mysqld.cnf 

	[mysqld]
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	datadir         = /var/lib/mysql
	log-error       = /var/log/mysql/error.log
	symbolic-links=0

	server_id=31
	gtid_mode=ON
	enforce_gtid_consistency=ON
	master_info_repository=TABLE
	relay_log_info_repository=TABLE
	binlog_checksum=NONE
	log_slave_updates=ON
	log_bin=binlog
	binlog_format=ROW

	transaction_write_set_extraction=XXHASH64
	loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
	loose-group_replication_start_on_boot=off
	loose-group_replication_local_address= "192.172.0.11:33061"
	loose-group_replication_group_seeds= "192.172.0.11:33061,192.172.0.21:33061,192.172.0.31:33061"
	loose-group_replication_bootstrap_group=off

	report_host=192.172.0.11

	root@d74d96e3c7f5:/# service mysql start
	/etc/init.d/mysql: line 63: /lib/apparmor/profile-load: No such file or directory
	No directory, logging in with HOME=/
	..
	 * MySQL Community Server 5.7.23 is started
	root@d74d96e3c7f5:/# 

<font color="#dd0000">b）登录mysql实例，配置组实例</font>
	
	#关闭日志记录
	mysql> SET SQL_LOG_BIN=0;
	Query OK, 0 rows affected (0.00 sec)

	#创建复制用户
	mysql> CREATE USER rpl_user@'%' IDENTIFIED BY 'rpl_user';
	Query OK, 0 rows affected (0.00 sec)

	#赋予复制用户权限
	mysql> GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
	Query OK, 0 rows affected (0.00 sec)

	#开启日志记录
	mysql> SET SQL_LOG_BIN=1;
	Query OK, 0 rows affected (0.00 sec)

	#设置master
	mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_user' FOR CHANNEL 'group_replication_recovery';
	Query OK, 0 rows affected, 2 warnings (0.40 sec)

	#安装组复制插件
	mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
	Query OK, 0 rows affected (0.02 sec)

	#设置ip白名单
	mysql> set global group_replication_ip_whitelist = '192.172.0.11,192.172.0.21,192.172.0.31';
	Query OK, 0 rows affected (0.00 sec)

	#启动组复制
	mysql>  set global group_replication_bootstrap_group=ON;
	Query OK, 0 rows affected (0.00 sec)

	mysql> start group_replication;
	Query OK, 0 rows affected (2.16 sec)

	mysql>  set global group_replication_bootstrap_group=OFF;
	Query OK, 0 rows affected (0.00 sec)

<font color="#dd0000">c）查看组复制成员</font>
	
	mysql> SELECT * FROM performance_schema.replication_group_members;
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | d01adbdc-a9ca-11e8-bdbc-0242c0ac000b | 192.172.0.11 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	1 row in set (0.00 sec)

#### 2、配置 mgr_n2 为从节点

<font color="#dd0000">a）编辑mysql配置文件，并启动mysql服务</font>

	root@b87b75c5dd02:/# vim /etc/mysql/mysql.conf.d/mysqld.cnf 

	[mysqld]
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	datadir         = /var/lib/mysql
	log-error       = /var/log/mysql/error.log
	symbolic-links=0

	server_id=21
	gtid_mode=ON
	enforce_gtid_consistency=ON
	master_info_repository=TABLE
	relay_log_info_repository=TABLE
	binlog_checksum=NONE
	log_slave_updates=ON
	log_bin=binlog
	binlog_format=ROW

	transaction_write_set_extraction=XXHASH64
	loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
	loose-group_replication_start_on_boot=off
	loose-group_replication_local_address= "192.172.0.21:33061"
	loose-group_replication_group_seeds= "192.172.0.11:33061,192.172.0.21:33061,192.172.0.31:33061"
	loose-group_replication_bootstrap_group=off

	report_host=192.172.0.21

	root@b87b75c5dd02:/# service mysql start
	/etc/init.d/mysql: line 63: /lib/apparmor/profile-load: No such file or directory
	No directory, logging in with HOME=/
	..
	 * MySQL Community Server 5.7.23 is started
	root@b87b75c5dd02:/# 

<font color="#dd0000">b）登录mysql实例，将此实例节点加入到组复制集群中</font>

	#关闭日志记录
	mysql> SET SQL_LOG_BIN=0;
	Query OK, 0 rows affected (0.00 sec)

	#创建复制用户
	mysql> CREATE USER rpl_user@'%' IDENTIFIED BY 'rpl_user';
	Query OK, 0 rows affected (0.00 sec)

	#赋予复制用户权限
	mysql> GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
	Query OK, 0 rows affected (0.00 sec)

	#开启日志记录
	mysql> SET SQL_LOG_BIN=1;
	Query OK, 0 rows affected (0.00 sec)

	#设置master
	mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_user' FOR CHANNEL 'group_replication_recovery';
	Query OK, 0 rows affected, 2 warnings (0.40 sec)

	#安装组复制插件
	mysql> INSTALL PLUGIN group_replication SONAME 'group_replication.so';
	Query OK, 0 rows affected (0.02 sec)

	#设置ip白名单
	mysql> set global group_replication_ip_whitelist = '192.172.0.11,192.172.0.21,192.172.0.31';
	Query OK, 0 rows affected (0.00 sec)

	#开启 group_replication_allow_local_disjoint_gtids_join  这一步和节点1 配置不一样
	mysql> set global group_replication_allow_local_disjoint_gtids_join=ON;
	Query OK, 0 rows affected, 1 warning (0.00 sec)

	#启动组复制
	mysql> start group_replication;
	Query OK, 0 rows affected, 1 warning (5.86 sec)

<font color="#dd0000">c）查看组复制成员</font>

	mysql> SELECT * FROM performance_schema.replication_group_members;
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | 68535416-a9cc-11e8-954d-0242c0ac0015 | 192.172.0.21 |        3306 | ONLINE       |
	| group_replication_applier | d01adbdc-a9ca-11e8-bdbc-0242c0ac000b | 192.172.0.11 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	2 rows in set (0.01 sec)

#### 3、配置 mgr_n3 为从节点，和 mgr_n2 操作步骤一样，只需要把 mysqld.cnf 配置文件中的 server_id、group_replication_local_address、report_host 三项改为 mgr_n3 对应的 ip 即可
	略

<font color="#dd0000">查看组复制成员</font>
	
	mysql> SELECT * FROM performance_schema.replication_group_members;
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | 68535416-a9cc-11e8-954d-0242c0ac0015 | 192.172.0.21 |        3306 | ONLINE       |
	| group_replication_applier | d01adbdc-a9ca-11e8-bdbc-0242c0ac000b | 192.172.0.11 |        3306 | ONLINE       |
	| group_replication_applier | ddd482b7-a9cd-11e8-8d06-0242c0ac001f | 192.172.0.31 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	3 rows in set (0.01 sec)

#### 4、测试组复制——数据同步，在 mgr_n1 上 进行一些操作，检测是否同步到 mgr_n2、mgr_n3 之中
	
<font color="#dd0000"> mgr_n1 节点上</font>

	mysql> CREATE DATABASE test;
	Query OK, 1 row affected (0.03 sec)

	mysql> USE test;
	Database changed
	mysql> CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL);
	Query OK, 0 rows affected (0.22 sec)

	mysql> INSERT INTO t1 VALUES (1, 'Luis');
	Query OK, 1 row affected (0.03 sec)

	mysql> SELECT * FROM t1;
	+----+------+
	| c1 | c2   |
	+----+------+
	|  1 | Luis |
	+----+------+
	1 row in set (0.00 sec)

	mysql> 


<font color="#dd0000"> mgr_n2 节点上 </font>

	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| sys                |
	| test               |
	+--------------------+
	5 rows in set (0.00 sec)

	mysql> select * from test.t1;  
	+----+------+
	| c1 | c2   |
	+----+------+
	|  1 | Luis |
	+----+------+
	1 row in set (0.00 sec)

	mysql> show global variables like '%read_only%';
	+-----------------------+-------+
	| Variable_name         | Value |
	+-----------------------+-------+
	| innodb_read_only      | OFF   |
	| read_only             | ON    |  
	| super_read_only       | ON    |      # 只读模式
	| transaction_read_only | OFF   |
	| tx_read_only          | OFF   |
	+-----------------------+-------+
	5 rows in set (0.00 sec)


	mysql> 

<font color="#dd0000"> mgr_n3 节点上 </font>

	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| sys                |
	| test               |
	+--------------------+
	5 rows in set (0.00 sec)

	mysql> 
	mysql> select * from test.t1;
	+----+------+
	| c1 | c2   |
	+----+------+
	|  1 | Luis |
	+----+------+
	1 row in set (0.00 sec)

	mysql> show global variables like '%read_only%';
	+-----------------------+-------+
	| Variable_name         | Value |
	+-----------------------+-------+
	| innodb_read_only      | OFF   |
	| read_only             | ON    |
	| super_read_only       | ON    |       #只读模式
	| transaction_read_only | OFF   |
	| tx_read_only          | OFF   |
	+-----------------------+-------+
	5 rows in set (0.01 sec)

#### 4、测试组复制——主从切换，模拟 mgr_n1 宕机，检测《主》是否切换。

<font color="#dd0000"> mgr_n1 节点上</font>

	root@d74d96e3c7f5:/# service mysql stop
	.............
	 * MySQL Community Server 5.7.23 is stopped


<font color="#dd0000"> mgr_n2 或 mgr_n3 节点上</font>

	#查看组复制实例成员
	mysql> SELECT * FROM performance_schema.replication_group_members;   
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | 68535416-a9cc-11e8-954d-0242c0ac0015 | 192.172.0.21 |        3306 | ONLINE       |
	| group_replication_applier | ddd482b7-a9cd-11e8-8d06-0242c0ac001f | 192.172.0.31 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	2 rows in set (0.00 sec)

	#显示组复制中的《主》
	mysql> SHOW STATUS LIKE 'group_replication_primary_member';
	+----------------------------------+--------------------------------------+
	| Variable_name                    | Value                                |
	+----------------------------------+--------------------------------------+
	| group_replication_primary_member | 68535416-a9cc-11e8-954d-0242c0ac0015 |
	+----------------------------------+--------------------------------------+
	1 row in set (0.02 sec)

##### 可见主节点宕机之后，组复制集群会从从节点中选举一个从节点作为《主节点》继续提供服务
##### 停止3个节点的mysql服务，继续配置多主。


### 四、多主——组复制集群搭建， 在《单主》集群的三个节点中，重新配置已搭建《多主》集群。 

#### 1、配置 mgr_n1 节点
<font color="#dd0000"> 1、编辑配置文件，并重启mysql实例 </font>

	root@d74d96e3c7f5:/# vim /etc/mysql/mysql.conf.d/mysqld.cnf 

	[mysqld]
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	datadir         = /var/lib/mysql
	log-error       = /var/log/mysql/error.log
	symbolic-links=0

	server_id=11
	gtid_mode=ON
	enforce_gtid_consistency=ON
	master_info_repository=TABLE
	relay_log_info_repository=TABLE
	binlog_checksum=NONE
	log_slave_updates=ON
	log_bin=binlog
	binlog_format=ROW

	transaction_write_set_extraction=XXHASH64
	loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
	loose-group_replication_start_on_boot=off
	loose-group_replication_local_address= "192.172.0.11:33061"
	loose-group_replication_group_seeds= "192.172.0.11:33061,192.172.0.21:33061,192.172.0.31:33061"
	loose-group_replication_bootstrap_group=off

	#下面2行和单主不同
	loose-group_replication_single_primary_mode = off    
	loose-group_replication_enforce_update_everywhere_checks = on

	report_host=192.172.0.11

	#启动mysql
	root@d74d96e3c7f5:/# service mysql start 
	 * Stopping MySQL Community Server 5.7.23
	..............
	 * MySQL Community Server 5.7.23 is stopped
	 * Re-starting MySQL Community Server 5.7.23
	/etc/init.d/mysql: line 63: /lib/apparmor/profile-load: No such file or directory
	No directory, logging in with HOME=/
	 * MySQL Community Server 5.7.23 is started

<font color="#dd0000"> 2、登录mysql实例，配置组实例成员 </font>

	# 重置之前单主的 master bin_log
	mysql> reset master;
	Query OK, 0 rows affected (0.06 sec)

	mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_user' FOR CHANNEL 'group_replication_recovery';
	Query OK, 0 rows affected, 2 warnings (0.14 sec)

	mysql> set global group_replication_ip_whitelist = '192.172.0.11,192.172.0.21,192.172.0.31';
	Query OK, 0 rows affected (0.00 sec)

	# 在其他节点未启动时 需要先打开次选项，然后再启动 组复制
	mysql>  SET GLOBAL group_replication_bootstrap_group = ON;   
	Query OK, 0 rows affected (0.00 sec)

	mysql>  start group_replication;
	Query OK, 0 rows affected (2.06 sec)

	mysql>  SET GLOBAL group_replication_bootstrap_group = OFF;
	Query OK, 0 rows affected (0.00 sec)

<font color="#dd0000"> 3、查看组实成员</font>

	mysql> SELECT * FROM performance_schema.replication_group_members;
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | d01adbdc-a9ca-11e8-bdbc-0242c0ac000b | 192.172.0.11 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	1 row in set (0.00 sec)


#### 2、配置 mgr_n2 节点
<font color="#dd0000"> 1、编辑配置文件，并重启mysql实例 </font>

	root@b87b75c5dd02:/# vim /etc/mysql/mysql.conf.d/mysqld.cnf 

	[mysqld]
	pid-file        = /var/run/mysqld/mysqld.pid
	socket          = /var/run/mysqld/mysqld.sock
	datadir         = /var/lib/mysql
	log-error       = /var/log/mysql/error.log
	symbolic-links=0

	server_id=21
	gtid_mode=ON
	enforce_gtid_consistency=ON
	master_info_repository=TABLE
	relay_log_info_repository=TABLE
	binlog_checksum=NONE
	log_slave_updates=ON
	log_bin=binlog
	binlog_format=ROW

	transaction_write_set_extraction=XXHASH64
	loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
	loose-group_replication_start_on_boot=off
	loose-group_replication_local_address= "192.172.0.21:33061"
	loose-group_replication_group_seeds= "192.172.0.11:33061,192.172.0.21:33061,192.172.0.31:33061"
	loose-group_replication_bootstrap_group=off

	# 开启多主模式
	loose-group_replication_single_primary_mode = off
	loose-group_replication_enforce_update_everywhere_checks = on

	report_host=192.172.0.21

	root@b87b75c5dd02:/# service mysql start 
	/etc/init.d/mysql: line 63: /lib/apparmor/profile-load: No such file or directory
	No directory, logging in with HOME=/
	..
	 * MySQL Community Server 5.7.23 is started


<font color="#dd0000"> 2、登录mysql实例，配置组实例成员 </font>

	mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_user' FOR CHANNEL 'group_replication_recovery';
	Query OK, 0 rows affected, 2 warnings (0.04 sec)

	mysql> set global group_replication_ip_whitelist = '192.172.0.11,192.172.0.21,192.172.0.31';
	Query OK, 0 rows affected (0.00 sec)

	mysql> reset master;
	Query OK, 0 rows affected (0.06 sec)

	# 这里直接启动了 组复制，因为已经有节点1 启动了 所以不需要 开启 group_replication_bootstrap_group
	mysql> start group_replication;
	Query OK, 0 rows affected (5.05 sec)

<font color="#dd0000"> 3、查看组实成员</font>

	mysql> SELECT * FROM performance_schema.replication_group_members;
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | 68535416-a9cc-11e8-954d-0242c0ac0015 | 192.172.0.21 |        3306 | ONLINE       |
	| group_replication_applier | d01adbdc-a9ca-11e8-bdbc-0242c0ac000b | 192.172.0.11 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	2 rows in set (0.00 sec)

#### 3、配置 mgr_n3 节点 ，配置过程 和 mrg_n2 一样

<font color="#dd0000"> 1、编辑配置文件，并重启mysql实例 </font>
	
	略

<font color="#dd0000"> 2、登录mysql实例，配置组实例成员 </font>

	略

<font color="#dd0000"> 3、查看组实成员</font>

	mysql> SELECT * FROM performance_schema.replication_group_members;
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST  | MEMBER_PORT | MEMBER_STATE |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	| group_replication_applier | 68535416-a9cc-11e8-954d-0242c0ac0015 | 192.172.0.21 |        3306 | ONLINE       |
	| group_replication_applier | d01adbdc-a9ca-11e8-bdbc-0242c0ac000b | 192.172.0.11 |        3306 | ONLINE       |
	| group_replication_applier | ddd482b7-a9cd-11e8-8d06-0242c0ac001f | 192.172.0.31 |        3306 | ONLINE       |
	+---------------------------+--------------------------------------+--------------+-------------+--------------+
	3 rows in set (0.00 sec)

#### 4、测试《多主》组复制——数据同步，在 mgr_n1 上 进行一些操作，检测是否同步到 mgr_n2、mgr_n3 之中
<font color="#dd0000"> mgr_n1 上 </font>

	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| sys                |
	| test               |
	+--------------------+
	5 rows in set (0.01 sec)

	mysql> CREATE DATABASE test2;
	Query OK, 1 row affected (0.01 sec)

	mysql> USE test2;
	Database changed
	mysql> CREATE TABLE t2 (c1 INT PRIMARY KEY, c2 TEXT NOT NULL); 
	Query OK, 0 rows affected (0.10 sec)

	mysql> INSERT INTO t2 VALUES (1, 'lalalalala'); 
	Query OK, 1 row affected (0.03 sec)

	mysql> SELECT * FROM t2;
	+----+------------+
	| c1 | c2         |
	+----+------------+
	|  1 | lalalalala |
	+----+------------+
	1 row in set (0.00 sec)

<font color="#dd0000"> mgr_n2、mgr_n3 上  </font>
	
	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| sys                |
	| test               |
	| test2              |
	+--------------------+
	6 rows in set (0.00 sec)

	mysql> SELECT * FROM test2.t2;
	+----+------------+
	| c1 | c2         |
	+----+------------+
	|  1 | lalalalala |
	+----+------------+
	1 row in set (0.00 sec)

	mysql> 

#### 4、测试《多主》组复制—— 模拟 mgr_n1 宕机，检测 mgr_n2、mgr_n3 是否可以继续读写，之后再启动 mgr_n1 ，检测 mgr_n1 是否同步了 mgr_n2、mgr_n3 上的操作
<font color="#dd0000"> mgr_n1 上 </font>

	root@d74d96e3c7f5:/# service mysql stop 
	.............
	 * MySQL Community Server 5.7.23 is stopped

<font color="#dd0000"> mgr_n2 上 </font>
	
	mysql> USE test2;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A

	Database changed
	mysql> INSERT INTO t2 VALUES (2, 'mgr_n2'); 
	Query OK, 1 row affected (0.03 sec)

	mysql> 

<font color="#dd0000"> mgr_n3 上 </font>

	mysql> USE test2;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A

	Database changed
	mysql> select * from test2.t2;
	+----+------------+
	| c1 | c2         |
	+----+------------+
	|  1 | lalalalala |
	|  2 | mgr_n2     |
	+----+------------+
	2 rows in set (0.00 sec)

	mysql> INSERT INTO t2 VALUES (3, 'mgr_n3'); 
	Query OK, 1 row affected (0.03 sec)

	mysql> select * from test2.t2;
	+----+------------+
	| c1 | c2         |
	+----+------------+
	|  1 | lalalalala |
	|  2 | mgr_n2     |
	|  3 | mgr_n3     |
	+----+------------+
	3 rows in set (0.00 sec)

	mysql> 

<font color="#dd0000"> mgr_n2 上 </font>

	mysql> select * from t2;
	+----+------------+
	| c1 | c2         |
	+----+------------+
	|  1 | lalalalala |
	|  2 | mgr_n2     |
	|  3 | mgr_n3     |
	+----+------------+
	3 rows in set (0.00 sec)

	mysql> 

##### 可见 mgr_n1 宕机之后 在 mgr_n2 的操作可以同步到 mgr_n3中，在 mgr_n3中的操作可以同步到 mgr_n2 中。

<font color="#dd0000"> 重新将 mgr_n1 加入组复制 上 </font>

	root@d74d96e3c7f5:/# service mysql start
	/etc/init.d/mysql: line 63: /lib/apparmor/profile-load: No such file or directory
	No directory, logging in with HOME=/
	..
	 * MySQL Community Server 5.7.23 is started


	mysql> CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='rpl_user' FOR CHANNEL 'group_replication_recovery';
	Query OK, 0 rows affected, 2 warnings (0.06 sec)

	mysql>  set global group_replication_ip_whitelist = '192.172.0.11,192.172.0.21,192.172.0.31';
	Query OK, 0 rows affected (0.00 sec)

	mysql> start group_replication;
	Query OK, 0 rows affected (3.43 sec)


	mysql> use test2;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A

	Database changed
	mysql> select * from t2;
	+----+------------+
	| c1 | c2         |
	+----+------------+
	|  1 | lalalalala |
	|  2 | mgr_n2     |      # 宕机期间 mgr_n2 上插入操作
	|  3 | mgr_n3     |      # 宕机期间 mgr_n3 上插入操作
	+----+------------+
	3 rows in set (0.00 sec)

	mysql> 

##### 可见 mgr_n1 重新加入组复制之后， 在 mgr_n1 宕机期间 其他组成员的操作记录 都会同步到 mgr_n1 中


#### 组复制之《单主》、《多主》到此搭建完成







