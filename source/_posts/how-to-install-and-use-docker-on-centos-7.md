---
layout: title
title: 怎样在CentOS 7上安装Docker
date: 2018-11-17 21:32:10
tags: 
- CentOS
- Docker
---

### 一、安装Docker

在CentOS 7官方源里面的Docker安装包可能不是最新版本，我们可以从Docker工厂安装最新最好的版本，本节将介绍如何做到这一点。

首先，更新包数据库：
```shell
$ sudo yum check-update
```

现在运行如下命令，它将添加官方的Docker工厂，下载并安装最新版本的Docker：
```shell
$ curl -fsSL https://get.docker.com/ | sh
```

安装完成后，启动Docker守护进程：
```shell
$ sudo systemctl start docker
```

验证运行：
```shell
$ sudo systemctl status docker
```

应该输出类似下面的内容，显示服务活跃并运行：
```text
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since 六 2018-11-17 17:36:56 CST; 33s ago
     Docs: https://docs.docker.com
 Main PID: 20949 (dockerd)
```

最后，确保每次服务器重启后自动开启：
```shell
$ sudo systemctl enable docker
```

现在不仅提供Docker服务（守护进程），而且也提供了`docker`命令行工具，或者叫Docker客户端。后面在探讨如果使用`docker`命令行工具。

<!-- more -->

### 二、不使用Sudo执行Docker命令（可选）

默认情况下，`docker`命令需要root权限——也就是必须加前缀`sudo`，它也可以被`docker`组里面的用户运行，`docker`组是在安装Docker的时候自动创建的。如果你试图不加`sudo`运行`docker`命令（或者不在`docker`组里的用户直接运行），你将看到类似如下的输出：     
```shell
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get http://%2Fvar%2Frun%2Fdocker.sock/v1.39/version: dial unix /var/run/docker.sock: connect: permission denied
```

如果你想避免每次执行docker命令都使用`sudo`，可以把你的用户名加入到docker组：
```shell
$ sudo usermod -aG docker $(whoami)
```

你需要退出再重新登录才能生效。

如果你需要将其它用户添加到docker组，可以使用：
```shell
$ sudo usermod -aG docker username
```

### 三、安装Docker Compose

#### 3.1 首先安装`python-pip`:
```shell
sudo yum install -y epel-release
sudo yum install -y python-pip
```

#### 3.2 安装Docker Compose：
```shell
$ sudo pip install docker-compose
```

你还需要升级你的Python包才能让`docker-compose`成功运行：
```shell
$ sudo yum upgrade python*
```

#### 3.3 使用Docker Compose运行容器

公共的Docker Hub仓库包含一个简单的*Hello World*镜像，现在让我们来测试它。

首先，为我们的YAML文件创建一个目录：
```shell
$ mkdir hello-world
```

进入目录：
```shell
$ cd hello-world
```

创建一个YAML文件：
```shell
$ vim docker-compose.yml
```

把下面的内容粘贴到文件：
```text
my-test:
  image: hello-world
```
第一行将被用作容器名称的一部分，第二行指定使用哪个镜像创建容器，这个镜像将从Docker Hub官方仓库中下载。

同时还在`~/hello-world`目录下执行下面命令创建容器：
```shell
$ docker-compose up
```

应该会输出类似如下内容：
```text
Pulling my-test (hello-world:)...
latest: Pulling from library/hello-world
d1725b59e92d: Pull complete
Creating hello-world_my-test_1_4da1f671c521 ... done
Attaching to hello-world_my-test_1_a07f0d2848f1
my-test_1_a07f0d2848f1 | 
my-test_1_a07f0d2848f1 | Hello from Docker!
my-test_1_a07f0d2848f1 | This message shows that your installation appears to be working correctly.
my-test_1_a07f0d2848f1 | 
my-test_1_a07f0d2848f1 | To generate this message, Docker took the following steps:
my-test_1_a07f0d2848f1 |  1. The Docker client contacted the Docker daemon.
my-test_1_a07f0d2848f1 |  2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
my-test_1_a07f0d2848f1 |     (amd64)
my-test_1_a07f0d2848f1 |  3. The Docker daemon created a new container from that image which runs the
my-test_1_a07f0d2848f1 |     executable that produces the output you are currently reading.
my-test_1_a07f0d2848f1 |  4. The Docker daemon streamed that output to the Docker client, which sent it
my-test_1_a07f0d2848f1 |     to your terminal.
my-test_1_a07f0d2848f1 | 
my-test_1_a07f0d2848f1 | To try something more ambitious, you can run an Ubuntu container with:
my-test_1_a07f0d2848f1 |  $ docker run -it ubuntu bash
my-test_1_a07f0d2848f1 | 
my-test_1_a07f0d2848f1 | Share images, automate workflows, and more with a free Docker ID:
my-test_1_a07f0d2848f1 |  https://hub.docker.com/
my-test_1_a07f0d2848f1 | 
my-test_1_a07f0d2848f1 | For more examples and ideas, visit:
my-test_1_a07f0d2848f1 |  https://docs.docker.com/get-started/
my-test_1_a07f0d2848f1 | 
hello-world_my-test_1_a07f0d2848f1 exited with code 0
```

输出内容表示Docker在做什么：

1. Docker客户端连接Docker守护进程
2. Docker守护进程从Docker Hub拉取`hello-world`镜像
3. Docker守护进程从镜像创建了一个容器，该容器运行生生你当前正在读取的输出可执行文件。
4. Docker守护进程将该输出流式传输到Docker客户端，后者将其发送到你的终端

如果该过程没有自动退出，请按`CTRL-C`。

