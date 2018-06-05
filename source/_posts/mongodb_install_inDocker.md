---
title: Docker中手动安装Mongodb，不使用Docker的MongoDB镜像。
date: 2018-03-16 10:01:25
---

### 一、环境准备（使用远程主机安装）。
   #### 1、Docker 的安装。(参考官方文档：https://docs.docker.com/engine/installation/)
   #### 2、Docker安装ubuntu镜像。
    docker pull registry.docker-cn.com/library/ubuntu
### 二、安装MongoDB。
#### 1、使用SecureCRT远程连接主机。
#### 2、查看docker 版本：
        root@gitserver-All-Series:~# docker --version
        Docker version 1.9.1, build a34a1d5
#### 3、查看docker镜像：
        root@gitserver-All-Series:~# docker images
        REPOSITORY                              			TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
        registry.docker-cn.com/library/ubuntu  	            latest              8396575af44c        5 months ago        122 MB
#### 4、启动ubuntu并进入：
     	root@gitserver-All-Series:~# docker run -it --name ubuntu_test registry.docker-cn.com/library/ubuntu:latest
        root@71ed819cb3a4:/# 
#### 5、查看ubuntu版本：
 	    root@71ed819cb3a4:/# cat /etc/issue
        Ubuntu 16.04.3 LTS \n \l
#### 6、安装wget：
        root@71ed819cb3a4:/# apt-get update
        root@71ed819cb3a4:/# apt-get install wget  
        root@71ed819cb3a4:/# wget --version
        GNU Wget 1.17.1 built on linux-gnu.
#### 7、使用wget下载MongoDB，并解压：
        root@71ed819cb3a4:/home# wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.6.3.tgz      (如果网络问题下载不下来-在本地下载上传到远程主机，再由远程主机复制到docker容器的Ubuntu中)
        root@71ed819cb3a4:/home# tar -zxvf mongodb-linux-x86_64-ubuntu1604-3.6.3.tgz 
        root@71ed819cb3a4:/home# ll
#### 8、移动解压的文件夹到/usr/local/mongodb中：
        root@71ed819cb3a4:/home# mv mongodb-linux-x86_64-ubuntu1604-3.6.3 /usr/local/mongodb
#### 9、配置环境变量：
#####   安装vim编辑器：
  	    root@71ed819cb3a4:/home# apt-get install vim
#####   编译~/.bashrc文件：
        root@71ed819cb3a4:/home# vim ~/.bashrc    
#####   在文件末尾加入：
        export PATH=/usr/local/mongodb/bin:$PATH
#####   使刚编辑的内容即可生效：
        root@71ed819cb3a4:/home# source ~/.bashrc 
#### 10、启动mongodb前准备：建立目录（/home/mongodb/config、/home/mongodb/data、/home/mongodb/data/db、/home/mongodb/log ）
        root@71ed819cb3a4:/home# mkdir -p /home/mongodb/config
        root@71ed819cb3a4:/home# mkdir -p /home/mongodb/data  
        root@71ed819cb3a4:/home# mkdir -p /home/mongodb/data/db
        root@71ed819cb3a4:/home# mkdir -p /home/mongodb/log  
#### 配置mongodb.conf配置文件
        root@71ed819cb3a4:/home/mongodb/config# vim /home/mongodb/config/mongodb.conf
#### 在配置文件中配置基本信息并保存：
    dbpath=/home/mongodb/data/db
    logpath=/home/mongodb/log/mongodb.log
    logappend=true
    port=27017
    bind_ip=0.0.0.0
#### 11、启动mongodb:
        root@71ed819cb3a4:/home/mongodb/config# mongod -f /home/mongodb/config/mongodb.conf &
#### 查看是否成功启动：
        root@71ed819cb3a4:/home/mongodb/config# ps -ef
        UID        PID  PPID  C STIME TTY          TIME CMD
        root         1     0  0 02:15 ?        00:00:00 /bin/bash
        root      2806     1  3 02:45 ?        00:00:00 mongod -f /home/mongodb/config/mongodb.conf     
        root      2829     1  0 02:45 ?        00:00:00 ps -ef
可以看到已成功启动。
#### 12、退出当前ubuntu实例，
        root@71ed819cb3a4:/home/mongodb/config# exit
#####   使用docker ps -a 找到刚才的容器id
        root@gitserver-All-Series:~# docker ps -a
        CONTAINER ID        IMAGE                                          COMMAND                  CREATED             STATUS                       PORTS           NAMES
        71ed819cb3a4        registry.docker-cn.com/library/ubuntu:latest   "/bin/bash"              33 minutes ago      Exited (0) 29 seconds ago                 ubuntu_test
#####  提交容器的更改为本地镜像，供以后使用。这样就不需要重新已上步骤再安装一遍。
        root@gitserver-All-Series:~# docker commit -m 'mongodb_test'  71ed819cb3a4 ubuntu:mongodb_test
#####  查看提交的镜像：
        root@gitserver-All-Series:~# docker images
        REPOSITORY                              TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
        ubuntu                                  mongodb_test        214527bb4054        20 seconds ago      926.7 MB

至此mongodb安装完成。