---
title: 利用docker搭建redis cluster(集群) ——3主3从
date: 2018-05-01 10:01:25
---

### 一、介绍
	Redis Cluster是Redis的分布式解决方案，在Redis 3.0版本正式推出的，
	有效解决了Redis分布式方面的需求。当遇到单机内存、并发、流量等瓶颈时，可以采用Cluster架构达到负载均衡的目的。
	Redis Cluster采用虚拟分槽，虚拟槽分区巧妙地使用了哈希空间，使用分散度良好的哈希函数把所有的数据映射到一个固定范围内的整数集合，
	整数定义为槽（slot）。比如Redis Cluster槽的范围是0 ～ 16383。槽是集群内数据管理和迁移的基本单位。
	采用大范围的槽的主要目的是为了方便数据的拆分和集群的扩展，每个节点负责一定数量的槽。
<a href="http://www.redis.cn/topics/cluster-tutorial.html" target="_blank">Redis Cluster详细文档</a>

### 二、节点准备(主机中需要装有docker)
#### 1、在主机中创建8001、8002、8003、8004、8005、8006目录用于存放6个Redis节点的配置文件redis.conf
	[root@izbp13cqwumhn4wi93is2wz cluster-test]# pwd
	/home/mars/cluster-test
	[root@izbp13cqwumhn4wi93is2wz cluster-test]# ll
	total 40
	drwxr-xr-x 2 root root 4096 Apr 28 11:46 8001
	drwxr-xr-x 2 root root 4096 Mar  9 22:28 8002
	drwxr-xr-x 2 root root 4096 Mar  9 22:28 8003
	drwxr-xr-x 2 root root 4096 Mar  9 22:28 8004
	drwxr-xr-x 2 root root 4096 Mar  9 22:29 8005
	drwxr-xr-x 2 root root 4096 Mar  9 22:29 8006

#### 2、分别在8001、8002、8003、8004、8005、8006中创建redis.conf文件并添加内容

##### 8001/redis.conf
	port 8001
	cluster-enabled yes
	cluster-config-file nodes-8001.conf
	cluster-node-timeout 5000
	appendonly yes

##### 8002/redis.conf
	port 8002
	cluster-enabled yes
	cluster-config-file nodes-8002.conf
	cluster-node-timeout 5000
	appendonly yes

##### 8002/redis.conf
	port 8003
	cluster-enabled yes
	cluster-config-file nodes-8003.conf
	cluster-node-timeout 5000
	appendonly yes

##### 8002/redis.conf
	port 8004
	cluster-enabled yes
	cluster-config-file nodes-8004.conf
	cluster-node-timeout 5000
	appendonly yes

##### 8002/redis.conf
	port 8005
	cluster-enabled yes
	cluster-config-file nodes-8005.conf
	cluster-node-timeout 5000
	appendonly yes

##### 8002/redis.conf
	port 8006
	cluster-enabled yes
	cluster-config-file nodes-8006.conf
	cluster-node-timeout 5000
	appendonly yes


#### 3、查看docker镜像并启动6个redis_v4.0.8容器实例（redis_v4.0.8镜像是在ubuntu镜像基础上自己装了redis，当运行redis_v4.0.8容器时相当于运行了一个ubuntu虚拟机）
![](/images/redis_cluster_inDocker/1.png) 
##### 启动8001对应实例——master_1：
	docker run -it --net=host --name master_1 -v /home/mars/cluster-test/8001/redis.conf:/home/redis-4.0.8/redis.conf docker.io/ubuntu:redis_v4.0.8
##### 启动8002对应实例——master_2：	
	docker run -it --net=host --name master_2 -v /home/mars/cluster-test/8002/redis.conf:/home/redis-4.0.8/redis.conf docker.io/ubuntu:redis_v4.0.8
##### 启动8003对应实例——master_3：
	docker run -it --net=host --name master_3 -v /home/mars/cluster-test/8003/redis.conf:/home/redis-4.0.8/redis.conf docker.io/ubuntu:redis_v4.0.8
##### 启动8004对应实例——slave_1：
	docker run -it --net=host --name slave_1 -v /home/mars/cluster-test/8004/redis.conf:/home/redis-4.0.8/redis.conf docker.io/ubuntu:redis_v4.0.8
##### 启动8005对应实例——slave_2：
	docker run -it --net=host --name slave_2 -v /home/mars/cluster-test/8005/redis.conf:/home/redis-4.0.8/redis.conf docker.io/ubuntu:redis_v4.0.8
##### 启动8006对应实例——slave_3：
	docker run -it --net=host --name slave_3 -v /home/mars/cluster-test/8006/redis.conf:/home/redis-4.0.8/redis.conf docker.io/ubuntu:redis_v4.0.8

##### 查看启动的6个容器实例
![](/images/redis_cluster_inDocker/2.png) 


<font color="#dd0000">目前相当于有6台装有redis服务的Ubuntu系统（redis未运行）。</font>

下面分别进入到6台ubuntu系统中去启动redis服务并配置。

### 三、启动redis节点并配置redis cluster。

#### 启动6个redis节点

##### 新开窗口进入master_1：
	docker exec -it ec7b22d64831 /bin/bash   #进入master_1容器
	cd home/redis-4.0.8						 #进入redis服务目录
	./src/redis-server redis.conf &	         #后台进程模式启动redis server服务

	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8001			#以集群模式连接本机redis-server
	127.0.0.1:8001> CLUSTER NODES  														# 查看集群节点状态
	24a3c7107afadafa95132420ac271e1e9a571b5c :8001@18001 myself,master - 0 0 0 connected
	127.0.0.1:8001> 

##### 新开窗口进入master_2：
	docker exec -it b0608539bb46 /bin/bash   #进入master_1容器
	cd home/redis-4.0.8						 #进入redis服务目录
	./src/redis-server redis.conf &	         #后台进程模式启动redis server服务

	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8002
	127.0.0.1:8002> 
	127.0.0.1:8002> CLUSTER NODES
	52b2395f63124148fb6f1d777a121a08319efd64 :8002@18002 myself,master - 0 0 0 connected
	127.0.0.1:8002> 

<font color="#dd0000"> 同样方式进入并启动redis-server （master_3、slave_1、slave_2、slave_3）</font>

现在在6台ubuntu系统中启动的6个单独的redis-server节点，彼此之间并没有任何联系。

#### 节点握手使6个节点之间能彼此发现
##### 切换到master_1，使用redis-cli连接本机redis暴露的端口8001
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8001

	127.0.0.1:8001> CLUSTER MEET 127.0.0.1 8002					#和8002节点握手
	OK
	127.0.0.1:8001> 36:M 30 Apr 15:05:34.450 # IP address for this node updated to 127.0.0.1
	127.0.0.1:8001> 
	127.0.0.1:8001> CLUSTER MEET 127.0.0.1 8003					#和8003节点握手
	OK
	127.0.0.1:8001> CLUSTER MEET 127.0.0.1 8004					#和8004节点握手
	OK
	127.0.0.1:8001> CLUSTER MEET 127.0.0.1 8005					#和8005节点握手
	OK
	127.0.0.1:8001> CLUSTER MEET 127.0.0.1 8006					#和8006节点握手
	OK
	127.0.0.1:8001> CLUSTER NODES 								#查看集群节点状态
	0f11017d651c07e8614e9221ecbdbb0f7f4ce438 127.0.0.1:8005@18005 master - 0 1525100795000 4 connected
	cf29d49076528a34604f7b9b447d927f9086196e 127.0.0.1:8003@18003 master - 0 1525100794000 0 connected
	24a3c7107afadafa95132420ac271e1e9a571b5c 127.0.0.1:8001@18001 myself,master - 0 1525100794000 1 connected
	52b2395f63124148fb6f1d777a121a08319efd64 127.0.0.1:8002@18002 master - 0 1525100795662 2 connected
	b45471df0395e92da1f4e0a5a07c6a9b8316f67b 127.0.0.1:8004@18004 master - 0 1525100795000 3 connected
	2471dfd538f59af44858bc7f7a6ba3eee93b2e30 127.0.0.1:8006@18006 master - 0 1525100796061 5 connected
	127.0.0.1:8001> 
	127.0.0.1:8001> CLUSTER INFO 								#查看集群信息
	cluster_state:fail
	cluster_slots_assigned:0
	cluster_slots_ok:0
	cluster_slots_pfail:0
	cluster_slots_fail:0
	cluster_known_nodes:6
	cluster_size:0
	cluster_current_epoch:5
	cluster_my_epoch:1
	cluster_stats_messages_ping_sent:266
	cluster_stats_messages_pong_sent:267
	cluster_stats_messages_meet_sent:5
	cluster_stats_messages_sent:538
	cluster_stats_messages_ping_received:267
	cluster_stats_messages_pong_received:271
	cluster_stats_messages_received:538
	127.0.0.1:8001> 

#### 给master_1、master_2、master_3分配槽（redis cluster总共有16384个槽），在任意节点上执行（本例演示在master_1节点）：
	ctrl+c      #断开redis-cli连接
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -h 127.0.0.1 -p 8001 cluster addslots {0..5461} #给master_1分配槽
	OK
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -h 127.0.0.1 -p 8002 cluster addslots {5462..10922}  #给master_2分配槽
	OK
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -h 127.0.0.1 -p 8003 cluster addslots {10923..16383}  #给master_3分配槽
	OK
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# 
	#redis-cli连接并查看集群节点信息和状态,可以发现master_1、master_2、master_3已经分配好了槽，并且集群状态是ok
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8001     
	127.0.0.1:8001> CLUSTER NODES
	0f11017d651c07e8614e9221ecbdbb0f7f4ce438 127.0.0.1:8005@18005 master - 0 1525101200362 4 connected
	cf29d49076528a34604f7b9b447d927f9086196e 127.0.0.1:8003@18003 master - 0 1525101199861 0 connected 10923-16383
	24a3c7107afadafa95132420ac271e1e9a571b5c 127.0.0.1:8001@18001 myself,master - 0 1525101197000 1 connected 0-5461
	52b2395f63124148fb6f1d777a121a08319efd64 127.0.0.1:8002@18002 master - 0 1525101199560 2 connected 5462-10922
	b45471df0395e92da1f4e0a5a07c6a9b8316f67b 127.0.0.1:8004@18004 master - 0 1525101199360 3 connected
	2471dfd538f59af44858bc7f7a6ba3eee93b2e30 127.0.0.1:8006@18006 master - 0 1525101200000 5 connected
	127.0.0.1:8001> 
	127.0.0.1:8001> CLUSTER INFO
	cluster_state:ok
	cluster_slots_assigned:16384
	cluster_slots_ok:16384
	cluster_slots_pfail:0
	cluster_slots_fail:0
	cluster_known_nodes:6
	cluster_size:3
	cluster_current_epoch:5
	cluster_my_epoch:1
	cluster_stats_messages_ping_sent:938
	cluster_stats_messages_pong_sent:943
	cluster_stats_messages_meet_sent:5
	cluster_stats_messages_sent:1886
	cluster_stats_messages_ping_received:943
	cluster_stats_messages_pong_received:943
	cluster_stats_messages_received:1886
<font color="#dd0000"> 现在3主已经完成。</font>


#### 为3个主节点分别配置一个从节点
##### 为master_1配置slave节点，任意节点中使用redis-cli连接slave_1节点中的redis-server服务：
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8004              
	127.0.0.1:8004> CLUSTER NODES    
	2471dfd538f59af44858bc7f7a6ba3eee93b2e30 127.0.0.1:8006@18006 master - 0 1525101718000 5 connected
	24a3c7107afadafa95132420ac271e1e9a571b5c 127.0.0.1:8001@18001 master - 0 1525101719000 1 connected 0-5461
	0f11017d651c07e8614e9221ecbdbb0f7f4ce438 127.0.0.1:8005@18005 master - 0 1525101719344 4 connected
	52b2395f63124148fb6f1d777a121a08319efd64 127.0.0.1:8002@18002 master - 0 1525101718043 2 connected 5462-10922
	cf29d49076528a34604f7b9b447d927f9086196e 127.0.0.1:8003@18003 master - 0 1525101718344 0 connected 10923-16383
	b45471df0395e92da1f4e0a5a07c6a9b8316f67b 127.0.0.1:8004@18004 myself,master - 0 1525101717000 3 connected
	127.0.0.1:8004> 
	127.0.0.1:8004> CLUSTER REPLICATE 24a3c7107afadafa95132420ac271e1e9a571b5c			#让当前节点slave_1（8004）节点成为master_1的从节点，24a3c7107afadafa95132420ac271e1e9a571b5c是master_1(8001)的节点id。
	OK

##### 为master_2配置slave节点，任意节点中使用redis-cli连接slave_2节点中的redis-server服务：
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8005
	127.0.0.1:8005> CLUSTER NODES
	cf29d49076528a34604f7b9b447d927f9086196e 127.0.0.1:8003@18003 master - 0 1525101855000 0 connected 10923-16383
	0f11017d651c07e8614e9221ecbdbb0f7f4ce438 127.0.0.1:8005@18005 myself,master - 0 1525101853000 4 connected
	2471dfd538f59af44858bc7f7a6ba3eee93b2e30 127.0.0.1:8006@18006 master - 0 1525101855061 5 connected
	24a3c7107afadafa95132420ac271e1e9a571b5c 127.0.0.1:8001@18001 master - 0 1525101854259 1 connected 0-5461
	52b2395f63124148fb6f1d777a121a08319efd64 127.0.0.1:8002@18002 master - 0 1525101853558 2 connected 5462-10922
	b45471df0395e92da1f4e0a5a07c6a9b8316f67b 127.0.0.1:8004@18004 slave 24a3c7107afadafa95132420ac271e1e9a571b5c 0 1525101855262 3 connected
	127.0.0.1:8005> CLUSTER REPLICATE  52b2395f63124148fb6f1d777a121a08319efd64		#让当前节点slave_2（8005）节点成为master_1的从节点，52b2395f63124148fb6f1d777a121a08319efd64 是master_2(8002)的节点id。
	OK

##### 为master_3配置slave节点，任意节点中使用redis-cli连接slave_3节点中的redis-server服务：
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8006
	127.0.0.1:8006> CLUSTER NODES
	cf29d49076528a34604f7b9b447d927f9086196e 127.0.0.1:8003@18003 master - 0 1525101986000 0 connected 10923-16383
	b45471df0395e92da1f4e0a5a07c6a9b8316f67b 127.0.0.1:8004@18004 slave 24a3c7107afadafa95132420ac271e1e9a571b5c 0 1525101986978 3 connected
	52b2395f63124148fb6f1d777a121a08319efd64 127.0.0.1:8002@18002 master - 0 1525101985576 2 connected 5462-10922
	24a3c7107afadafa95132420ac271e1e9a571b5c 127.0.0.1:8001@18001 master - 0 1525101987579 1 connected 0-5461
	0f11017d651c07e8614e9221ecbdbb0f7f4ce438 127.0.0.1:8005@18005 slave 52b2395f63124148fb6f1d777a121a08319efd64 0 1525101987000 4 connected
	2471dfd538f59af44858bc7f7a6ba3eee93b2e30 127.0.0.1:8006@18006 myself,master - 0 1525101986000 5 connected
	127.0.0.1:8006> CLUSTER REPLICATE cf29d49076528a34604f7b9b447d927f9086196e     #让当前节点slave_3（8006）节点成为master_1的从节点，cf29d49076528a34604f7b9b447d927f9086196e 是master_3(8003)的节点id。
	OK
	#查看集群最终节点状态和信息,可见3主3从已经配置完成
	127.0.0.1:8006> CLUSTER NODES   
	cf29d49076528a34604f7b9b447d927f9086196e 127.0.0.1:8003@18003 master - 0 1525102074120 0 connected 10923-16383
	b45471df0395e92da1f4e0a5a07c6a9b8316f67b 127.0.0.1:8004@18004 slave 24a3c7107afadafa95132420ac271e1e9a571b5c 0 1525102073118 3 connected
	52b2395f63124148fb6f1d777a121a08319efd64 127.0.0.1:8002@18002 master - 0 1525102073519 2 connected 5462-10922
	24a3c7107afadafa95132420ac271e1e9a571b5c 127.0.0.1:8001@18001 master - 0 1525102073000 1 connected 0-5461
	0f11017d651c07e8614e9221ecbdbb0f7f4ce438 127.0.0.1:8005@18005 slave 52b2395f63124148fb6f1d777a121a08319efd64 0 1525102072617 4 connected
	2471dfd538f59af44858bc7f7a6ba3eee93b2e30 127.0.0.1:8006@18006 myself,slave cf29d49076528a34604f7b9b447d927f9086196e 0 1525102073000 5 connected
	127.0.0.1:8006> 

<font color="#dd0000"> 到此3主3从redis cluster集群配置完成</font>

### 四、测试数据（以key:{10}:111,key:{test}:111,key:{hu}:111为例，虚拟槽计算规则当遇到有{}时，就会以{}里面的字符串来做槽的计算：slot = CRC16(key)&16383 ）
	#使用 CLUSTER KEYSLOT 来查看key会被分配到那个槽中
	127.0.0.1:8001> CLUSTER KEYSLOT key:{10}:111
	(integer) 247			#247槽 属于master_1
	127.0.0.1:8001> CLUSTER KEYSLOT key:{test}:111
	(integer) 6918			#6918槽 属于master_2
	127.0.0.1:8001> CLUSTER KEYSLOT key:{hu}:111
	(integer) 11441			#11441槽 属于master_3
	127.0.0.1:8001> 
	#分配为以上3个key设置对应的value。
	127.0.0.1:8001> set key:{10}:111 value:{10}:111
	OK
	127.0.0.1:8001> set key:{test}:111 value:{test}:111				#slot 属于master_2
	-> Redirected to slot [6918] located at 127.0.0.1:8002			#在master_1上设置，重定向到了master_2上
	OK
	127.0.0.1:8002> set key:{hu}:111 value:{hu}:111					#slot 属于master_3
	-> Redirected to slot [11441] located at 127.0.0.1:8003			#在master_2上设置，重定向到了master_3上
	OK
	127.0.0.1:8003> 
	# 获取以上设置的value，连接8001节点
	root@izbp13cqwumhn4wi93is2wz:/home/redis-4.0.8# ./src/redis-cli -c -p 8001
	127.0.0.1:8001> 
	127.0.0.1:8001> get key:{10}:111			#slot 属于master_1，获取时能在本机直接取到value
	"value:{10}:111"
	127.0.0.1:8001> 	
	127.0.0.1:8001> get key:{test}:111			#slot 属于master_2，获取时重定向到master_2中去取value
	-> Redirected to slot [6918] located at 127.0.0.1:8002
	"value:{test}:111"
	127.0.0.1:8002> 
	127.0.0.1:8002> get key:{hu}:111			#slot 属于master_3，获取时重定向到master_3中去取value
	-> Redirected to slot [11441] located at 127.0.0.1:8003
	"value:{hu}:111"
	127.0.0.1:8003> 

<font color="#dd0000"> 测试完成</font>




