---
title: MySQL Replication 集群
---

### 一、介绍
	为了保证mysql server的高可用，避免单点故障而造成整个系统不可用，Mysql提供了 Replication主从备份机制来解决单点故障。
	Mysql Replication提供多个服务器节点共同对外提供服务而组成的集群，每个节点都保存有一份完整的数据。
	当集群中的某一个节点出现故障不可用时，集群中的其他节点还可以继续对外提供服务，通过这种方式来保存Mysql Server的高可用。
	Mysql 的 Replication 是一个异步的复制过程，从一个 Mysql instace(我们称之为 Master)复制到另一个Mysql instance(我们称之 Slave)。
	在 Master 与 Slave 之间的 实现整个复制过程主要由三个线程来完成， 其中两个线程Sql(线程和IO线程)在Slave 端， 另外一个线程(IO 线程)在 Master 端。

### 二、主从集群架构

#### A:主从复制架构——（Master-Slaves)
![](/images/mysql_replication_docker/1.png) 
#### B:双主复制架构——（Dual Master)
![](/images/mysql_replication_docker/2.png) 
#### C:级联复制架构——（Master-Slaves-Slaves)
![](/images/mysql_replication_docker/3.png) 
#### D:双主和级联复制结合架构——（Master-Master-Slaves)
![](/images/mysql_replication_docker/4.png)

### 三、模拟搭建《D》集群架构 

![](/images/mysql_replication_docker/8.png)

#### 1）环境准备: docker mysql_5.5 镜像 （mysql镜像中已经安装了mysql数据库，并且设置了root用户的密码、所有权限以及远程访问）
![](/images/mysql_replication_docker/5.png)

#### 2) 根据mysql镜像启动3个容器实例
	docker run --name master_1 -it -p 13306:3306 ubuntu:mysql_5.5
	docker run --name master_2 -it -p 13307:3306 ubuntu:mysql_5.5     #新窗口执行
	docker run --name slave_1_1 -it -p 13308:3306 ubuntu:mysql_5.5	 #新窗口执行
![](/images/mysql_replication_docker/6.png)

#### 3）配置master_1   内部ip：172.17.0.16
	docker exec -it 96eee3e421dc /bin/bash   #进入master容器
	ifconfig			#ip查看
	echo '' > /etc/my.cnf 		#清空mysql配置文件my.cnf
	vim /etc/my.cnf 		#重新编辑my.cnf  输入下面内容并保存退出

	#############

	[client]
	port            = 3306
	socket          = /tmp/mysql.sock

	[mysqld]
	port            = 3306
	socket          = /tmp/mysql.sock
	skip-external-locking
	key_buffer_size = 16M
	max_allowed_packet = 1M
	table_open_cache = 64
	sort_buffer_size = 512K
	net_buffer_length = 8K
	read_buffer_size = 256K
	read_rnd_buffer_size = 512K
	myisam_sort_buffer_size = 8M

	log-bin=mysql-bin					#开启binlog日志 用于主从同步
	binlog-do-db=sakila					#同步数据库 sakila
	binlog_format=mixed

	server-id       = 1					#服务id，必须唯一

	[mysqldump]
	quick
	max_allowed_packet = 16M

	[mysql]
	no-auto-rehash

	[myisamchk]
	key_buffer_size = 20M
	sort_buffer_size = 20M
	read_buffer = 2M
	write_buffer = 2M

	[mysqlhotcopy]
	interactive-timeout

	#############

	cd /usr/local/mysql 			#进入mysql安装目录
	./bin/mysqld_safe &				#启动mysql

##### 使用客户端工具Navicat 测试master_1启动成功、并准备需要同步的数据库及数据
![](/images/mysql_replication_docker/9.png)
执行数据库schema文件和数据文件：[结构文件sakila-schema](/images/mysql_replication_docker/sakila-schema.sql)、[数据文件sakila-data](/images/mysql_replication_docker/sakila-data.sql)
执行之后最终结果
![](/images/mysql_replication_docker/10.png)

##### 创建一个用户用于master_2从master_1中同步数据
	用户名: repl_1_2
	密码: repl_1_2

	./bin/mysql -uroot -p 			#连接mysql 服务
	CREATE USER 'repl_1_2'@'172.17.0.17' IDENTIFIED BY 'repl_1_2';   #172.17.0.17 是master_2的ip
	GRANT REPLICATION SLAVE ON *.* TO 'repl_1_2'@'172.17.0.17' IDENTIFIED BY 'repl_1_2';     #分配权限
	flush privileges;   #刷新权限

##### 创建一个用户用于 slave_1_1 从master_1中同步数据
	用户名: repl_1_11
	密码: repl_1_11

	./bin/mysql -uroot -p 			#连接mysql 服务
	CREATE USER 'repl_1_11'@'172.17.0.18' IDENTIFIED BY 'repl_1_11';   #172.17.0.18 是 slave_1_1 的ip
	GRANT REPLICATION SLAVE ON *.* TO 'repl_1_11'@'172.17.0.18' IDENTIFIED BY 'repl_1_11';     #分配权限
	flush privileges;   #刷新权限

#### 4）配置master_2		内部ip：172.17.0.17
	docker exec -it 12031981101f /bin/bash   #进入master容器
	echo '' > /etc/my.cnf 		#清空mysql配置文件my.cnf
	vim /etc/my.cnf 		#重新编辑my.cnf  输入下面内容并保存退出

	#######

	[client]
	port            = 3306
	socket          = /tmp/mysql.sock

	[mysqld]
	port            = 3306
	socket          = /tmp/mysql.sock
	skip-external-locking
	key_buffer_size = 16M
	max_allowed_packet = 1M
	table_open_cache = 64
	sort_buffer_size = 512K
	net_buffer_length = 8K
	read_buffer_size = 256K
	read_rnd_buffer_size = 512K
	myisam_sort_buffer_size = 8M

	log-bin=mysql-bin
	binlog-do-db=sakila
	binlog_format=mixed

	server-id       = 2

	[mysqldump]
	quick
	max_allowed_packet = 16M

	[mysql]

	[myisamchk]
	key_buffer_size = 20M
	sort_buffer_size = 20M
	read_buffer = 2M
	write_buffer = 2M

	[mysqlhotcopy]
	interactive-timeout

	#######

	cd /usr/local/mysql 			#进入mysql安装目录
	./bin/mysqld_safe &				#启动mysql

##### 使用客户端工具Navicat 测试master_2连接，现在没有sakila数据结构和数据
![](/images/mysql_replication_docker/11.png)

##### 创建一个用户用于master_1从master_2中同步数据
	用户名: repl_2_1
	密码: repl_2_1

	./bin/mysql -uroot -p 			#连接mysql 服务
	CREATE USER 'repl_2_1'@'172.17.0.16' IDENTIFIED BY 'repl_2_1';   #172.17.0.16 是 master_1 的ip
	GRANT REPLICATION SLAVE ON *.* TO 'repl_2_1'@'172.17.0.16' IDENTIFIED BY 'repl_2_1';     #分配权限
	flush privileges;   #刷新权限

##### 连接mysql服务并配置master_1作为master_2的主库
	./bin/mysql -uroot -p
	CHANGE MASTER TO MASTER_HOST='172.17.0.16', MASTER_USER='repl_1_2', MASTER_PASSWORD='repl_1_2';   # 配置master_1 为自己的主库
	start slave;    			#开启备份
	show slave status;      	#查看备份状态  Slave_IO_Running | Slave_SQL_Running   都为yes 则配置成功

##### 使用客户端工具Navicat 检查master_2中是否有sakila数据库和数据
![](/images/mysql_replication_docker/12.png)
可以看到master_2数据已经同步下来

##### 使用客户端工具Navicat 测试修改master_1的数据，检查是否同步更新到master_2中
![](/images/mysql_replication_docker/1.gif)

#### 5）配置master_2 作为 master_1的主库

##### 连接 master_1 mysql服务并配置master_2作为master_1的主库
	./bin/mysql -uroot -p 	 #连接master_1  
	CHANGE MASTER TO MASTER_HOST='172.17.0.17', MASTER_USER='repl_2_1', MASTER_PASSWORD='repl_2_1';   #配置主库
	start slave;   #开始同步
	show slave status;      	#查看备份状态  Slave_IO_Running | Slave_SQL_Running   都为yes 则配置成功

##### 使用客户端工具Navicat 测试修改 master_1 的数据，检查是否同步更新到 master_2 中、测试修改 master_2 的数据，检查是否同步更新到 master_1 中
![](/images/mysql_replication_docker/2.gif)

#### 到此 Dual Master (Mysql双主同步) 配置完成

#### 6）配置slave_1_1, 使其成为master_1的从库
	
##### 配置slave_1_1，内部ip：172.17.0.18
	docker exec -it f5e6eb4a50ee /bin/bash   #进入master容器
	echo '' > /etc/my.cnf 		#清空mysql配置文件my.cnf
	vim /etc/my.cnf 		#重新编辑my.cnf  输入下面内容并保存退出

	#######

	[client]
	port            = 3306
	socket          = /tmp/mysql.sock

	[mysqld]
	port            = 3306
	socket          = /tmp/mysql.sock
	skip-external-locking
	key_buffer_size = 16M
	max_allowed_packet = 1M
	table_open_cache = 64
	sort_buffer_size = 512K
	net_buffer_length = 8K
	read_buffer_size = 256K
	read_rnd_buffer_size = 512K
	myisam_sort_buffer_size = 8M


	server-id       = 11

	[mysqldump]
	quick
	max_allowed_packet = 16M

	[mysql]

	[myisamchk]
	key_buffer_size = 20M
	sort_buffer_size = 20M
	read_buffer = 2M
	write_buffer = 2M

	[mysqlhotcopy]
	interactive-timeout

	#######

	cd /usr/local/mysql 			#进入mysql安装目录
	./bin/mysqld_safe &				#启动mysql

##### 使用客户端工具Navicat 测试连接 slave_1_1 
![](/images/mysql_replication_docker/13.png)

##### 连接 slave_1_1 mysql服务并配置 master_1 作为 slave_1_1 的主库
	./bin/mysql -uroot -p 	 #连接 master_1  
	CHANGE MASTER TO MASTER_HOST='172.17.0.16', MASTER_USER='repl_1_11', MASTER_PASSWORD='repl_1_11';  #配置主库
	start slave;   #开始同步
	show slave status;      	#查看备份状态  Slave_IO_Running | Slave_SQL_Running   都为yes 则配置成功

##### 使用客户端工具Navicat 测试连接 slave_1_1 ,可以看到master_1中的数据同步到了slave_1_1
![](/images/mysql_replication_docker/14.png)

##### 返回来看master_1中的数据，发现有一行记录没有同步到 slave_1_1 中去
![](/images/mysql_replication_docker/15.png)

##### 回忆之前，ED_33333 数据的修改是在master_2上主动修改，然后同步master_2上修改的binlog到master_1的relay_log上，而master_1根据relay_log的修改记录来修改数据，这个修改数据的”动作“并没有在master_1上产生binlog，所以没有同步到slave_1_1中去。

##### 解决上述问题需要在master_1上的mysql服务器开启log_slave_updates配置，修改/etc/my.cnf，加上 log_slave_updates=1 并重启mysql

##### 使用客户端工具Navicat 测试修改 master_1 和 master_2 上的数据，看是否都同步到slave_1_1中
![](/images/mysql_replication_docker/3.gif)

#### 到此 Dual Master Slave 架构搭建完成




