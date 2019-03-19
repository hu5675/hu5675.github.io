---
title: 利用Nginx作为ES负载均衡
date: 2019-3-19 10:01:25
---

### 一、需求缘起

	学习慕课网《基于ElasticSearch搜房网实战》做笔记

	es节点可分为：
	
	《主节点》 

	配置 node.master=true,node.data=false 即可成为主节点。 主节点的主要职责是和集群操作相关的内容，如创建或删除索引，跟踪哪些节点是群集的一部分，并决定哪些分片分配给相关的节点。

	《数据节点》

	配置 node.master=false,node.data=true 即可成为数据节点。 数据节点主要是存储索引数据的节点，主要对文档进行增删改查操作，聚合操作等。数据节点对cpu，内存，io要求较高。

	《客户端节点》

	配置 node.master=false,node.data=false 即可成为客户端节点。该节点只能处理路由请求，处理搜索，分发索引操作等，可看着是负载平衡器，将检索请求均衡的分发到不同的节点。

	es 生产环境最好是把主节点、数据节点和客户端节点单独分开，让其利用各自硬件资源各司其职发挥其最大作用。

	当数据量大时，索引检索时通过客户端节点负载到数据节点时，压力会集中在客户端节点。 这时可引入其他负载中间件（nginx）作为负载节点，将请求直接负载到数据节点。

### 二、环境准备

	es 集群安装，可参考《ELK日志收集框架部署步骤》（略）

	es-head 安装, 可参考《ELK日志收集框架部署步骤》（略）

	nginx 安装（略）

### 三、检查安装是否成功

![](/images/es_load/1.png) 

![](/images/es_load/2.png) 

![](/images/es_load/3.png) 

### 四、利用postman创建mars索引
	
	{
	  "settings": {
	    "number_of_replicas": 0,
	    "number_of_shards": 5,
	    "index.store.type": "niofs",
	    "index.query.default_field": "title",
	    "index.unassigned.node_left.delayed_timeout": "5m"
	  },


	  "mappings": {
	    "house": {
	      "dynamic": "strict",
	      "_all": {
	        "enabled": false
	      },
	      "properties": {
	        "houseId": {
	          "type": "long"
	        },
	        "title": {
	          "type": "text",
	          "index": "analyzed",
	          "analyzer": "ik_smart",
	          "search_analyzer": "ik_smart"
	        },
	        "price": {
	          "type": "integer"
	        },
	        "area": {
	          "type": "integer"
	        },
	        "createTime": {
	          "type": "date",
	          "format": "strict_date_optional_time||epoch_millis"
	        },
	        "lastUpdateTime": {
	          "type": "date",
	          "format": "strict_date_optional_time||epoch_millis"
	        },
	        "cityEnName": {
	          "type": "keyword"
	        },
	        "regionEnName": {
	          "type": "keyword"
	        },
	        "direction": {
	          "type": "integer"
	        },
	        "distanceToSubway": {
	          "type": "integer"
	        },
	        "subwayLineName": {
	          "type": "keyword"
	        },
	        "subwayStationName": {
	          "type": "keyword"
	        },
	        "tags": {
	          "type": "text"
	        },
	        "street": {
	          "type": "keyword"
	        },
	        "district": {
	          "type": "keyword"
	        },
	        "description": {
	          "type": "text",
	          "index": "analyzed",
	          "analyzer": "ik_smart",
	          "search_analyzer": "ik_smart"
	        },
	        "layoutDesc" : {
	          "type": "text",
	          "index": "analyzed",
	          "analyzer": "ik_smart",
	          "search_analyzer": "ik_smart"
	        },
	        "traffic": {
	          "type": "text",
	          "index": "analyzed",
	          "analyzer": "ik_smart",
	          "search_analyzer": "ik_smart"
	        },
	        "roundService": {
	          "type": "text",
	          "index": "analyzed",
	          "analyzer": "ik_smart",
	          "search_analyzer": "ik_smart"
	        },
	        "rentWay": {
	          "type": "integer"
	        },
	        "suggest": {
	          "type": "completion"
	        },
	        "location": {
	          "type": "geo_point"
	        }
	      }
	    }
	  }
	}

![](/images/es_load/4.png) 
	

### 四、利用Java Transport client 在mars索引中添加一条数据

	需要创建项目并编写代码（不具体列出，可参考源码）

### 五、利用es-head检查索引 mars 以及数据

![](/images/es_load/5.png) 


### 六、利用Java Transport client 检查索引 mars 

	1、test es 配置如下

![](/images/es_load/7.png) 

	2、test es 结果如下

![](/images/es_load/6.png) 

<br/>

<font color="#dd0000" size="16">到此，测试环境以及数据全部准备完成。</font>

### 七、配置Nginx Tcp反向代理。

	编辑nginx.conf , 在events 平级下面加入以下TCP负载配置，并重启。

	stream {
        upstream backend {
                server 127.0.0.1:9300;
                server 127.0.0.1:9301;
                server 127.0.0.1:9302;
        }

        server {
                listen 9999;
                proxy_timeout 20s;
                proxy_pass backend;
        }
	}

### 八、测试es transport client 连接端口9999是否能正常访问

	1、test es 配置如下

![](/images/es_load/9.png) 

	2、test es 结果如下

![](/images/es_load/8.png) 

<br/>

<font color="#dd0000" size="16">到此，Nginx 负载 ES 搭建完成</font>