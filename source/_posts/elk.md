---
title: ELK日志收集框架部署步骤
date: 2018-11-19 16:01:25
---

### 一、需求缘起

	项目业务越来越复杂，单体项目维护越来越麻烦。 微服务化亟需落实，但随之而来的服务日志怎么管理.
	每个服务都有自己的日志文件，排查问题需要连接到服务主机去查看日志，非常不便。
	需要一个办法将所有服务的日志都收集起来统一管理，这就引入了有名的ELK框架。

### 二、环境准备

#### 主机中安装Docker 参照：  https://docs.docker.com/install/linux/docker-ce/centos/#prerequisites  执行如下命令：
	yum install -y yum-utils \
  		device-mapper-persistent-data \
  		lvm2

 	yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

	yum-config-manager --enable docker-ce-edge
	yum install docker-ce

	# 查看docker 版本
	docker --version

	# 启动docker
	systemctl start docker

	#安装docker-compose
	sudo curl -L "https://github.com/docker/compose/releases/download/1.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
	sudo chmod +x /usr/local/bin/docker-compose
	docker-compose --version

### 三、镜像与ELK配置文件相关准备

#### 1、镜像准备，执行命令：

	docker pull elasticsearch:6.4.2
	docker pull logstash:6.4.2
	docker pull kibana:6.4.2
	docker pull mobz/elasticsearch-head:5

	# 查看相关镜像
![](/images/elk/docker-images.png) 

#### 2、ELK配置相关准备:
	# 拉取相关配置以及脚本：
	git clone https://github.com/hu5675/docker-elk.git elk
![](/images/elk/elk-directory.png) 

<font color="#dd0000">后缀.txt的都不需要关心</font>

	修改start.sh\stop.sh 为可执行文件：
	chmod +x start.sh stop.sh

### 四、构建Elasticseach集群、Elasticseach-head、Logstash和Kibana
#### 1、执行start.sh 构建：
	./start.sh
![](/images/elk/elk-pwd.png) 

<font color="#dd0000">注意执行过程中需要输入elastic、kibana、logstash_system、beats_system密码，elastic的密码需要配置在kibana、logstash目录下的配置中</font>
<br>
<font color="#dd0000">注：以上4个框分别输入的密码是：elastic、kibana、logstash、filebeat。也就是es内置4个用户的密码</font>

#### 2、查看运行的容器：
![](/images/elk/elk-container.png) 

#### 3、（此步不做）执行stop.sh 可删除以上构建的容器、网络等。需手动删除es的数据目录文件：
	rm -rf es-cluster/data/es2/*
	rm -rf es-cluster/data/es/*
	rm -rf es-cluster/data/es/*

### 五、验证部署环境

#### 1、利用es-head验证es集群是否正常：
	# 浏览器打开
	http://118.190.165.171:9100/?auth_user=elastic&auth_password=elastic
![](/images/elk/es-head.png) 
<font color="#dd0000">可见es集群正常，注意箭头所指的地址记得修改</font>

#### 2、利用es api接口验证：
	# 浏览器打开
	http://118.190.165.171:9200/_cat/nodes  会要求输入用户名和密码  用户名：elastic  密码：elastic
![](/images/elk/es-api.png) 

#### 3、验证logstash 是否正常：
	# 浏览器输入： 
	http://118.190.165.171:9600/ 
![](/images/elk/logstash.png) 

#### 4、验证kibana是否正常：
	# 浏览器输入：
	http://118.190.165.171:5601  会要求输入用户名和密码 用户名：elastic  密码：elastic
![](/images/elk/kibana.png) 

<font color="#dd0000">到此 es集群、Logstash、es-head以及kibana 搭建完成</font>

### 六、部署Filebeat日志收集组件

#### 1、另外找台主机部署Filebeat，同样从git拉取elk。
	git clone https://github.com/hu5675/docker-elk.git elk 
<font color="#dd0000">需用到elk目录下的filebeat文件件和特定日志格式的日志模板ZHLERROR.log</font>

#### 2、 进入elk/filebeat，并修改目录下的配置文件filebeat.yml，主要配置收集的日志目录和logstash地址。
![](/images/elk/filebeat-yml.png) 

#### 3、部署并启动
	chmod +x deploy_filebeat.sh
	./deploy_filebeat.sh

#### 4、检查下filebeat是否启动成功
![](/images/elk/filebeat-ps.png) 

### 七、测试Filebeat以及在Kibana中查看收集的日志

#### 1、将ZHLERROR.log文件的内容输出到 /root/logs/1.log中
	cat /elk/ZHLERROR.log > /root/logs/1.log

#### 2、查看es-head的索引变化
![](/images/elk/es-head-1.png) 
![](/images/elk/es-head-2.png) 
<font color="#dd0000">可见filebeat已经将日志收集并发送到es集群中</font>

#### 3、也可在kibana中查看日志记录：

![](/images/elk/kibana-data.png) 

Kibana其他高级功能略过
<br>
<font color="#dd0000">至此 ELK日志收集环境搭建完成。</font>
