title: CentOS安装nginx
date: 2017-12-16 16:23:52
tags:
- Linux
- CentOS
- Nginx
---

本文使用编译源码的方式安装

### 第零步 安装依赖

```
[root@VM_64_70_centos ~]# yum install -y gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

### 第一步 下载

```
[root@VM_64_70_centos ~]# wget https://nginx.org/download/nginx-1.12.2.tar.gz
```

### 第二步 安装

解压：

```
[root@VM_64_70_centos ~]# tar -zxvf nginx-1.12.2.tar.gz
```

```
[root@VM_64_70_centos ~]# cd nginx-1.12.2/
[root@VM_64_70_centos nginx-1.12.2]# ./configure \
> --prefix=/usr/local/nginx \
> --pid-path=/var/run/nginx/nginx.pid \
> --lock-path=/var/lock/nginx.lock \
> --error-log-path=/var/log/nginx/error.log \
> --http-log-path=/var/log/nginx/access.log \
> --with-http_gzip_static_module \
> --http-client-body-temp-path=/var/temp/nginx/client \
> --http-proxy-temp-path=/var/temp/nginx/proxy \
> --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
> --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
> --http-scgi-temp-path=/var/temp/nginx/scgi
```

上边将临时文件目录指定为/var/temp/nginx，需要在/var下创建temp及nginx目录：

```
[root@VM_64_70_centos nginx-1.12.2]# mkdir -p /var/temp/nginx
```

编译安装：
```
[root@VM_64_70_centos nginx-1.12.2]# make && make install
```

启动：
```
[root@VM_64_70_centos nginx-1.12.2]# cd /usr/local/nginx/
[root@VM_64_70_centos nginx]# ./sbin/nginx 
```

<!-- more -->

### 第三步 验证

查询nginx进程：
```
[root@VM_64_70_centos nginx]# ps aux | grep nginx
root     16677  0.0  0.0  20008   648 ?        Ss   12:29   0:00 nginx: master process ./sbin/nginx
nobody   16678  0.0  0.0  20452  1556 ?        S    12:29   0:00 nginx: worker process
root     17090  0.0  0.0   6444   696 pts/0    R+   12:32   0:00 grep nginx
```
16677是nginx主进程的进程id，16678是nginx工作进程的进程id

访问：[http://localhost/](http://localhost/)

### 第四步 配置

#### 4.1 停止

* 方式一：快速停止

```
[root@VM_64_70_centos nginx]# ./sbin/nginx -s stop
```
此方式相当于先查出nginx进程id再使用kill命令强制杀死进程

* 方式二：完整停止
```
[root@VM_64_70_centos nginx]# ./sbin/nginx -s quit
```
此方式会待nginx进程处理任务完毕后停止，推荐使用。

#### 4.2 重启

* 方式一：先停止再启动
```
[root@VM_64_70_centos nginx]# ./sbin/nginx -s quit
[root@VM_64_70_centos nginx]# ./sbin/nginx
```

* 方式二：重新加载配置文件 当nginx的配置文件nginx-conf修改后，要想让配置生效需要重启nginx，使用-s reload不用先停止再启动即可将配置信息生效：
```
[root@VM_64_70_centos nginx]# ./sbin/nginx -s reload
```

#### 4.3 配置虚拟主机

虚拟主机是一种特殊的软硬件技术，它可以将网络上的每一台计算机分成多个虚拟主机，每个虚拟主机可以独立对外提供www服务，这样就可以实现一台主机对外提供多个web服务，每个虚拟主机之间是独立的，互不影响。

Nginx支持三种类型的虚拟主机配置：

* 基于ip的虚拟主机
* 基于域名的虚拟主机
* 基于端口的虚拟主机
