title: CentOS安装Redis
date: 2018-03-18 20:17:35
tags:
- Linux
- CentOS
- Tomcat
---

> 本文基于CentOS7和Redis3.2.11版本，其它版本的安装方式可能会有些出处，请自行辨别。

目前Redis刚Release 4.0版本，据说有不少改进，对比了各个版本的使用情况，感觉3.x的最后一个版本（也就是4.0的上一个版本）应该目前是稳定的，并且风险最小的。而且文档上有这么一句话：

> Redis 3.2 is the previous stable release. Does not include all the improvements in Redis 4.0 but is a very battle tested release, probably a good pick for critical applications while 4.0 matures more in the next months. 
See the release notes or download 3.2.11.

所以本文以3.2.11版本为例，记录各个安装步骤。

### 一、下载

```
$ wget http://download.redis.io/releases/redis-3.2.11.tar.gz
```

### 二、解压

```
$ tar -zxvf redis-3.2.11.tar.gz
```

### 三、安装

##### 1. 安装到`/user/local/redis`目录

```
$ cd redis-3.2.11
$ make PREFIX=/usr/local/redis install
```

<!-- more -->

##### 2. 配置

`redis.conf`是Redis的配置文件，位于源码目录里面，拷贝此配置文件到安装目录下：

```
$ cd /usr/local/redis
$ mkdir conf
$ cp ~/redis-3.2.11/redis.conf .
```

此时目录结构应该是这样子的：

```
|-- bin
|   |-- dump.rdb
|   |-- redis-benchmark
|   |-- redis-check-aof
|   |-- redis-check-rdb
|   |-- redis-cli
|   |-- redis-sentinel -> redis-server
|   `-- redis-server
`-- conf
    `-- redis.conf
```
解释一下:

cmd | comments
---|---
redis-server | Redis服务器
redis-cli | Redis命令行客户端
redis-benchmark | Redis性能测试工具
redis-check-aof | AOF文件修复工具
redis-check-rdb | RBD文件修复工具
redis-sentinel | Redis集群管理工具

### 四、启动

#### 1. 前端模式启动

直接运行`bin/redis-server`将以前端模式启动，之所以叫前端模式，就是启动后会占用命令行窗口，如果窗口关闭或结束进程，redis-server也会结束，主要用于实时方便测试。

```
$ ./redis-server 
26093:C 17 Mar 22:45:12.374 # Warning: no config file specified, using the default config. In order to specify a config file use ./redis-server /path/to/redis.conf
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 3.2.11 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 26093
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

26093:M 17 Mar 22:45:12.375 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
26093:M 17 Mar 22:45:12.375 # Server started, Redis version 3.2.11
26093:M 17 Mar 22:45:12.375 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
26093:M 17 Mar 22:45:12.375 * The server is now ready to accept connections on port 6379
```

新开一个终端窗口测试一下：

```
$ ./redis-cli 
127.0.0.1:6379> ping
PONG
```

#### 2. 后端模式启动

后端模式启动需要修改配置文件(`redis.conf`)，把此文件里面的`daemonize`改成`yes`就可以了，默认是`no`:

```
# By default Redis does not run as a daemon. Use 'yes' if you need it.
# Note that Redis will write a pid file in /var/run/redis.pid when daemonized.
daemonize yes
```

启动的时候指定使用我们的配置文件：

```
$ ./redis-server ../conf/redis.conf 
$ 
```

可以看到，启动完成后不会占用终端窗口。

测试一下：
```
$ ./redis-cli 
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> set testkey hello
OK
127.0.0.1:6379> get testkey
"hello"
127.0.0.1:6379> 
```

Redis默认使用`6379`端口，可以更改配置文件(`redis.conf`)修改此端口号：

```
# Accept connections on the specified port, default is 6379 (IANA #815344).
# If port 0 is specified Redis will not listen on a TCP socket.
port 6379
```

### 五、连接和关闭

#### 1.连接

可以通过`/user/local/bin/redis-cli`命令来连接本地或远程的Redis。

前面已经测试过连接本地的Redis服务，下面讲一下连接远程的。

默认情况下，Redis不允许远程连接，需要修改配置文件(`redis.conf`)：

 * 注释掉`bind 127.0.0.1`
 * 修改`protected-mode no`

```
$ redis-cli -h x.x.x.x -p 6379
```

#### 2.设置连接密码

默认情况下连接redis是不需要密码的。如果打开可远程连接，建议设置密码提高安全性。设置密码也需要修改配置文件(`redis.conf`)：

打开`requirepass`的注释，并且设置你自己的密码：
```
requirepass mypassword
```

之后在远程连接的时候就需要加上密码参数：

```
$ redis-cli -h x.x.x.x -p 6379 -a mypassword
```

#### 3.关闭

在客户端执行：
```
x.x.x.x:6379> shutdown
```