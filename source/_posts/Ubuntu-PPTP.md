title: Ubuntu 14.04使用PPTP配置VPN
date: 2015-03-01 18:21:19
tags: 
- PPTP
- VPN
---
### 第一步 安装PPTP

```
apt-get update
```

```
apt-get install pptpd
```

修改pptpd的配置文件`/etc/pptpd.conf`，将以下两行的注释符`#`去掉：

```
localip 192.168.0.1
remoteip 192.168.0.234-238,192.168.0.245
```

修改`/etc/ppp/chap-secrets`文件，添加认证用户：

```
# Secrets for authentication using CHAP
# client    server  secret          IP address
test1       pptpd   123456          *
test2       *       123456          *
```

### 第二步 添加DNS服务器

修改`/etc/ppp/pptpd-options`文件，添加两行：

```
ms-dns 8.8.8.8
ms-dns 8.8.4.4
```

现在可以开启PPTP守护进程了：

```
service pptpd restart
```

可以使用`netstat -alpn | grep :1723`验证运行状态。

### 第三步 设置IP转发

简单地编辑`/etc/sysctl.conf`文件，添加下面这一行（如果不存在的话）：

```
net.ipv4.ip_forward = 1
```

执行`sysctl -p`使之立即生效。

### 第四步 为iptables创建NAT规则

```
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE && iptables-save
```

然后重启pptpd服务：

```
service pptpd restart
```

---------------

然后应该可以妥妥地翻墙了。