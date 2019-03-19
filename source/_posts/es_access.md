---
title: 利用Nginx作为ES安全访问控制
date: 2019-3-19 10:01:25
---

### 一、需求缘起

	学习慕课网《基于ElasticSearch搜房网实战》做笔记

	描述：es-head/es http请求可不经过任何安全验证在浏览器访问，具有巨大的潜在威胁。

	方案：利用Nginx进行访问控制。

### 二、环境准备

	参考《利用Nginx作为ES负载均衡》 一到六。

### 修改 ES 各个节点配置，不允许外网访问，并重启， 本练习中使用docker构建环境，所以配置有点区别。

	es 节点

	cluster.name: es-cluster
	node.name: es
	node.master: true
	node.data: true

	network.host: es

	http.port: 9200
	transport.tcp.port: 9300
	http.cors.enabled: true
	http.cors.allow-origin: "*"
	http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type

	discovery.zen.ping.unicast.hosts: ["es:9300", "es1:9300", "es2:9300"]
	discovery.zen.minimum_master_nodes: 2
	discovery.zen.ping_timeout: 5s

	bootstrap.memory_lock: true
	action.destructive_requires_name: true

	es1 节点

	cluster.name: es-cluster
	node.name: es1
	node.master: true
	node.data: true

	network.host: es1

	http.port: 9200
	transport.tcp.port: 9300
	http.cors.enabled: true
	http.cors.allow-origin: "*"
	http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type

	discovery.zen.ping.unicast.hosts: ["es:9300", "es1:9300", "es2:9300"]
	discovery.zen.minimum_master_nodes: 2
	discovery.zen.ping_timeout: 5s

	bootstrap.memory_lock: true
	action.destructive_requires_name: true

	es2 节点

	cluster.name: es-cluster
	node.name: es2
	node.master: true
	node.data: true

	network.host: es2

	http.port: 9200
	transport.tcp.port: 9300
	http.cors.enabled: true
	http.cors.allow-origin: "*"
	http.cors.allow-headers: Authorization,X-Requested-With,Content-Length,Content-Type

	discovery.zen.ping.unicast.hosts: ["es:9300", "es1:9300", "es2:9300"]
	discovery.zen.minimum_master_nodes: 2
	discovery.zen.ping_timeout: 5s

	bootstrap.memory_lock: true
	action.destructive_requires_name: true

	使用curl访问docker容器中的es节点：
![](/images/es_access/2.png) 

	在浏览器上不能访问

![](/images/es_access/1.png) 

### 配置Nginx http负载

	
    upstream elasticsearch {
        server 192.168.80.2:9200;
        server 192.168.80.3:9200;
        server 192.168.80.4:9200;
    }

    server {
        listen       8888;
        server_name  localhost;

        location / {
                proxy_pass    http://elasticsearch;
                proxy_redirect off;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    重启并外网验证

![](/images/es_access/3.png) 


### 配置Nginx 安全访问

	1） 在主机上新建目录password，执行密码生成命令。

	[root@iZm5ehu0llwzjkgb44sii3Z es-cluster]# mkdir password
	[root@iZm5ehu0llwzjkgb44sii3Z es-cluster]# 
	[root@iZm5ehu0llwzjkgb44sii3Z es-cluster]# cd password/
	[root@iZm5ehu0llwzjkgb44sii3Z password]# 
	[root@iZm5ehu0llwzjkgb44sii3Z password]# printf "mars:$(openssl passwd -crypt elasticsearch)\n" > passwords
	Warning: truncating password to 8 characters
	[root@iZm5ehu0llwzjkgb44sii3Z password]# 
	[root@iZm5ehu0llwzjkgb44sii3Z password]# ll
	total 4
	-rw-r--r-- 1 root root 19 Mar 19 17:28 passwords
	[root@iZm5ehu0llwzjkgb44sii3Z password]# 
	[root@iZm5ehu0llwzjkgb44sii3Z password]# 

	2）配置Nginx 密码访问

	upstream elasticsearch {
        server 192.168.80.2:9200;
        server 192.168.80.3:9200;
        server 192.168.80.4:9200;
    }

    server {
        listen       8888;
        server_name  localhost;

        auth_basic "Protection";
        auth_basic_user_file /mnt/docker-elk/es-cluster/password/passwords;

        location / {
                proxy_pass    http://elasticsearch;
                proxy_redirect off;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    重启并在浏览器验证密码是否生效

![](/images/es_access/4.png) 

<br/>

![](/images/es_access/5.png) 


###  es-head 验证

![](/images/es_access/6.png) 

	可以看到es-head还是不能访问，nginx需要配置跨域。

	 upstream elasticsearch {
        server 192.168.80.2:9200;
        server 192.168.80.3:9200;
        server 192.168.80.4:9200;
    }

    server {
        listen       8888;
        server_name  localhost;

        auth_basic "Protection";
        auth_basic_user_file /mnt/docker-elk/es-cluster/password/passwords;

        location / {
                proxy_pass    http://elasticsearch;
                proxy_redirect off;

                if ($request_method = 'OPTIONS') {
                        add_header 'Access-Control-Allow-Origin' '*';
                        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, POST, PUT';
                        add_header 'Access-Control-Allow-Headers' 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,X-CustomHeader,Content-Range,Range';
                        add_header 'Acessss-Control-Max-Age' 172800;
                        add_header 'Content-Type' 'text-plain;charset=utf-8';
                        add_header 'Content-Length' 0;
                        return 204;
                }
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    重启 es-head再次验证

![](/images/es_access/7.png) 


### 选择性认证，如： _cat/xxx , 让 _cat 相关的健康检查接口不需要验证。
	
	upstream elasticsearch {
        server 192.168.80.2:9200;
        server 192.168.80.3:9200;
        server 192.168.80.4:9200;
    }


    server {
        listen       8888;
        server_name  localhost;

        location @general {
                proxy_pass    http://elasticsearch;
                proxy_redirect off;

        }

        location @need_protection {
                auth_basic "Protection";
                auth_basic_user_file /mnt/docker-elk/es-cluster/password/passwords;

                proxy_pass    http://elasticsearch;
                proxy_redirect off;
        }

        location / {

                if ($request_method = 'OPTIONS') {
                        add_header 'Access-Control-Allow-Origin' '*';
                        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, POST, PUT';
                        add_header 'Access-Control-Allow-Headers' 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,X-CustomHeader,Content-Range,Range';
                        add_header 'Acessss-Control-Max-Age' 172800;
                        add_header 'Content-Type' 'text-plain;charset=utf-8';
                        add_header 'Content-Length' 0;
                        return 204;
                }

                error_page 598 = @general;
                error_page 599 = @need_protection;

                if ($request_uri ~ ^/_cat.*$) {
                        return 598;
                }
                return 599;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }


### 重启再次验证,

![](/images/es_access/8.png) 

<br/>

![](/images/es_access/9.png)     

	可见，_cat/***  不需要验证  非 _cat/*** 则需要验证。

