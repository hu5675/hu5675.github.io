---
title: Docker中搭建mongodb分片副本集集群 (mongodb sharding repliset cluster)。
---

### 一、概念：sharded 集群由 shard、mongos、config server 三部分组成。
![](/images/mongodb_sharding_inDocker/1.png) 

### 二、本示例中的cluster组成如下（所有实例都在docker中创建,事先已经安在docker中安装好mongodb数据库）。docker中启动了10个容器，每个容器实例是一个Ubuntu系统，ubuntu中安装了mongodb数据库。
![](/images/mongodb_sharding_inDocker/2.png) 

#### 1、3组shard server 副本集replset (每个副本集2个mongod实例),配置如下：
![](/images/mongodb_sharding_inDocker/3.png) 
![](/images/mongodb_sharding_inDocker/4.png) 

#### 2、1组config server 副本集 (包含3个mongod实例),配置如下:
![](/images/mongodb_sharding_inDocker/5.png) 

#### 3、1个route (1个mongos实例),配置如下:
![](/images/mongodb_sharding_inDocker/6.png) 

主要红色部分是mongos启动时连接的配置服务器副本集地址。

### 三、启动副本集中的各个mongod实例并配置shard副本集、config server副本集。
#### 配置rs0副本集：
##### 进入mongodb容器实例(172.17.0.8):
	docker exec -it b083f0f79f10 /bin/bash
##### 启动mongod实例：
	mongod --shardsvr -f config/mongodb.conf &

##### 进入mongodb容器实例(172.17.0.9):
	docker exec -it 25d9bb2986c4 /bin/bash
##### 启动mongod实例：
	mongod --shardsvr -f config/mongodb.conf &

##### 切换到172.17.0.8，使用mongo连接mongod服务：
	mongo

	rs.initiate( {
	   _id : "rs0",
	   members: [
	      { _id: 0, host: "172.17.0.8:27017" },
	      { _id: 1, host: "172.17.0.9:27017" },
	   ]
	})

	rs.status()   //查询副本集状态
![](/images/mongodb_sharding_inDocker/7.png) 

#### 配置rs1副本集：
##### 进入mongodb容器实例(172.17.0.10):
	docker exec -it 152003f6729f /bin/bash
##### 启动mongod实例：
	mongod --shardsvr -f config/mongodb.conf &

##### 进入mongodb容器实例(172.17.0.15):
	docker exec -it 058202b95746 /bin/bash
##### 启动mongod实例：
	mongod --shardsvr -f config/mongodb.conf &

##### 切换到172.17.0.10，使用mongo连接mongod服务：
	mongo

	rs.initiate( {
	   _id : "rs1",
	   members: [
	      { _id: 0, host: "172.17.0.10:27017" },
	      { _id: 1, host: "172.17.0.15:27017" },
	   ]
	})

	rs.status()   //查询副本集状态

#### 配置rs2副本集：
##### 进入mongodb容器实例(172.17.0.16):
	docker exec -it 7528e1ebe846 /bin/bash
##### 启动mongod实例：
	mongod --shardsvr -f config/mongodb.conf &

##### 进入mongodb容器实例(172.17.0.17):
	docker exec -it 8c79ddd73cbc /bin/bash
##### 启动mongod实例：
	mongod --shardsvr -f config/mongodb.conf &

##### 切换到172.17.0.16
	mongo

	rs.initiate( {
	   _id : "rs2",
	   members: [
	      { _id: 0, host: "172.17.0.16:27017" },
	      { _id: 1, host: "172.17.0.17:27017" },
	   ]
	})

	rs.status()   //查询副本集状态

#### 配置config_srv副本集：
##### 进入mongodb容器实例(172.17.0.11):
	docker exec -it 7f04043533d2 /bin/bash
##### 启动mongod实例：
	mongod --configsvr -f config/mongodb.conf &
##### 进入mongodb容器实例(172.17.0.13):
	docker exec -it dc69a80c9749 /bin/bash
##### 启动mongod实例：
	mongod --configsvr -f config/mongodb.conf &
##### 进入mongodb容器实例(172.17.0.14):
	docker exec -it 9203651efd00 /bin/bash
##### 启动mongod实例：
	mongod --configsvr -f config/mongodb.conf &

##### 切换到172.17.0.11
	mongo

	rs.initiate( {
	   _id : "config_srv",
	   members: [
	      { _id: 0, host: "172.17.0.11:27017" },
	      { _id: 1, host: "172.17.0.13:27017" },
	      { _id: 2, host: "172.17.0.14:27017" },
	   ]
	})

	rs.status()   //查询副本集状态

### 四、启动mongos实例
##### 进入mongodb容器实例(172.17.0.12):
	docker exec -it a0c9b726c541 /bin/bash
##### 启动mongod实例：
	mongos -f config/mongodb.conf &
##### 为数据库（test）集合(users)添加分片信息：
	mongo    //连接mongos服务
	use admin   //切换admin数据库
	db.runCommand( {addshard : "rs0/172.17.0.8:27017,172.17.0.9:27017"});   //添加rs0副本集分片
	db.runCommand( {addshard : "rs1/172.17.0.10:27017,172.17.0.15:27017"})   //添加rs1副本集分片
	db.runCommand({enablesharding:"test"});   //设置分片存储的数据库
	db.runCommand({shardcollection:"test.users",key:{age:1}}); //设置分片的集合名称，且必须指定Shard key
	sh.status() 		//查看分片信息
![](/images/mongodb_sharding_inDocker/8.png) 


### 五、测试Sharding是否正常工作
#### 切换到route(172.17.0.12):
	mongo    //连接mongos服务
	mongos> use test  //使用test数据库
	mongos> db.users.getShardDistribution()   //查看集合的分片信息

	Shard rs0 at rs0/172.17.0.8:27017,172.17.0.9:27017
	 data : 0B docs : 0 chunks : 1
	 estimated data per chunk : 0B
	 estimated docs per chunk : 0

	Totals
	 data : 0B docs : 0 chunks : 1
	 Shard rs0 contains NaN% data, NaN% docs in cluster, avg obj size on shard : NaNGiB

	mongos> for(var i=1;i<=30000;i++) db.users.insert({age:i,name:"xu",addr:"BJ",country:"china"})     //插入3万测试数据
	WriteResult({ "nInserted" : 1 })
	mongos> db.users.getShardDistribution()   //查看集合的分片信息
	Shard rs0 at rs0/172.17.0.8:27017,172.17.0.9:27017
	 data : 2.28MiB docs : 30000 chunks : 1
	 estimated data per chunk : 2.28MiB
	 estimated docs per chunk : 30000

	Totals
	 data : 2.28MiB docs : 30000 chunks : 1
	 Shard rs0 contains 100% data, 100% docs in cluster, avg obj size on shard : 80B
为什么数据只在rs0分片上，rs1分片上没有。因为每个chunk的大小默认是200M，当插入的数据大小超过200M是就会触发chunk分裂。我们可以去手动修改chunk的大小。
#### 切换到config_srv副本集(172.17.0.11):
	mongo
	use config
	config_srv:PRIMARY> db.settings.find()   //查看chunk大小，首页查看无记录
	config_srv:PRIMARY> db.settings.save( { _id:"chunksize", value: 1 } )  //设置chunk的大小
#### 切换到route(172.17.0.12):
	mongo
	mongos> use test
	switched to db test
	mongos> for(var i=30001;i<=50000;i++) db.users.insert({age:i,name:"xu",addr:"BJ",country:"china"})   //再插入2万条数据
	WriteResult({ "nInserted" : 1 })
	mongos> db.users.getShardDistribution()

	Shard rs0 at rs0/172.17.0.8:27017,172.17.0.9:27017
	 data : 3.81MiB docs : 50000 chunks : 3
	 estimated data per chunk : 1.27MiB
	 estimated docs per chunk : 16666

	Shard rs1 at rs1/172.17.0.10:27017,172.17.0.15:27017
	 data : 1023KiB docs : 13107 chunks : 2
	 estimated data per chunk : 511KiB
	 estimated docs per chunk : 6553

	Totals
	 data : 4.81MiB docs : 63107 chunks : 5
	 Shard rs0 contains 79.23% data, 79.23% docs in cluster, avg obj size on shard : 80B
	 Shard rs1 contains 20.76% data, 20.76% docs in cluster, avg obj size on shard : 80B
可以看到分片rs0、rs1上都有数据记录了，说明我们的sharding能正常工作。

#### 继续在route(172.17.0.12)上操作添加rs2分片(即模拟当集合非常大时横向扩展)：
	mongo
	mongos> use admin
	switched to db admin
	mongos> db.runCommand( {addshard : "rs2/172.17.0.16:27017,172.17.0.17:27017"})  //添加rs2分片
	{
	        "shardAdded" : "rs2",
	        "ok" : 1,
	        "$clusterTime" : {
	                "clusterTime" : Timestamp(1524815645, 5),
	                "signature" : {
	                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
	                        "keyId" : NumberLong(0)
	                }
	        },
	        "operationTime" : Timestamp(1524815645, 5)
	}
	mongos> sh.status()
	--- Sharding Status --- 
	  sharding version: {
	        "_id" : 1,
	        "minCompatibleVersion" : 5,
	        "currentVersion" : 6,
	        "clusterId" : ObjectId("5ae2cd6e25f160add5630e46")
	  }
	  shards:
	        {  "_id" : "rs0",  "host" : "rs0/172.17.0.8:27017,172.17.0.9:27017",  "state" : 1 }
	        {  "_id" : "rs1",  "host" : "rs1/172.17.0.10:27017,172.17.0.15:27017",  "state" : 1 }
	        {  "_id" : "rs2",  "host" : "rs2/172.17.0.16:27017,172.17.0.17:27017",  "state" : 1 }
	  active mongoses:
	        "3.6.3" : 1
	  autosplit:
	        Currently enabled: yes
	  balancer:
	        Currently enabled:  yes
	        Currently running:  no
	        Failed balancer rounds in last 5 attempts:  0
	        Migration Results for the last 24 hours: 
	                3 : Success
	  databases:
	        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
	                config.system.sessions
	                        shard key: { "_id" : 1 }
	                        unique: false
	                        balancing: true
	                        chunks:
	                                rs0     1
	                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : rs0 Timestamp(1, 0) 
	        {  "_id" : "test",  "primary" : "rs0",  "partitioned" : true }
	                test.users
	                        shard key: { "age" : 1 }
	                        unique: false
	                        balancing: true
	                        chunks:
	                                rs0     2
	                                rs1     2
	                                rs2     1
	                        { "age" : { "$minKey" : 1 } } -->> { "age" : 2 } on : rs1 Timestamp(2, 0) 
	                        { "age" : 2 } -->> { "age" : 13108 } on : rs1 Timestamp(3, 0) 
	                        { "age" : 13108 } -->> { "age" : 19662 } on : rs2 Timestamp(4, 0) 
	                        { "age" : 19662 } -->> { "age" : 26216 } on : rs0 Timestamp(4, 1) 
	                        { "age" : 26216 } -->> { "age" : { "$maxKey" : 1 } } on : rs0 Timestamp(1, 5) 
	mongos> use test
	switched to db test
	mongos> for(var i=50001;i<=70000;i++) db.users.insert({age:i,name:"xu",addr:"BJ",country:"china"})   //再插入2万条记录
	WriteResult({ "nInserted" : 1 })
	mongos> db.users.getShardDistribution()  			//查看集成在分片上的存储信息  可以看到rs0、rs1、rs2上都有数据分布，到此user集合已经存有7万条数据

	Shard rs0 at rs0/172.17.0.8:27017,172.17.0.9:27017
	 data : 3.9MiB docs : 51240 chunks : 4
	 estimated data per chunk : 1000KiB
	 estimated docs per chunk : 12810

	Shard rs1 at rs1/172.17.0.10:27017,172.17.0.15:27017
	 data : 1.35MiB docs : 17708 chunks : 3
	 estimated data per chunk : 461KiB
	 estimated docs per chunk : 5902

	Shard rs2 at rs2/172.17.0.16:27017,172.17.0.17:27017
	 data : 1.58MiB docs : 20715 chunks : 3
	 estimated data per chunk : 539KiB
	 estimated docs per chunk : 6905

	Totals
	 data : 6.83MiB docs : 89663 chunks : 10
	 Shard rs0 contains 57.14% data, 57.14% docs in cluster, avg obj size on shard : 80B
	 Shard rs1 contains 19.74% data, 19.74% docs in cluster, avg obj size on shard : 80B
	 Shard rs2 contains 23.1% data, 23.1% docs in cluster, avg obj size on shard : 80B
#### 继续在route(172.17.0.12)上操作删除rs2分片
	mongo
	mongos> use admin
	switched to db admin  
	mongos> db.runCommand( {removeshard : "rs2/172.17.0.16:27017,172.17.0.17:27017"})    //移除rs2分片  state状态是started
	{
	        "msg" : "draining started successfully",
	        "state" : "started",
	        "shard" : "rs2",
	        "note" : "you need to drop or movePrimary these databases",
	        "dbsToMove" : [ ],
	        "ok" : 1,
	        "$clusterTime" : {
	                "clusterTime" : Timestamp(1524816991, 2),
	                "signature" : {
	                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
	                        "keyId" : NumberLong(0)
	                }
	        },
	        "operationTime" : Timestamp(1524816991, 2)
	}
	mongos> db.runCommand( {removeshard : "rs2/172.17.0.16:27017,172.17.0.17:27017"})  //再次执行	，可以看到state状态是ongoing，说明正在将rs2上的数据移动到其他分片上去
	{
	        "msg" : "draining ongoing",
	        "state" : "ongoing",
	        "remaining" : {
	                "chunks" : NumberLong(1),
	                "dbs" : NumberLong(0)
	        },
	        "note" : "you need to drop or movePrimary these databases",
	        "dbsToMove" : [ ],
	        "ok" : 1,
	        "$clusterTime" : {
	                "clusterTime" : Timestamp(1524817016, 1),
	                "signature" : {
	                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
	                        "keyId" : NumberLong(0)
	                }
	        },
	        "operationTime" : Timestamp(1524817016, 1)
	}
	mongos> db.runCommand( {removeshard : "rs2/172.17.0.16:27017,172.17.0.17:27017"})  //过一会再次执行，可以看到stata状态是completed
	{
	        "msg" : "removeshard completed successfully",
	        "state" : "completed",
	        "shard" : "rs2",
	        "ok" : 1,
	        "$clusterTime" : {
	                "clusterTime" : Timestamp(1524817221, 2),
	                "signature" : {
	                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
	                        "keyId" : NumberLong(0)
	                }
	        },
	        "operationTime" : Timestamp(1524817221, 2)
	}
	mongos> sh.status()    		//查看是否移除rs2分片
	--- Sharding Status --- 
	  sharding version: {
	        "_id" : 1,
	        "minCompatibleVersion" : 5,
	        "currentVersion" : 6,
	        "clusterId" : ObjectId("5ae2cd6e25f160add5630e46")
	  }
	  shards:
	        {  "_id" : "rs0",  "host" : "rs0/172.17.0.8:27017,172.17.0.9:27017",  "state" : 1 }
	        {  "_id" : "rs1",  "host" : "rs1/172.17.0.10:27017,172.17.0.15:27017",  "state" : 1 }
	  active mongoses:
	        "3.6.3" : 1
	  autosplit:
	        Currently enabled: yes
	  balancer:
	        Currently enabled:  yes
	        Currently running:  no
	        Failed balancer rounds in last 5 attempts:  0
	        Migration Results for the last 24 hours: 
	                8 : Success
	  databases:
	        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
	                config.system.sessions
	                        shard key: { "_id" : 1 }
	                        unique: false
	                        balancing: true
	                        chunks:
	                                rs0     1
	                        { "_id" : { "$minKey" : 1 } } -->> { "_id" : { "$maxKey" : 1 } } on : rs0 Timestamp(1, 0) 
	        {  "_id" : "test",  "primary" : "rs0",  "partitioned" : true }
	                test.users
	                        shard key: { "age" : 1 }
	                        unique: false
	                        balancing: true
	                        chunks:
	                                rs0     5
	                                rs1     5
	                        { "age" : { "$minKey" : 1 } } -->> { "age" : 2 } on : rs1 Timestamp(2, 0) 
	                        { "age" : 2 } -->> { "age" : 13108 } on : rs1 Timestamp(3, 0) 
	                        { "age" : 13108 } -->> { "age" : 19662 } on : rs1 Timestamp(7, 0) 
	                        { "age" : 19662 } -->> { "age" : 26216 } on : rs0 Timestamp(5, 1) 
	                        { "age" : 26216 } -->> { "age" : 32769 } on : rs0 Timestamp(4, 2) 
	                        { "age" : 32769 } -->> { "age" : 39323 } on : rs0 Timestamp(4, 3) 
	                        { "age" : 39323 } -->> { "age" : 51240 } on : rs0 Timestamp(4, 4) 
	                        { "age" : 51240 } -->> { "age" : 57793 } on : rs0 Timestamp(8, 0) 
	                        { "age" : 57793 } -->> { "age" : 65400 } on : rs1 Timestamp(9, 0) 
	                        { "age" : 65400 } -->> { "age" : { "$maxKey" : 1 } } on : rs1 Timestamp(6, 0) 
	
	mongos> use test		//切换test
	switched to db test
	mongos> db.users.getShardDistribution()    //查看移除rs2分片之后  user集合的数据存储分布情况

	Shard rs0 at rs0/172.17.0.8:27017,172.17.0.9:27017
	 data : 2.9MiB docs : 38131 chunks : 5
	 estimated data per chunk : 595KiB
	 estimated docs per chunk : 7626

	Shard rs1 at rs1/172.17.0.10:27017,172.17.0.15:27017
	 data : 2.43MiB docs : 31869 chunks : 5
	 estimated data per chunk : 497KiB
	 estimated docs per chunk : 6373

	Totals
	 data : 5.33MiB docs : 70000 chunks : 10
	 Shard rs0 contains 54.47% data, 54.47% docs in cluster, avg obj size on shard : 80B
	 Shard rs1 contains 45.52% data, 45.52% docs in cluster, avg obj size on shard : 80B

到此已经cluster搭建测试结束。





