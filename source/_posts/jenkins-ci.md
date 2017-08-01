---
title: Jenkins 持续集成工具使用介绍
---

#### 需求缘起

在软件开发过程中尤其在进行到测试阶段时候，开发人员需要打包提交到测试人员 ——> 测试人员发现bug并报告给开发人员 ——> 开发人员fix bug并再次打包提交到测试人员。反复这个流程直到没有bug为止。

#### 传统打包方法-1 （全手动）

1、本地编译。

2、本地导出war。

3、上传war到服务器。

4、重启应用服务。

#### 传统打包方法-2 （半自动）

1、本地提交最新代码到版本管理服务器。

2、ssh登录远程服务器。

3、在远程服务器拉取最新代码。

4、使用mvn、sh脚本等构建war。

5、重启应用服务。

##### 传统打包方法缺点

1、每次需要手动操作。

2、每次需要浪费时间（根据业务复杂度耗时不同）。特别在测试阶段这种浪费时间的打包流程会很频繁。

##### 既然传统方案会浪费大量的非业务相关的开发时间，那有没有其他方案能减少这种浪费？

##### 持续集成工具：Jenkins

### 什么是Jenkins？

Jenkins是一个可扩展的持续集成引擎

软件构建发布自动化

自动化测试

支持构建IOS、Android、Java等平台。

运行原理如下图：

![运行图](/images/jenkins/jenkins-1.png)

### Jenkins的安装 （Mac为例）

1、首先安装Java JDK。

   [Java JDK下载](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)	
   在命令行中输入：java -version 查看jdk信息
   java version "1.8.0_121"
   Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
   Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)

2、安装Tomcat。

   [Tomcat 下载](http://tomcat.apache.org/download-80.cgi)
   根据各自情况下载tomcat，并解压到某一目录。

   命令行进入tomcat/bin 目录下，修改所有sh文件为可执行 ： chmod +x *.sh
   执行startup.sh文件启动tomcat: ./startup.sh
   在浏览器中输入localhost:8080 出现tomcat界面说明安装成功。   

3、java maven项目需要要安装maven，IOS项目需要Mac电脑并安装XCode。
   
   [Maven下载](http://maven.apache.org/download.cgi)
   根据各自情况下载tMaven，并解压到某一目录。
   配置maven环境变量
   编译 ~/.bash_profile 文件添加2行并保存。

   export M2_HOME=/Users/mars/Documents/java/apache-maven-3.3.9
   export PATH=$PATH:$M2_HOME/bin

   在cmd中输入 mvn -v 
   Maven home: /Users/mars/Documents/java/apache-maven-3.3.9
   Java version: 1.8.0_121, vendor: Oracle Corporation
   Java home: /Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home/jre
   Default locale: zh_CN, platform encoding: UTF-8
   OS name: "mac os x", version: "10.12.5", arch: "x86_64", family: "mac"
   说明安装成功

4、下载Jenkins.war 放到tomcat webapps目录下。

   [Jenkins下载](https://jenkins.io/download/)
   下载Jenkins.war 放到tomcat/webapps目录下。
   重启tomcat。

   浏览器中输入：http://localhost:8080/jenkins  出现如下界面说明Jenkins安装成功。
   ![运行图](/images/jenkins/jenkins-2.png)

### Jenkins使用
<br/>
1、浏览器输入http://localhost:8080/jenkins  并输入初始密码（/Users/mars/.jenkins/secrets/initialAdminPassword 文件中）进入如下界面：
   ![运行图](/images/jenkins/jenkins-3.png)
<br/>
2、点击第一个选择，安装Jenkins建议安装的插件(如下图):
   ![运行图](/images/jenkins/jenkins-4.png)  
<br/>
3、插件安装成功后创建换一个管理员密码，这样避免每次都输入初始密码，也可以跳过此步骤进入主界面之后再行创建管理员账号（如下图):
   ![运行图](/images/jenkins/jenkins-5.png)  
<br/>
4、在此，先行创建管理员账号之后进入主界面（如下图）：
   ![运行图](/images/jenkins/jenkins-6.png)
   ![运行图](/images/jenkins/jenkins-7.png)





