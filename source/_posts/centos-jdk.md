title: CentOS 7 安装 JDK
date: 2017-12-16 16:23:39
tags:
- Linux
- CentOS
- JDK
---

### 第零步 准备

更新包
```
[root@VM_64_70_centos ~]# yum update
```

检查服务器上是否已安装java
```
[root@VM_64_70_centos ~]# java --version
```

如果有安装旧版本，先移除
```
[root@VM_64_70_centos ~]# yum remove java-1.7.0-openjdk
```

### 第一步 下载

本文以安装[jdk8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)为例。

先下载对应的rpm包：

```
[root@VM_64_70_centos ~]# wget --no-cookies --no-check-certificate --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u151-b12/e758a0de34e24606bca991d704f6dcbf/jdk-8u151-linux-x64.rpm

```

<!-- more -->

### 第二步 安装

```
[root@VM_64_70_centos ~]# rpm -ivh jdk-8u151-linux-x64.rpm 
```

将会安装在`/usr/java/jdk1.8.0_151`目录下

### 第三步 配置环境变量

在`/etc/profile`文件后面添加：
```
export JAVA_HOME=/usr/java/default
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```

使环境变量生效：
```
[root@VM_64_70_centos ~]# source /etc/profile
```

### 第四步 验证

```
[root@VM_64_70_centos ~]# java -version
java version "1.8.0_151"
Java(TM) SE Runtime Environment (build 1.8.0_151-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.151-b12, mixed mode)

```