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
<br/>
5、设置Jenkins的工作目录：系统管理——> 系统设置——> 主目录展开：
   ![运行图](/images/jenkins/jenkins-8.png)
<br/>
6、其他配置暂不管，现在创建一个IOS的工作空间：My Views ——> 创建一个新任务
   ![运行图](/images/jenkins/jenkins-9.png)
<br/>
7、输入项目名、构建一个自由风格项目，点击OK。
   ![运行图](/images/jenkins/jenkins-10.png)
<br/>
8、配置源码管理地址。
   a) 添加credential
   ![运行图](/images/jenkins/jenkins-11.png)
   b) 配置代码服务器地址
   ![运行图](/images/jenkins/jenkins-12.png)

<br/>
9、点击保存，回到面板点击立即构建先把服务器代码签到本地。
   ![运行图](/images/jenkins/jenkins-13.png)
<br/>
10、打开刚签下来的本地代码，确保可以编译可成功。
<br/>
11、回到Jenkins配置项目的构建脚本。
   ![运行图](/images/jenkins/jenkins-14.png)
   备：
   archive_upload.sh 内容如下:
   xcodebuild -workspace Zoharo_swift.xcworkspace -scheme Zoharo_swift -configuration Debug  clean archive -archivePath ZoharoArchive DEVELOPMENT_TEAM=H4FGQGLX69

   ./xcbuild-safe.sh -exportArchive -archivePath ZoharoArchive.xcarchive -exportOptionsPlist exportOptions.plist -exportPath ZoharoIPA

   #蒲公英上的User Key
   uKey="78024b5845de2957c313725e00d1e146"
   #蒲公英上的API Key
   apiKey="7552f5ff809ea22c5452e22da87eea78"
   #要上传的ipa文件路径
   IPA_PATH="ZoharoIPA/Zoharo_swift.ipa"

   #执行上传至蒲公英的命令
   echo "++++++++++++++upload+++++++++++++"
   curl -F "file=@${IPA_PATH}" -F "uKey=${uKey}" -F "_api_key=${apiKey}" http://www.pgyer.com/apiv1/app/upload
<br/>
   archive_upload 文件会用到xcbuild-safe.sh 、 exportOptions.plist 两个文件
   xcbuild-safe.sh内容如下：

   which rvm > /dev/null

   if [[ $? -eq 0 ]]; then
      echo "RVM detected, forcing to use system ruby"
      [[ -s "$HOME/.rvm/scripts/rvm" ]] && source "$HOME/.rvm/scripts/rvm"
      rvm use system
   fi

   if which rbenv > /dev/null; then
      echo "rbenv detected, removing env variables"
      rbenv shell system
   fi

   unset RUBYLIB
   unset RUBYOPT
   unset BUNDLE_BIN_PATH
   unset _ORIGINAL_GEM_PATH
   unset BUNDLE_GEMFILE

   unset GEM_HOME
   unset GEM_PATH

   set -x          # echoes commands
   xcodebuild "$@" # calls xcodebuild with all the arguments passed to this
<br/>
   exportOptions.plist内容如下：
   ![运行图](/images/jenkins/jenkins-15.png)
<br/>
12、回到面板，点击立即构建，确保构建脚本能构建成功。可以查看构建日志拍错。
   ![运行图](/images/jenkins/jenkins-16.png)
   点击正在构建的任务——>Console Output查看构建日志
   ![运行图](/images/jenkins/jenkins-17.png)
<br/>
13、等待构建成功,可以看到打包成功、导出成功、上传蒲同英成功
   ![运行图](/images/jenkins/jenkins-18.png)
<br/>
14、以上都需要手动点击立即构建才执行构建，有没有什么办法能不需要登录Jenkins，不点击立即构建就能让其自动构建。（需要安装gitlab插件）
   利用gitlab hook设置可在代码提交时自动触发Jenkins构建。
   首先先在Jenkins的构建触发器里设置一个构建token。
   ![运行图](/images/jenkins/jenkins-19.png)
   其次登录git服务器点击项目设置里的Webhooks
   ![运行图](/images/jenkins/jenkins-20.png)
   配置url点击add hook
   ![运行图](/images/jenkins/jenkins-21.png)
   点击Test 测试是否配置成功。
   ![运行图](/images/jenkins/jenkins-22.png)

15、到现在为此，所有设置已完成。可在本地编辑代码提交到git服务器，测试下是否能自动触发jenkins构建。

总结:现实开发中可能会很频繁的提交代码，就会很频繁的触发构建。这样会造成大量的流量浪费和服务器资源。 
   解决方法：单独创建一个构建分支代码，待开发到一定程度需要构建时，本地把正在开发的分支合并到构建分支中提交以触发构建分支的代码构建。









