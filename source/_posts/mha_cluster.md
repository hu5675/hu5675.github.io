---
title: MHA架构模拟搭建
date: 2018-08-22 17:07
---

### 架构图
![运行图](/images/mha_cluster/1.png)

### MHA介绍
	MHA（Master High Availability）目前在MySQL高可用方面是一个相对成熟的解决方案，它由日本DeNA公司youshimaton（现就职于Facebook公司）开发，
	是一套优秀的作为MySQL高可用性环境下故障切换和主从提升的高可用软件。在MySQL故障切换过程中，MHA能做到在0~30秒之内自动完成数据库的故障切换操作，
	并且在进行故障切换的过程中，MHA能在最大程度上保证数据的一致性，以达到真正意义上的高可用。

### MHA提供什么功能
	1、监控主数据库服务器是否可用
	2、当主DB不可用时，从多个从服务器中选举新的主数据库服务器
	3、提供了主从切换和故障转移功能

### MHA主从切换过程
	1、尝试从出现故障的主数据库保存二进制日志
	2、从多个备选服务器中选举出新的备选主服务器
	3、在备选主服务器和其他从服务器之间进行同步差异二进制数据

### MHA配置步骤
	1、建立主从复制集群
	2、配置集群内所有主机的SSH免认证登录
	3、在数据库主机和监控主机上安装MHA-node软件包
	4、在监控主机上安装MHA-manager软件包，并配置


### 环境参数说明
	主: 				mha_master_01    192.172.0.11  
	从:			 	mha_slave_01     192.172.0.13
	从:			 	mha_slave_02     192.172.0.14    
	mha管理节点：		mha_manager		 192.172.0.20

### 搭建主从集群（具体可参照之前文章——MySQL Replication 集群）
#### 1）配置SQL启动SQL服务
###### <font color="#dd0000">启动3个节点</font>
	docker run -it --name mha_master_01 --net=mha_cluster --ip=192.172.0.11 ubuntu:mysql

	docker run -it --name mha_slave_01 --net=mha_cluster --ip=192.172.0.13 ubuntu:mysql
	docker run -it --name mha_slave_02 --net=mha_cluster --ip=192.172.0.14 ubuntu:mysql

###### <font color="#dd0000">mha_master_01 配置</font>
	vim /etc/mysql/mysql.conf.d/mysqld.cnf

	[mysqld]
	server_id=11
	gtid_mode=on
	enforce_gtid_consistency=on

	log_bin=master-binlog
	log-slave-updates=1
	binlog_format=row

	skip_slave_start=1

###### <font color="#dd0000">mha_slave_01 配置</font>
	vim /etc/mysql/mysql.conf.d/mysqld.cnf

	[mysqld]
	server_id=13
	gtid_mode=on
	enforce_gtid_consistency=on

	log-bin=slave-binlog
	log-slave-updates=1
	binlog_format=row

	skip_slave_start=1

###### <font color="#dd0000">mha_slave_02 配置 把server_id改成14</font>
	vim /etc/mysql/mysql.conf.d/mysqld.cnf

	[mysqld]
	server_id=14
	gtid_mode=on
	enforce_gtid_consistency=on

	log-bin=slave-binlog
	log-slave-updates=1
	binlog_format=row

	skip_slave_start=1

###### <font color="#dd0000">分别启动3个节点中SQL服务</font>
	service mysql start

#### 2）配置主从集群
###### <font color="#dd0000"> #在 mha_master_01 上以root用户登录，并分别创建复制用户</font>
	grant replication slave on *.* to 'replication'@'192.172.0.%' identified by 'replication';
	flush privileges;

###### <font color="#dd0000"> #在 mha_slave_01、mha_slave_02 上配置mha_master_01为主库</font>
	CHANGE MASTER TO MASTER_HOST='192.172.0.11',  MASTER_USER='replication', MASTER_PASSWORD='replication', MASTER_AUTO_POSITION=1
	start slave;
	show slave status \G;  
![运行图](/images/mha_cluster/2.png)

#### 3）主从测试
###### <font color="#dd0000"> #在 mha_master_01 上创建个数据库 test ， 看 mha_slave_01、mha_slave_02 是否同步创建了test库</font>
	mysql> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| sys                |
	+--------------------+
	4 rows in set (0.00 sec)

	mysql> create database test; 
	Query OK, 1 row affected (0.04 sec)

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

###### <font color="#dd0000"> # 在从节点 mha_slave_01、mha_slave_02 上查看数据库</font>

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

### 配置所有主机中SSH免认证登录
###### <font color="#dd0000">1、修改所有主机 sshd_config 配置文件 PermitRootLogin yes，并启动sshd服务</font>
	
	vim /etc/ssh/sshd_config 

	Port 22
	Protocol 2
	HostKey /etc/ssh/ssh_host_rsa_key
	HostKey /etc/ssh/ssh_host_dsa_key
	HostKey /etc/ssh/ssh_host_ecdsa_key
	HostKey /etc/ssh/ssh_host_ed25519_key
	UsePrivilegeSeparation yes

	KeyRegenerationInterval 3600
	ServerKeyBits 1024

	SyslogFacility AUTH
	LogLevel INFO

	LoginGraceTime 120
	PermitRootLogin yes
	StrictModes yes
	.
	.
	.

###### <font color="#dd0000">2、启动sshd</font>

	root@d3ca3e85de35:/# /etc/init.d/ssh start 
 	* Starting OpenBSD Secure Shell server sshd     

###### <font color="#dd0000">3、修改每台主机用户root的密码为111111</font>
 	passwd root

###### <font color="#dd0000">4、在mha_master_01上生成ssh-key</font>

	root@d3ca3e85de35:/# ssh-keygen 
	Generating public/private rsa key pair.

	Enter file in which to save the key (/root/.ssh/id_rsa): Enter passphrase (empty for no passphrase): 
	Enter same passphrase again: 
	Your identification has been saved in /root/.ssh/id_rsa.
	Your public key has been saved in /root/.ssh/id_rsa.pub.
	The key fingerprint is:
	SHA256:AIn1lcRzr6Xyd2bDtLBfNRPz11GHKZDFYx0E8QCB1GA root@d3ca3e85de35
	The key's randomart image is:
	+---[RSA 2048]----+
	|   oo. +E=+B==o+o|
	|  . .o o= + =o+ o|
	|      o  o o o.+ |
	|       .    o   *|
	|        S  +   o=|
	|        . o . . =|
	|         o   = ..|
	|          . o B. |
	|           . =.. |
	+----[SHA256]-----+

###### <font color="#dd0000">5、将mha_master_01的公钥同步到本机和其他主机</font>
	#1、同步本机
	root@d3ca3e85de35:/# ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.172.0.11
	/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
	The authenticity of host '192.172.0.11 (192.172.0.11)' can't be established.
	ECDSA key fingerprint is SHA256:4cluU7bAFjMtxNiRQVJYh2CrMF6o0IgdmVksPjmCSeM.
	Are you sure you want to continue connecting (yes/no)? yes
	/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
	/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
	root@192.172.0.11's password: 

	Number of key(s) added: 1

	Now try logging into the machine, with:   "ssh 'root@192.172.0.11'"
	and check to make sure that only the key(s) you wanted were added.

	#2、验证本机ssh免认证登录
	root@d3ca3e85de35:/# ssh 'root@192.172.0.11'
	Welcome to Ubuntu 16.04.3 LTS (GNU/Linux 3.10.0-693.17.1.el7.x86_64 x86_64)

	 * Documentation:  https://help.ubuntu.com
	 * Management:     https://landscape.canonical.com
	 * Support:        https://ubuntu.com/advantage
 	root@d3ca3e85de35:~# 

	#其他主机   重复上述步骤 1、2 ，将公钥同步到 mha_slave_01、mha_slave_02 、mha_manager 

###### <font color="#dd0000">6、在其他主机中重复4、5步骤，确保每台主机之间能相互的ssh免认证登录</font>
	略

### 所有主机中安装MHA-node软件包
	apt install libdbd-mysql-perl    //依赖包
	apt install make  				 //工具包
	wget https://downloads.mariadb.com/MHA/mha4mysql-node-0.56.tar.gz
	tar -zxvf mha4mysql-node-0.56.tar.gz 
	cd mha4mysql-node-0.56
	perl Makefile.PL 
	make && make install

### 监控主机中安装MHA-manger软件包
	apt install libconfig-tiny-perl				//依赖包
	apt install liblog-dispatch-perl			//依赖包
	apt install libparallel-forkmanager-perl    //依赖包

	wget https://downloads.mariadb.com/MHA/mha4mysql-manager-0.56.tar.gz
	tar -zxvf mha4mysql-manager-0.56.tar.gz 
	cd mha4mysql-manager-0.56
	perl Makefile.PL 
	make && make install

### 配置 MHA Manager
	# 创建目录和文件
	mkdir /home/mha            
	touch /home/mha/manager.log
	mkdir /etc/mha
	touch /etc/mha/mysql_mha.conf

	#编辑
	vim /etc/mha/mysql_mha.conf

	[server default]
	manager_workdir=/home/mha/
	manager_log=/home/mha/manager.log
	master_binlog_dir=/var/lib/mysql
	user=root
	password=root
	remote_workdir=/home/mha    #每台数据库主机中要存在此目录
	repl_user=replication
	repl_password=replication
	ssh_user=root

	[server1]
	hostname=192.172.0.11
	port=3306

	[server2]
	hostname=192.172.0.13
	port=3306
	candidate_master=1
	check_repl_delay=0

	[server3]
	hostname=192.172.0.14
	port=3306

### 使用 masterha_check_ssh 检测每个主机节点中的ssh是否正常
	root@f7e548d16865:/home/mha# masterha_check_ssh --conf=/etc/mha/mysql_mha.conf 
	Thu Aug 23 02:29:05 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
	Thu Aug 23 02:29:05 2018 - [info] Reading application default configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 02:29:05 2018 - [info] Reading server configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 02:29:05 2018 - [info] Starting SSH connection tests..
	Thu Aug 23 02:29:06 2018 - [debug] 
	Thu Aug 23 02:29:05 2018 - [debug]  Connecting via SSH from root@192.172.0.11(192.172.0.11:22) to root@192.172.0.13(192.172.0.13:22)..
	Thu Aug 23 02:29:05 2018 - [debug]   ok.
	Thu Aug 23 02:29:05 2018 - [debug]  Connecting via SSH from root@192.172.0.11(192.172.0.11:22) to root@192.172.0.14(192.172.0.14:22)..
	Thu Aug 23 02:29:06 2018 - [debug]   ok.
	Thu Aug 23 02:29:06 2018 - [debug] 
	Thu Aug 23 02:29:05 2018 - [debug]  Connecting via SSH from root@192.172.0.13(192.172.0.13:22) to root@192.172.0.11(192.172.0.11:22)..
	Thu Aug 23 02:29:06 2018 - [debug]   ok.
	Thu Aug 23 02:29:06 2018 - [debug]  Connecting via SSH from root@192.172.0.13(192.172.0.13:22) to root@192.172.0.14(192.172.0.14:22)..
	Thu Aug 23 02:29:06 2018 - [debug]   ok.
	Thu Aug 23 02:29:07 2018 - [debug] 
	Thu Aug 23 02:29:06 2018 - [debug]  Connecting via SSH from root@192.172.0.14(192.172.0.14:22) to root@192.172.0.11(192.172.0.11:22)..
	Thu Aug 23 02:29:06 2018 - [debug]   ok.
	Thu Aug 23 02:29:06 2018 - [debug]  Connecting via SSH from root@192.172.0.14(192.172.0.14:22) to root@192.172.0.13(192.172.0.13:22)..
	Thu Aug 23 02:29:06 2018 - [debug]   ok.
	Thu Aug 23 02:29:07 2018 - [info] All SSH connection tests passed successfully.

### 使用 masterha_check_repl 检测数据库主机中的主从复制是否正常
	
	 root@f7e548d16865:/home/mha# masterha_check_repl --conf=/etc/mha/mysql_mha.conf   
	Thu Aug 23 02:30:53 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
	Thu Aug 23 02:30:53 2018 - [info] Reading application default configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 02:30:53 2018 - [info] Reading server configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 02:30:53 2018 - [info] MHA::MasterMonitor version 0.56.
	Thu Aug 23 02:30:53 2018 - [error][/usr/local/share/perl/5.22.1/MHA/ServerManager.pm, ln255] Got MySQL error when connecting 192.172.0.13(192.172.0.13:3306) :1045:Access denied for user 'root'@'mha_manager.mha_cluster' (using password: YES), but this is not mysql crash. Check MySQL server settings.
	 at /usr/local/share/perl/5.22.1/MHA/ServerManager.pm line 251.
	Thu Aug 23 02:30:53 2018 - [error][/usr/local/share/perl/5.22.1/MHA/ServerManager.pm, ln255] Got MySQL error when connecting 192.172.0.11(192.172.0.11:3306) :1045:Access denied for user 'root'@'mha_manager.mha_cluster' (using password: YES), but this is not mysql crash. Check MySQL server settings.
	 at /usr/local/share/perl/5.22.1/MHA/ServerManager.pm line 251.
	Thu Aug 23 02:30:53 2018 - [error][/usr/local/share/perl/5.22.1/MHA/ServerManager.pm, ln255] Got MySQL error when connecting 192.172.0.14(192.172.0.14:3306) :1045:Access denied for user 'root'@'mha_manager.mha_cluster' (using password: YES), but this is not mysql crash. Check MySQL server settings.
	 at /usr/local/share/perl/5.22.1/MHA/ServerManager.pm line 251.
	Thu Aug 23 02:30:54 2018 - [error][/usr/local/share/perl/5.22.1/MHA/ServerManager.pm, ln263] Got fatal error, stopping operations
	Thu Aug 23 02:30:54 2018 - [error][/usr/local/share/perl/5.22.1/MHA/MasterMonitor.pm, ln401] Error happend on checking configurations.  at /usr/local/share/perl/5.22.1/MHA/MasterMonitor.pm line 306.
	Thu Aug 23 02:30:54 2018 - [error][/usr/local/share/perl/5.22.1/MHA/MasterMonitor.pm, ln500] Error happened on monitoring servers.
	Thu Aug 23 02:30:54 2018 - [info] Got exit code 1 (Not master dead).

	MySQL Replication Health is NOT OK!

	# 报错，解决方法在 主库中创建个用户 mha，赋相应权限(因为配置的主从，所有从库上也创建了相应的用户)，并修改配置文件
	# 1、创建用户并赋予权限
	mysql> grant all privileges on *.* to mha@'192.172.0.%' identified by 'mah';  
	mysql> flush privileges;

	# 2、修改MHA 配置文件 mysql_mha.conf

	vim /etc/mha/mysql_mha.conf

	[server default]
	manager_workdir=/home/mha/
	manager_log=/home/mha/manager.log
	master_binlog_dir=/var/lib/mysql
	user=mha 							#新创建的用户
	password=mha
	remote_workdir=/home/mha
	repl_user=replication
	repl_password=replication
	ssh_user=root

	[server1]
	hostname=192.172.0.11
	port=3306

	[server2]
	hostname=192.172.0.13
	port=3306
	candidate_master=1
	check_repl_delay=0

	[server3]
	hostname=192.172.0.14
	port=3306

	#重新执行  masterha_check_repl --conf=/etc/mha/mysql_mha.conf

	root@f7e548d16865:/home/mha# masterha_check_repl --conf=/etc/mha/mysql_mha.conf
	Thu Aug 23 03:03:58 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
	Thu Aug 23 03:03:58 2018 - [info] Reading application default configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 03:03:58 2018 - [info] Reading server configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 03:03:58 2018 - [info] MHA::MasterMonitor version 0.56.
	Thu Aug 23 03:03:59 2018 - [error][/usr/local/share/perl/5.22.1/MHA/MasterMonitor.pm, ln401] Error happend on checking configurations. Redundant argument in sprintf at /usr/local/share/perl/5.22.1/MHA/NodeUtil.pm line 184.
	Thu Aug 23 03:03:59 2018 - [error][/usr/local/share/perl/5.22.1/MHA/MasterMonitor.pm, ln500] Error happened on monitoring servers.
	Thu Aug 23 03:03:59 2018 - [info] Got exit code 1 (Not master dead).

	# 报错，解决方法: 修改 /usr/local/share/perl/5.22.1/MHA/NodeUtil.pm 文件,定位184行

	将以下内容
	sub parse_mysql_version($) {
		my $str = shift;
		my $result = sprintf( '%03d%03d%03d', $str =~ m/(\d+)/g );
		return $result;
	}
	sub parse_mysql_major_version($) {
	    my $str = shift;
	    my $result = sprintf( '%03d%03d', $str =~ m/(\d+)/g );
	    return $result;
	}

	修改为下面内容

	sub parse_mysql_version($) {
		my $str = shift;
		$str =~ /(\d+)\.(\d+)/;
		my $strmajor = "$1.$2.$3";
		my $result = sprintf( '%03d%03d%03d', $strmajor =~ m/(\d+)/g );
		return $result;
	}

	sub parse_mysql_major_version($) {
	    my $str = shift;
	    $str =~ /(\d+)\.(\d+)/;
	    my $strmajor = "$1.$2";
	    my $result = sprintf( '%03d%03d', $strmajor =~ m/(\d+)/g );
	    return $result;
	}

	#重新执行  masterha_check_repl --conf=/etc/mha/mysql_mha.conf

	root@f7e548d16865:/home/mha# masterha_check_repl --conf=/etc/mha/mysql_mha.conf
	Thu Aug 23 03:11:34 2018 - [warning] Global configuration file /etc/masterha_default.cnf not found. Skipping.
	Thu Aug 23 03:11:34 2018 - [info] Reading application default configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 03:11:34 2018 - [info] Reading server configurations from /etc/mha/mysql_mha.conf..
	Thu Aug 23 03:11:34 2018 - [info] MHA::MasterMonitor version 0.56.
	Thu Aug 23 03:11:35 2018 - [info] Dead Servers:
	Thu Aug 23 03:11:35 2018 - [info] Alive Servers:
	Thu Aug 23 03:11:35 2018 - [info]   192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:11:35 2018 - [info]   192.172.0.13(192.172.0.13:3306)
	Thu Aug 23 03:11:35 2018 - [info]   192.172.0.14(192.172.0.14:3306)
	Thu Aug 23 03:11:35 2018 - [info] Alive Slaves:
	Thu Aug 23 03:11:35 2018 - [info]   192.172.0.13(192.172.0.13:3306)  Version=5.7.23-0ubuntu0.16.04.1-log (oldest major version between slaves) log-bin:enabled
	Thu Aug 23 03:11:35 2018 - [info]     GTID ON
	Thu Aug 23 03:11:35 2018 - [info]     Replicating from 192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:11:35 2018 - [info]     Primary candidate for the new Master (candidate_master is set)
	Thu Aug 23 03:11:35 2018 - [info]   192.172.0.14(192.172.0.14:3306)  Version=5.7.23-0ubuntu0.16.04.1-log (oldest major version between slaves) log-bin:enabled
	Thu Aug 23 03:11:35 2018 - [info]     GTID ON
	Thu Aug 23 03:11:35 2018 - [info]     Replicating from 192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:11:35 2018 - [info] Current Alive Master: 192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:11:35 2018 - [info] Checking slave configurations..
	Thu Aug 23 03:11:35 2018 - [info]  read_only=1 is not set on slave 192.172.0.13(192.172.0.13:3306).
	Thu Aug 23 03:11:35 2018 - [warning]  relay_log_purge=0 is not set on slave 192.172.0.13(192.172.0.13:3306).
	Thu Aug 23 03:11:35 2018 - [info]  read_only=1 is not set on slave 192.172.0.14(192.172.0.14:3306).
	Thu Aug 23 03:11:35 2018 - [warning]  relay_log_purge=0 is not set on slave 192.172.0.14(192.172.0.14:3306).
	Thu Aug 23 03:11:35 2018 - [info] Checking replication filtering settings..
	Thu Aug 23 03:11:35 2018 - [info]  binlog_do_db= , binlog_ignore_db= 
	Thu Aug 23 03:11:35 2018 - [info]  Replication filtering check ok.
	Thu Aug 23 03:11:35 2018 - [info] GTID is supported. Skipping all SSH and Node package checking.
	Thu Aug 23 03:11:35 2018 - [info] Checking SSH publickey authentication settings on the current master..
	Thu Aug 23 03:11:35 2018 - [info] HealthCheck: SSH to 192.172.0.11 is reachable.
	Thu Aug 23 03:11:35 2018 - [info] 
	192.172.0.11 (current master)
	 +--192.172.0.13
	 +--192.172.0.14

	Thu Aug 23 03:11:35 2018 - [info] Checking replication health on 192.172.0.13..
	Thu Aug 23 03:11:35 2018 - [info]  ok.
	Thu Aug 23 03:11:35 2018 - [info] Checking replication health on 192.172.0.14..
	Thu Aug 23 03:11:35 2018 - [info]  ok.
	Thu Aug 23 03:11:35 2018 - [warning] master_ip_failover_script is not defined.
	Thu Aug 23 03:11:35 2018 - [warning] shutdown_script is not defined.
	Thu Aug 23 03:11:35 2018 - [info] Got exit code 0 (Not master dead).

	MySQL Replication Health is OK.

### 启动MHA-manager
	root@f7e548d16865:/home/mha# nohup masterha_manager --conf=/etc/mha/mysql_mha.conf &
	root@f7e548d16865:/home/mha# ps -ef | grep master
	root      1231     1  0 03:15 ?        00:00:00 perl /usr/local/bin/masterha_manager --conf=/etc/mha/mysql_mha.conf
	root      1268     1  0 03:15 ?        00:00:00 grep --color=auto master

### 查看manager日志
	root@f7e548d16865:/home/mha# cat manager.log 
	Thu Aug 23 03:15:08 2018 - [info] MHA::MasterMonitor version 0.56.
	Thu Aug 23 03:15:09 2018 - [info] Dead Servers:
	Thu Aug 23 03:15:09 2018 - [info] Alive Servers:
	Thu Aug 23 03:15:09 2018 - [info]   192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:15:09 2018 - [info]   192.172.0.13(192.172.0.13:3306)
	Thu Aug 23 03:15:09 2018 - [info]   192.172.0.14(192.172.0.14:3306)
	Thu Aug 23 03:15:09 2018 - [info] Alive Slaves:
	Thu Aug 23 03:15:09 2018 - [info]   192.172.0.13(192.172.0.13:3306)  Version=5.7.23-0ubuntu0.16.04.1-log (oldest major version between slaves) log-bin:enabled
	Thu Aug 23 03:15:09 2018 - [info]     GTID ON
	Thu Aug 23 03:15:09 2018 - [info]     Replicating from 192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:15:09 2018 - [info]     Primary candidate for the new Master (candidate_master is set)
	Thu Aug 23 03:15:09 2018 - [info]   192.172.0.14(192.172.0.14:3306)  Version=5.7.23-0ubuntu0.16.04.1-log (oldest major version between slaves) log-bin:enabled
	Thu Aug 23 03:15:09 2018 - [info]     GTID ON
	Thu Aug 23 03:15:09 2018 - [info]     Replicating from 192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:15:09 2018 - [info] Current Alive Master: 192.172.0.11(192.172.0.11:3306)
	Thu Aug 23 03:15:09 2018 - [info] Checking slave configurations..
	Thu Aug 23 03:15:09 2018 - [info]  read_only=1 is not set on slave 192.172.0.13(192.172.0.13:3306).
	Thu Aug 23 03:15:09 2018 - [warning]  relay_log_purge=0 is not set on slave 192.172.0.13(192.172.0.13:3306).
	Thu Aug 23 03:15:09 2018 - [info]  read_only=1 is not set on slave 192.172.0.14(192.172.0.14:3306).
	Thu Aug 23 03:15:09 2018 - [warning]  relay_log_purge=0 is not set on slave 192.172.0.14(192.172.0.14:3306).
	Thu Aug 23 03:15:09 2018 - [info] Checking replication filtering settings..
	Thu Aug 23 03:15:09 2018 - [info]  binlog_do_db= , binlog_ignore_db= 
	Thu Aug 23 03:15:09 2018 - [info]  Replication filtering check ok.
	Thu Aug 23 03:15:09 2018 - [info] GTID is supported. Skipping all SSH and Node package checking.
	Thu Aug 23 03:15:09 2018 - [info] Checking SSH publickey authentication settings on the current master..
	Thu Aug 23 03:15:09 2018 - [info] HealthCheck: SSH to 192.172.0.11 is reachable.
	Thu Aug 23 03:15:09 2018 - [info] 
	192.172.0.11 (current master)
	 +--192.172.0.13
	 +--192.172.0.14

	Thu Aug 23 03:15:09 2018 - [warning] master_ip_failover_script is not defined.
	Thu Aug 23 03:15:09 2018 - [warning] shutdown_script is not defined.
	Thu Aug 23 03:15:09 2018 - [info] Set master ping interval 1 seconds.
	Thu Aug 23 03:15:09 2018 - [warning] secondary_check_script is not defined. It is highly recommended setting it to check master reachability from two or more routes.
	Thu Aug 23 03:15:09 2018 - [info] Starting ping health check on 192.172.0.11(192.172.0.11:3306)..
	Thu Aug 23 03:15:09 2018 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..

### 测试——将mha_master_01的mysql停止，看主库是否切换
	root@d3ca3e85de35:/home/mha# service mysql stop     #主库上执行 

	#查看mha-manager监控日志
	略

	#在 mha_slave_02 看查看备份信息
	mysql> show slave status \G;
	*************************** 1. row ***************************
	               Slave_IO_State: Waiting for master to send event
	                  Master_Host: 192.172.0.13
	                  Master_User: replication
	                  Master_Port: 3306
	                Connect_Retry: 60
	              Master_Log_File: slave-binlog.000001
	          Read_Master_Log_Pos: 2216
	               Relay_Log_File: 4aa81d364f7d-relay-bin.000002
	                Relay_Log_Pos: 423
	        Relay_Master_Log_File: slave-binlog.000001
	             Slave_IO_Running: Yes
	            Slave_SQL_Running: Yes
	.
	.
	.

	#可以看出 Master_Host 从之前主库地址 192.172.0.11 切换到 mha_slave_01 192.172.0.13


	#重启 mha_master_01（192.172.0.11） 上的msyql，并配置为 mha_slave_01（192.172.0.13 目前是主库） 的从库
	#重新构建 主-从-从 集群
	root@d3ca3e85de35:/home/mha# service mysql start
	root@d3ca3e85de35:/home/mha# mysql -p
	mysql> CHANGE MASTER TO MASTER_HOST='192.172.0.11',  MASTER_USER='replication', MASTER_PASSWORD='replication', MASTER_AUTO_POSITION=1;
	mysql> start slave;

	#启动 mha_manager 继续监控，（刚才切换主从之后，mha_manager就退出了，不知道原因，所有每次主从切换之后要重新构建”主-从-从“集群之后再次启动mha_manager）
	#可以继续模拟 mha_slave_01 主库宕机，使主库回到 mha_master_01 上

	略





