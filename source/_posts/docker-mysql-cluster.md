title: Docker搭建MySQL集群
date: 2018-11-4 02:36:11
categories: Docker
tags:
- Docker 
- MySql
comments: true
---

#### 零、MySQL集群方案介绍

常见的MySQL集群方案：
1. [Replication](https://www.digitalocean.com/community/tutorials/how-to-set-up-master-slave-replication-in-mysql)
    
    特点：速度快，弱一致性，低价值
    
    应用举例：日志、新闻、帖子

2. [PXC](https://www.percona.com/software/mysql-database/percona-xtradb-cluster)

    特点：速度慢，强一致性，高价值
    
    应用举例：订单、账户、财务
    
本文将讲解PXC文案的使用。

<!-- more -->

#### 一、安装PXC镜像
  
本文MySQL集群文案使用pxc

docker的镜像仓库中包含了pxc数据库的镜像，下载即可：
```bash
docker pull percona/percona-xtradb-cluster
```
也可以从本地导入镜像：
```bash
docker load < /home/soft/pxc.tar.gz
```

下载成功以后，可以看到：
```bash
[root@VM_0_3_centos docker_shared]# docker images
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
docker.io/percona/percona-xtradb-cluster   latest              7ad1b9c338b6        4 weeks ago         413 MB
```

可以看到，这个镜像名字有点长，方便后面使用，我们修改一下：
```bash
docker tag docker.io/percona/percona-xtradb-cluster pxc
```

然后原来名字的镜像就可以删除了：
```bash
docker rmi docker.io/percona/percona-xtradb-cluster
```

#### 二、创建内部网络

出于安全考虑，需要给PXC集群实例创建Docker内部网络

创建网段：
```bash
docker network create --subnet=172.18.0.0/24 net1 
```

查看一下：
```bash
docker inspect net1
```
将看到类似如下信息：
```
[
    {
        "Name": "net1",
        "Id": "3d199b40cf4abfcd6f7f9500396fc9a2a0a860f4809e97c386edc494149f2a07",
        "Created": "2018-11-04T20:59:38.530177174+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/24"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

此网段也可以删除：
```bash
docker network rm net1
```

#### 三、创建Docker卷

一旦docker容器创建完成后，尽量不要在容器里面保存业务数据，要把数据保存在宿主机上，使用的技术就是通过目录映射，可以把宿主机上的一个目录，映射到容器内，在运行容器的时候，把业务数据保存在这个映射目录里面，也就相当于存储在宿主机上面，这样如果出现什么故障的话，只需要把故障的容器删除掉，重新创建一个新的容器，然后把目录映射给新的容器，那么新容器启动后就自带了这些业务数据。

PXC这种技术，运行在docker容器里面，它无法直接使用映射的目录，这里采用另一种目录映射技术：Docker卷。

创建数据卷：
```bash
docker volume create v1
```
查看数据卷到底是创建在宿主机的位置：
```bash
docker inspect v1
```
将看类似输出：
```
[
    {
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/v1/_data",
        "Name": "v1",
        "Options": {},
        "Scope": "local"
    }
]
```
可以看到数据卷在宿主机上的真实位置是：`/var/lib/docker/volumes/v1/_data`

如果想删除数据卷：
```bash
docker volume rm v1
```
查看所有的数据卷：
```bash
docker volume ls
```
移除所有未使用的数据卷：
```bash
docker volume prune
```

#### 四、创建PXC容器

只需要向PXC镜像传入运行参数就能创建出PXC容器：
```bash
docker run -d -p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e CLUSTER_NAME=PXC \
-e XTRABACKUP_PASSWORD=abc123456 \
-v v1:/var/lib/mysql \
--privileged --name=node1 \
--net=net1 --ip 172.18.0.2 pxc
```
参数解释：
* `-d`: 创建出来的容器在后台运行
* `-p`: 端口映射，前面是宿主机端口，后面是容器端口
* `-e`: 启动参数：
  * `MYSQL_ROOT_PASSWORD`：MySQL的Root用户密码
  * `CLUSTER_NAME`：PXC集群的名字
  * `XTRABACKUP_PASSWORD`：数据库节点同步需要的密码
* `-v`: 目录映射，把v1数据卷映射到容器里mysql的数据目录

* `--privilleged`：给权限
* `name`：容器的名字
* `--net`: 使用的内部网段
* `--ip`: 分到网段的IP地址
* `pxc`: 镜像的名字

以上执行会比较耗时，要耐心等待。

如果第一个pxc容器里的mysql没有启动，你就创建了第2个pxc容器，第2个pxc容器启动会闪退，因为第2个里面的mysql和第一个容器里面的mysql是同步的，它发现第1个容器的mysql未启动成功，它就会出故障，导致闪退。

一定要等第一个容器里mysql成功始化，并且通过客户端能连接这个mysql实例了，再去创建第2个、第3个、第4个、第n个实例。

> 注：与该实例建立连接的用户名和密码分别是：**root**和**123456**，比较简单的方式是使用DataGrip

创建第二个pxc容器：
```bash
docker run -d -p 3307:3306 \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e CLUSTER_NAME=PXC \
-e XTRABACKUP_PASSWORD=abc123456 \
-e CLUSTER_JOIN=node1 \
-v v2:/var/lib/mysql \
--privileged --name=node2 \
--net=net1 --ip 172.18.0.3 pxc
```
参数解释（与第一个容器不一样的参数）：
* -p: 端口映射，容器的3306端口映射宿主的3307
* CLUSTER_JOIN：因为这个节点需要加入集群，需要指定和哪个节点进行同步。这里指定跟第一个容器的节点同步。
* -v: 映射的数据卷是v2
* --name: 节点名字
* --ip: 分配的IP地址不一样


按照以上方式，创建五个pxc容器：
```bash
[root@VM_0_3_centos _data]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                   NAMES
d6f2cf84f2b4        pxc                 "/entrypoint.sh "   2 minutes ago       Up 2 minutes        4567-4568/tcp, 0.0.0.0:3310->3306/tcp   node5
4a98ea2eaad8        pxc                 "/entrypoint.sh "   2 minutes ago       Up 2 minutes        4567-4568/tcp, 0.0.0.0:3309->3306/tcp   node4
fed9859cc58a        pxc                 "/entrypoint.sh "   4 minutes ago       Up 3 minutes        4567-4568/tcp, 0.0.0.0:3308->3306/tcp   node3
8663b08d8ec3        pxc                 "/entrypoint.sh "   5 minutes ago       Up 5 minutes        4567-4568/tcp, 0.0.0.0:3307->3306/tcp   node2
68138538a901        pxc                 "/entrypoint.sh "   20 minutes ago      Up 20 minutes       0.0.0.0:3306->3306/tcp, 4567-4568/tcp   node1
[root@VM_0_3_centos _data]# netstat -apnlt | grep docker
tcp6       0      0 :::3307                 :::*                    LISTEN      13263/docker-proxy- 
tcp6       0      0 :::3308                 :::*                    LISTEN      14589/docker-proxy- 
tcp6       0      0 :::3309                 :::*                    LISTEN      15926/docker-proxy- 
tcp6       0      0 :::3310                 :::*                    LISTEN      17230/docker-proxy- 
tcp6       0      0 :::3306                 :::*                    LISTEN      11209/docker-proxy- 
```

##### 4.1 测试PXC集群是否成功：

用DataGrid创建5个MySQL连接，在其中一个数据里面创建schema，再创建一张表，如果有同步到其它数据库，说成PXC集群搭建成功。或者通过命令行。

注：我这里使用最新版的DataGrid测试有问题，如果遇到同样的情况，建议使用命令行测试。

```bash
[root@VM_0_3_centos ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                   NAMES
837a571fbb0b        pxc                 "/entrypoint.sh "   9 seconds ago       Up 8 seconds        4567-4568/tcp, 0.0.0.0:3307->3306/tcp   node2
b7bc29bf17ea        pxc                 "/entrypoint.sh "   2 minutes ago       Up 2 minutes        0.0.0.0:3306->3306/tcp, 4567-4568/tcp   node1
[root@VM_0_3_centos ~]# docker exec -it b7 bash
mysql@b7bc29bf17ea:/$ mysql -h localhost -uroot -pabc123456
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 88
Server version: 5.7.23-23-57 Percona XtraDB Cluster (GPL), Release rel23, Revision f5578f0, WSREP version 31.31, wsrep_31.31

Copyright (c) 2009-2018 Percona LLC and/or its affiliates
Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database test;
Query OK, 1 row affected (0.03 sec)

mysql> use test;
Database changed
mysql> create table ttt(id int auto_increment primary key);
Query OK, 0 rows affected (0.13 sec)

mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ttt            |
+----------------+
1 row in set (0.00 sec)

mysql> insert ttt (id) values (100);
Query OK, 1 row affected (0.01 sec)

mysql> insert ttt (id) values (101);
Query OK, 1 row affected (0.00 sec)

mysql> select * from ttt;
+-----+
| id  |
+-----+
| 100 |
| 101 |
+-----+
2 rows in set (0.00 sec)

mysql> exit
Bye
mysql@b7bc29bf17ea:/$ exit
exit
[root@VM_0_3_centos ~]# docker exec -it 83 bash
mysql@837a571fbb0b:/$ mysql -h localhost -uroot -pabc123456 
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.7.23-23-57 Percona XtraDB Cluster (GPL), Release rel23, Revision f5578f0, WSREP version 31.31, wsrep_31.31

Copyright (c) 2009-2018 Percona LLC and/or its affiliates
Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| test               |
+--------------------+
5 rows in set (0.00 sec)

mysql> use test;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+----------------+
| Tables_in_test |
+----------------+
| ttt            |
+----------------+
1 row in set (0.00 sec)

mysql> select * from ttt;
+-----+
| id  |
+-----+
| 100 |
| 101 |
+-----+
2 rows in set (0.00 sec)
```
以上可以看出，测试成功。

#### 五、使用Haproxy进行负载均衡

拉取Haproxy：
```bash
docker pull haproxy
```

创建Haproxy配置文件，这里直接在宿主机上创建，然后映射 到容器中。
```
touch /root/docker_shared/haproxy.cfg
```
配置文件详情可以参考：https://zhangge.net/5125.html

启动Haproxy容器:
```bash
docker run -it -d \
-p 4001:8888 -p 4002:3306 \
-v /root/docker_shared:/usr/local/etc/haproxy \
--name h1 --privileged --net=net1 haproxy
```
参数解释：

* -p: 端口映射
  * 4002:3306, haproxy对外提供一个数据库负载均衡的服务，映射到宿主机上的4002端口，因为宿主机上的3306端口已经被占用了。
  * 4001:8888，haproxy提供了一个监控页面，在配置文件中定义的8888端口。
* -v: 目录映射
* --name: 起名，命名h1是因为后面可能会配置成双节点。
* --net: 网段，因为haproxy这个负载均衡的实例都是net1网段。

进入到刚才启动的haproxy的容器里：
```bash
docker exec -it h1 bash
```

在容器里启动haproxy（注意，是在容器里执行）:
```bash
root@9e41b6d6d1b9:/# haproxy -f /usr/local/etc/haproxy/haproxy.cfg
```

需要在数据上创建一个haproxy的账号，因为haproxy这个中间件要用这个账号登录数据库，然后发心跳检测。
```sql
CREATE USER 'haproxy'@'%' IDENTIFIED BY '';
```

表示任何IP都可以以haproxy账号登录到数据库，密码为空。

在浏览器上访问：http://ip:4001/dbs 查看监控状态。（用户名和密码配置haproxy.cfg里面）

测试：

在DataGrid上为Haproxy创建连接进行测试：
* `Host`：ip
* `Port`: 4002
* `User`: root
* `Passowrd`: abc123456

通过操作haproxy这个连接操作数据库，测试是否可以通步到所有集群节点。

#### 六、对Haproxy进行高可用配置

##### 6.1 安装Keepalived

Keepalived必须安装在Haproxy所在的容器之内

```bash
docker exec -it h1 bash
```

更新apt-get:
```bash
apt-get update
```
安装keepalived:
```bash
apt-get install keepalived
```

创建Keepalived配置文件：/etc/keepalived/keepalived.conf
```
vrrp_instance  VI_1 {
    state  MASTER
    interface  eth0
    virtual_router_id  51
    priority  100
    advert_int  1
    authentication {
        auth_type  PASS
        auth_pass  123456
    }
    virtual_ipaddress {
        172.18.0.201
    }
}
```
启动Keepalived：
```
service keepalived start
```

启动完成后，宿主机可以Ping通虚拟IP：
```
[root@VM_0_3_centos docker_shared]# ping 172.18.0.201
PING 172.18.0.201 (172.18.0.201) 56(84) bytes of data.
64 bytes from 172.18.0.201: icmp_seq=1 ttl=64 time=0.101 ms
64 bytes from 172.18.0.201: icmp_seq=2 ttl=64 time=0.081 ms
^C
--- 172.18.0.201 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 999ms
rtt min/avg/max/mdev = 0.081/0.091/0.101/0.010 ms
```

#### 七、使用XtraBackup进行数据备份

XtraBackup是一款基于InnoDB的在线热备工具，具有开源免费、支持在线热备、占用磁盘空间小等特点，能够非常快速地备份与恢复mysql数据库。

因为该工具需要安装在数据库所在的容器之内，它备份出来的数据就直接保存在容器里了，这么做tvei好，我们应该把备份出的数据保存在宿主机上，应该采用映射的技术。

所以先要在宿主机上创建一个数据卷，然后映射到某一个数据库的节点上。

(以下主要讲解全量数据的热备份和冷恢复)

创建数据卷：
```
docker volume create backup
```

挑node1这个节点进行操作(先删掉，然后再创建一个新的，因为之前创建的时候对数据有做映射，所以数据不会丢)
```bash
docker stop node1
docker rm node1
```

创建一个新的：
```bash
docker run -d -p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e CLUSTER_NAME=PXC \
-e XTRABACKUP_PASSWORD=abc123456 \
-e CLUSTER_JOIN=node2 \
-v v2:/var/lib/mysql \
-v backup:/data \
--privileged --name=node1 \
--net=net1 --ip 172.18.0.2 pxc
```
参数解释(仅列出与前面不同的)：
* -v backup:/data 映射容器目录
* -e CLUSTER_JOIN=node2 现在创建的节点需要跟什么节点同步(以前是其它节点与我同步的)，现在我停止以后，再创建一个新的node1，要与现在的数据库集群里某一个节点进行同步，这里选择node2，现在有的里随便挑一个就行。

PXC容器中安装XtraBackup，并执行全量备份：
```
apt-get update
apt-get install percona-xtrabackup-24
innobackupex --user=root --password=abc123456 /data/backup/full
```
> 注意，如果进入容器之后不是管理员身份，可以加--user参数指定：docker exec -it --user root node1 bash

执行完成之后，备份的数据在：/var/lib/docker/volumes/backup/_data/backup/full/2018-11-05_01-51-23

PXC全量恢复步骤：

1. 数据库可以热备份，但是不能热还原。为了避免恢复过程中的数据同步，我们采用空白的MySQL还原数据，然后再建立PXC集群。
2. 还原数据前要将未提交的事务回滚，还原数据之后重启MySQL。


操作：
```
docker stop node1 node2 node3
docker rm node1 node2 node3
docker volume rm v1 v2 v3
docker volume create v1
```

创建一个新的：
```
docker run -d -p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=abc123456 \
-e CLUSTER_NAME=PXC \
-e XTRABACKUP_PASSWORD=abc123456 \
-v v1:/var/lib/mysql \
-v backup:/data \
--privileged --name=node1 \
--net=net1 --ip 172.18.0.2 pxc
```
再进入这个容器：
```
docker exec -it node1 bash
```

执行清空MySQL指令，把没有提交的事务回滚：
```
innobackupex --user=root --password=abc123456 --apply-back /data/backup/full/2018-11-05_01-51-23
```

执行冷还原：
```
innobackupex --user=root --password=abc123456 --copy-back /data/backup/full/2018-11-05_01-51-23
```

退出容器，然后重新node1节点：
```
exit
docker stop node1
docker start node1
```

查看node1里面的数据是否有正确恢复。