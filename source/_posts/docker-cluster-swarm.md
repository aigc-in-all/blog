title: Docker集群之Swarm
date: 2018-11-5 01:11:23
categories: Docker
tags:
- Docker 
comments: true
---

### 介绍

Docker三剑客：
* docker-machine # 提供容器服务
* docker-compose # 提供脚本执行服务
* docker-swarm # 提供容器的集群技术

docker-swarm集群节点分为manager和worker节点，manager节点是管理swarm集群的，worker是工作节点，是用来运行容器部署项目的。manager节点除了管理集群之外，其自身也会承担work节点的工作。

无论是manager还是work节点，都可以配置多节点。如果一个manger节点挂掉，其它的manager节点就会选举一个新的manager节点来管理swarm集群。worker节点也是一样。

<!-- more -->

### 创建swarm集群

```bash
docker swarm init
```
可加参数：
* `--listen-addr ip:port` # 管理者节点
* `--advertise-addr ip` # 广播地址

### 加入swarm集群

添加manager或者work节点到集群，只需要执行对应的命令即可：
```bash
docker swarm join-token manager
docker swarm join-token worker
```

### 查看Swarm集群节点

只可以在Manager节点执行该命令：
```bash
docker node ls
```

### 查看Swarm集群网络

docker集群创建完成后，它自带了一个共享的网络。

查看网络信息：
```bash
docker network ls
```
会看找到一个ingress的共享网络，它的类型是overlay，归属于swarm集群。

ingress网络不是用来作容器和容器之间业务通信的，它是用来管理swarm集群的。我们需要创建新的共享网络。

### 创建共享网络

```bash
docker network create -d overlay --attachable swarm_test
```

这样创建的共享网络就可以作为容器间进行业务通信了。

### 创建容器

如果在创建容器的时候，如果想使用共享网络，只需要加上`--net=swarm_test`即可：

```bash
docker run -it --net=swarm_test ...
```

举例：在主机1里面创建docker容器的时候使用swarm_test共享网络，在主机2里面也创建docker容器的时候也使用swarm_test共享网络，那么两个主机的docker容器就可以利用这个共享网络实现业务的通讯。

### 退出Swarm集群

主动退出集群：

```bash
docker swarm leave --forece
```
manager退出集群必须要使用--force参数

被动退出集群：

如果是manager节点想要退出集群，需要在其它节点里面把这个manager节点进行降级成worker节点：
```bash
docker node demote 节点名字
```
删除：
```bash
docker node rm 节点名字
```
删除任何的节点必须要先停止它的Docker服务，Manager节点必须先降级成Worker节点，再删除。