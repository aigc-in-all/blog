title: CentOS 7 安装 MySQL
date: 2017-12-16 15:29:41
tags:
- Linux
- CentOS
- MySQL
---

MySQL是一个开源的数据库管理系统，通常作为流行的LEMP(Linux，Nginx，MySQL/MariaDB，PHP/Python/Perl)堆栈安装，它使用关系数据库和SQL结构化语言。

如果你在CentOS 7上运行`yum install mysql`，那就是安装了`MariaDB`，而不是MySQL。如果你想了解`MySQL`和`MariaDB`对比，`MariaDB`通常可以无缝地代替`MySQL`。具体差异可参考其它相关文档，这篇文章只介绍在CentOS 7系统下安装`MySQL`。

### 预备知识

要开始这篇文章，你需要有以下预备知识：

* 一个CentOS 7的系统，并且具有安装软件的权限
* 基本的Linux命令

### 第一步 安装MySQL

前面提到过，默认情况下使用yum将安装MariaDB，要安装MySQL，我们需要访问MySQL社区提供的Yum资源包。

在浏览器上访问：

```
https://dev.mysql.com/downloads/repo/yum/
```

下载：
```
[root@VM_64_70_centos ~]# wget https://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm
```
下载完成后，可以与网站上对比一下`md5sum`，保证下载文件的完整性。

安装这个包：
```
[root@VM_64_70_centos ~]# rpm -ivh mysql57-community-release-el7-9.noarch.rpm 
```
这将添加两个新的MySQL源，接下来就可以使用它去安装MySQL服务了。

```
[root@VM_64_70_centos ~]# yum install -y mysql-server
```

<!-- more -->

### 第二步 开启MySQL

使用下面的命令启动守护进程：
```
[root@VM_64_70_centos ~]# systemctl start mysqld
```
如果没有任何输出就表示启动成功了。可以使用下面命令检查运行状态：
```
[root@VM_64_70_centos ~]# systemctl status mysqld
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Sat 2017-12-16 14:31:26 CST; 15s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 9334 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 9258 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 9339 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─9339 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

Dec 16 14:31:15 VM_64_70_centos systemd[1]: Starting MySQL Server...
Dec 16 14:31:26 VM_64_70_centos systemd[1]: Started MySQL Server.
```
注：MySQL安装后会自动开机启动，可以执行`systemctl disable mysqld`改变这种默认行为。

安装完成后，会为root用户生成一个默认的密码：
```
[root@VM_64_70_centos ~]# grep 'temporary password' /var/log/mysqld.log
2017-12-16T06:31:17.089898Z 1 [Note] A temporary password is generated for root@localhost: Ktp#)8eg4fp!
```
这个密码在下一步的安装配置中将会用到，并且会让你强制修改。

### 第三步 配置MySQL

MySQL包含一个脚本，可以简单配置一些不太安全的默认选项，比如远程root登录。

```
[root@VM_64_70_centos ~]# mysql_secure_installation 
```
将会提示你输入root用户的密码，也就是前面自动生成的默认密码。然后会需要修改密码。

注：默认的密码安全策略：至少包含一个大写字母，一个小写字母，一个数字，一个特殊字符。

然后可能会有一些提示问你是否要删除匿名用户、是否开启远程登录，删除测试数据库等，建议这些都关掉，后面需要的时候手动配置。

Ok，到这里就已经安装完了，并且一些基本的配置也搞定了。

### 第四步 测试MySQL

```
[root@VM_64_70_centos ~]# mysqladmin -u root -p version
```
输入root用户密码后，将看到类似这样的输出：
```
mysqladmin  Ver 8.42 Distrib 5.7.20, for Linux on x86_64
Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Server version		5.7.20
Protocol version	10
Connection		Localhost via UNIX socket
UNIX socket		/var/lib/mysql/mysql.sock
Uptime:			5 min 50 sec

Threads: 1  Questions: 17  Slow queries: 0  Opens: 113  Flush tables: 1  Open tables: 106  Queries per second avg: 0.048
```
表明已经安装成功。

### 第五步 开启远程连接

如果在第三步执行配置脚本的时候后面选择了不开启远程登录，那么需要手动配置开启远程登录。

编译MySQL配置文件：
```
[root@VM_64_70_centos ~]# vim /etc/my.cnf
```
把下面的配置粘贴到`[mysqld]`块的最下面
```
bind-address = *
```
重启MySQL
```
[root@VM_64_70_centos ~]# systemctl restart mysqld
```
下面为远程连接登录一个新用户，并且授予所有的特权：
```
mysql> create user 'username'@'%' identified by 'password';
mysql> grant all privileges on *.* to 'username'@'%' identified by 'password';
mysql> flush privileges;
```

```
mysql> select user,host from user;
+---------------+-----------+
| user          | host      |
+---------------+-----------+
| heqingbao     | %         |
| mysql.session | localhost |
| mysql.sys     | localhost |
| root          | localhost |
+---------------+-----------+
4 rows in set (0.00 sec)
```
可以看到对应的账户已经创建成功。

测试一下：
```
➜  ~ mysql -h xxx.xxx.xxx.xxx -P 3306 -u heqingbao -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 17
Server version: 5.7.20 MySQL Community Server (GPL)

Copyright (c) 2000, 2017, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```
Ok，可以看到已经远程登录成功了。

