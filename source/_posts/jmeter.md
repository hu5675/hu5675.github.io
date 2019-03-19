---
title: Jmeter 对api接口进行分布式压测
date: 2019-2-19 10:01:25
---

### 一、需求缘起

	公司要求对一级api接口进行压测，目标：1万qps

### 二、压测部署图

![](/images/jmeter/1.png) 

### 二、硬件准备(Linux 4核8g)

	Jmeter Master : jmeter_master_ip

	Jmeter Slave 01: jmeter_slave_ip_01

	Jmeter Slave 02: jmeter_slave_ip_02

	Jmeter Slave 03: jmeter_slave_ip_03

	Nginx : nginx_ip

	Server 01 : server_ip_01

	Server 02 : server_ip_02

	Server 03 : server_ip_03

### 三、Nginx负载搭建（略）

### 四、Jmeter安装

	直接官网下载解压在某个目录即可

### 五、Jmeter Slave 01\02\03 配置，修改jmeter目录下bin/jmeter.properties文件

	(1)server_port=1099

	(2)server.rmi.port=1099

	(3)server.rmi.ssl.disable=true

	(4)启动:  ./bin/jmeter-server 

### 六、Jmeter Master 配置,修改jmeter目录下bin/jmeter.properties文件

	(1)remote_hosts=jmeter_slave_ip_01:1099,jmeter_slave_ip_01:1099,jmeter_slave_ip_01:1099

	(2)server.rmi.ssl.disable=true

### 七、制作jmx脚本

	在window下安装并启动jmeter可视化界面，填写api接口信息，保存为getcnunitlist.jmx


![](/images/jmeter/2.png) 

### 八、上传getcnunitlist.jmx脚本文件到jmeter-master主机。

### 九、在jmeter-master上执行  jmeter -n -r -t getcnunitlist.jmx -l test.jtl 进行压测

### 十、观察测试结果并调整参数，如下调整脚本中的线程数和循环数

![](/images/jmeter/3.png) 

### 十一、在压测机和服务器都正常的情况下反复进行测试，当压测机jmeter-slave有压力时可继续增加压测机，当服务器有压力时可继续增加服务器，预估增加服务器带来的qps上升量。

