title: CentOS安装Tomcat
date: 2017-12-16 16:52:16
tags:
- Linux
- CnetOS
- Tomcat
---

### 第一步 下载
本文以安装[Tomcat7](https://tomcat.apache.org/download-70.cgi)为例

先下载：

```
[root@VM_64_70_centos ~]# wget http://mirrors.hust.edu.cn/apache/tomcat/tomcat-7/v7.0.82/bin/apache-tomcat-7.0.82.tar.gz
```

### 第二步 安装

#### 2.1 解压：
```
[root@VM_64_70_centos ~]# tar zxvf apache-tomcat-7.0.82.tar.gz 
```

Tomcat直接是一个压缩包，解压后就可以直接使用。这里为了和其它软件统一，把它放到移到`/usr/local`目录下：
```
[root@VM_64_70_centos ~]# mv ~/apache-tomcat-7.0.82 /usr/local/
```

启动：

```
[root@VM_64_70_centos ~]# /usr/local/apache-tomcat-7.0.82/bin/startup.sh
```

访问：

```
http://localhost:8080/
```

停止：

```
[root@VM_64_70_centos ~]# /usr/local/apache-tomcat-7.0.82/bin/shutdown.sh
```

<!-- more -->

### 第三步 配置

#### 3.1 配置角色管理

在`/usr/local/apache-tomcat-7.0.82/conf/tomcat-users.xml`末尾添加：
```
<role rolename="admin-gui"/>
<role rolename="admin-script"/>
<role rolename="manager-gui"/>
<role rolename="manager-script"/>
<user username="username" password="password" roles="admin-gui,admin-script,manager-gui,manager-script"/>
```