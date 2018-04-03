---
title: Maven + Docker + GitLab + Jenkins 构建镜像并提交Docker私有镜像库， 搭建持续集成环境
---

### 需求缘起

在软件开发过程中尤其在进行到测试阶段时候，开发人员需要打包提交到测试人员 ——> 测试人员发现bug并报告给开发人员 ——> 开发人员fix bug并再次打包提交到测试人员。反复这个流程直到没有bug为止。

### 环境准备

#### 1、本机  java开发环境、Docker、Maven （安装步骤省略）

![](/images/maven+docker/1.png)

#### 2、服务器 200.168.12.69   Docker （安装步骤省略）

![](/images/maven+docker/2.png)

#### 3、本地使用spring boot 搭建一个最简单的测试项目

##### 项目概要
![](/images/maven+docker/3.png) 
![](/images/maven+docker/4.png)
![](/images/maven+docker/5.png)
![](/images/maven+docker/6.png)		

##### 项目就是这么简单，运行并访问 http://localhost:8080/hello 

##### 本地和服务的Docker都启动

### 操作步骤

#### 1、手动构建maven项目并构建docker镜像。（本机操作）

##### 构建过程会用到 docker-maven-plugin 插件，在pom.xml文件中添加插件

	<?xml version="1.0" encoding="UTF-8"?>
	<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>house-parent</artifactId>
        <groupId>com.mooc.house</groupId>
        <version>1.0.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>house-web</artifactId>
    <dependencies>
        <dependency>
            <groupId>com.mooc.house</groupId>
            <artifactId>house-biz</artifactId>
            <version>1.0.0-SNAPSHOT</version>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-test</artifactId>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.freemarker</groupId>
            <artifactId>freemarker</artifactId>
            <version>2.3.23</version>
        </dependency>


    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>2.4</version>
                <executions>
                    <execution>
                        <id>attach-sources</id>
                        <goals>
                            <goal>jar</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>docker-maven-plugin</artifactId>
                <version>0.4.12</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <imageName>${project.artifactId}:${project.version}</imageName>
                    <baseImage>java</baseImage>
                    <exposes>8090</exposes>
                    <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
                    <resources>
                        <resource>
                            <directory>${project.build.directory}</directory>
                            <include>${project.build.finalName}.jar</include>
                        </resource>
                    </resources>
                </configuration>
            </plugin>
        </plugins>
    </build>

	</project>

##### 打开terminal进入到项目目录，执行

	mvn clean package docker:build

##### 待构建目录执行成功之后，会在项目目录target中出现docker文件夹。

![](/images/maven+docker/7.png)

##### 查看本地docker镜像，执行 
	docker images
![](/images/maven+docker/8.png)	

##### OK 本地镜像构建成功。

#### 2、提交docker镜像到阿里云镜像库或自己搭建的docker registry私有库。

##### 1、提交docker镜像到阿里云镜像库。 （本机操作）

###### a) 登录阿里云创建镜像仓库:https://dev.aliyun.com/，  我的最终创建的仓库地址：registry.cn-hangzhou.aliyuncs.com/bmars/hello

###### b) 在terminal中登录阿里云docker，执行 docker login --username=xxxxxxx registry.cn-hangzhou.aliyuncs.com  需要输入密码
	docker login --username=xxxxxxx registry.cn-hangzhou.aliyuncs.com
![](/images/maven+docker/9.png)	

###### c) 给镜像打tag 执行 docker tag 686dd274e354 registry.cn-hangzhou.aliyuncs.com/bmars/hello:1.0
	docker tag 686dd274e354 registry.cn-hangzhou.aliyuncs.com/bmars/hello:1.0
![](/images/maven+docker/10.png)	

###### d) 将registry.cn-hangzhou.aliyuncs.com/bmars/hello:1.0 镜像上传到阿里云，执行：
	docker push registry.cn-hangzhou.aliyuncs.com/bmars/hello:1.0
![](/images/maven+docker/11.png)

###### e) 在阿里云查看是否提交成功。
![](/images/maven+docker/12.png)

###### f) 在测试服务器去拉取镜像，并运行测试我们的spring boot程序。 使用ssh连接工具连接到测试服务器

	查看之前的镜像：docker images
![](/images/maven+docker/13.png)
	拉取镜像： docker pull registry.cn-hangzhou.aliyuncs.com/bmars/hello:1.0   (如果阿里云建的是私有的需要先登录：docker login --username=xxxxxx registry.cn-hangzhou.aliyuncs.com )
	查看拉取之后的镜像：docker images
![](/images/maven+docker/14.png)

###### g) 启动镜像容器，查看spring boot程序是否正常运行。 
	docker run -it -p 8082:8080 --name hello registry.cn-hangzhou.aliyuncs.com/bmars/hello:1.0  
![](/images/maven+docker/15.png)

在浏览器中访问 http://200.168.12.69:8082/hello  得到想要的结果（为什么是8082，因为我服务器的8080端口被占用了，所有用8082端口映射到容器中的8080端口）。

##### 2、提交docker镜像到自己搭建的docker registry私有库。（本机操作）

###### a) 在服务器上安装docker registry镜像, 并运行镜像容器：
	docker pull registry
	docker images
	docker run -d -p 5000:5000 --name registry 8fe0c76b17fc
![](/images/maven+docker/16.png)

###### b) 在本地terminal中查看 镜像信息
	 mars$ curl -X GET 200.168.12.69:5000/v2/_catalog
	 {"repositories":[]}

###### c) 提交本地镜像到服务器：
	mars$ docker tag d97cb62fd731 200.168.12.69:5000/hello:1.0    #重新打tag
	mars$ docker push 200.168.12.69:5000/hello:1.0  				      #推送到服务器

	The push refers to a repository [200.168.12.69:5000/hello]
	Get https://200.168.12.69:5000/v2/: http: server gave HTTP response to HTTPS client     
	报错：因为本地采用https，服务端采用http。 解决方法在本地修改docker配置文件 daemon.json ,加入：
	
	"insecure-registries" : [
    	"200.168.12.69:500"
  	],

  	重启docker。
	 
	mars$ docker push 200.168.12.69:5000/hello:1.0  	 #重新推送到服务器
	
	daemon.json 配置文件如下图:
![](/images/maven+docker/17.png)

###### d) 在本地terminal中再次查看 镜像信息
	mars$ curl -X GET 200.168.12.69:5000/v2/_catalog
	{"repositories":["hello"]}
	#可以看到服务器镜像仓库信息多了 hello镜像
	mars$ curl -X GET 200.168.12.69:5000/v2/hello/tags/list   #查看hello镜像的版本信息
	{"name":"hello","tags":["1.0"]}

###### e) 在服务器上去测试我们刚提交的镜像
	docker images			#查看已有镜像
	docker pull localhost:5000/hello:1.0    #拉取本地registry中的hello镜像
	docker images			#查看拉取后的镜像
![](/images/maven+docker/18.png)

###### f) 运行hello进行
	docker run -it -p 8082:8080 --name hello localhost:5000/hello:1.0    #运行hello镜像，端口8082映射容器内8080端口
![](/images/maven+docker/19.png)

###### g) 浏览器测试我们的打包进hello镜像的spring boot程序
![](/images/maven+docker/20.png)

###### 到此我们把持续集成的步骤拆分，没个步骤依靠手动去单独部署。还没用到 gitlab + jenkins 。

###### 接下来将搭建依靠  gitlab + jenkins 去自动化

###### Maven + Docker + GitLab + Jenkins 构建镜像并提交Docker私有镜像库， 搭建持续集成环境（续）







