---
layout: title
title: CentOS 7 服务器初始化
date: 2018-11-17 16:58:37
tags: CentOS
---

#### 介绍

首先创建一个新服务器时,有一些配置步骤,你应该在早期的基本设置。这将增加服务器的安全性和可用性,为后续的行为做一个坚实的基础。

<!-- more -->

### 一、Root登录

要登录到服务器，首先你需要知道服务器的公有IP地址和Root密码。

```shell
ssh root@SERVER_IP_ADDRESS
```

下一步是建立一个替代用户帐户与日常工作的减少影响的范围。

### 二、创建一个新用户

一旦你以Root用户登录，我们准备添加新的用户账户，并且使用这个账户登录。

这个示例创建了一个名为“heqingbao”的新用户,但是你应该把它换成一个你喜欢的用户名:

```shell
[root@VM_0_3_centos ~]# adduser heqingbao
```

接下来为新用户分配一个密码：
```shell
[root@VM_0_3_centos ~]# passwd heqingbao 
更改用户 heqingbao 的密码 。
新的 密码：
无效的密码： 密码包含用户名在某些地方
重新输入新的 密码：
passwd：所有的身份验证令牌已经成功更新。
```

### 三、权限管理

现在，我们有一个新的普通账户，然而我们有时可能需要做一些管理任务。

为了避免退出普通账户再登录Root账户，我们可以建立一个`超级用户`或为普通用户提供Root权限，这将允许我们普通用户以管理员权限运行命令，在执行命令前加`sudo`命令。

将这些权限添加到我们的新用户，我们需要将新用户添加到`wheel`组。默认情况下，在CentOS7系统，属于`wheel`组的用户可以使用`sudo`命令。

作为`root`，运行如下命令把新用户添加到`wheel`组：

```shell
[root@VM_0_3_centos ~]# gpasswd -a heqingbao wheel
正在将用户“heqingbao”加入到“wheel”组中
```
现在，heqingbao这个用户可以以超级用户的特权运行命令！

### 四、添加Public Key认证（推荐）

下一步是为新用户设置public key认证以保护你的服务器。设置这一步后会要求新用户用私有的ssh密钥进行登录，增加服务器的安全性。

#### 4.1 生成一个密钥对

在本机上（比如我这里使用Mac），如果你没有SSH密钥对（包括公钥和私钥），你需要生成一个。如果你已经有一个，略过本节，直接跳到4.2节`复制公钥`步骤。

要生成新的密钥，在本机终端执行以下命令：

```shell
➜  ~ ssh-keygen
```

将看到如下输出：
```shell
Generating public/private rsa key pair.
Enter file in which to save the key (/home/heqingbao/.ssh/id_rsa): 
```

按回车键将接受这个文件名称和路径，如果需要可以输入一个新的名称。我这里直接按回车使用默认。

接下来你将被提示为这个密钥Key设置密码，你可以输入密码或者留空，这里按回车使用默认的空。

**注意：** 如果你使用空密码，使用私钥的时候将不会要求输入密码进行身份验证。如果使用非空密码，将需要私钥和密码来登录进行身份难，使用密钥+密码的方式会更加安全。但是这两种方法都比使用基本的密码登录更安全。

这将在当前用户的 home目录的`.ssh`目录里生成一个私钥Key`id_rsa`和一个公钥Key`id_rsa.pub`文件。记住，私钥不应该共享给任何不应该访问你服务器的人或服务。

#### 4.2 复制公钥Key

生成完SSH密钥对后，你想要把你的公钥复制到新的服务器上。以下介绍两种简单的方法。

##### 4.2.1 使用ssh-copy-id

如果你的本地机器安装有`ssh-copy-id`脚本，你可以用它来安装你的公钥到任何你登录的服务器。

运行`ssh-copy-id`脚本，通过指定用户名和服务器的IP地址：

```shell
➜  ~ ssh-copy-id heqingbao@SERVER_IP_ADDRESS
```

在输入服务器的密码后，你的公钥key将被添加到远程用户的`.ssh/authorized_keys`文件里。对应的私钥（比如我这里Mac上的）现在可以用于登录到服务器。

可以使用以下命令登录测试：
```shell
➜  ~ ssh heqingbao@SERVER_IP_ADDRESS       
Last login: Sat Nov 17 14:55:43 2018 from LOCAL_IP_ADDRESS
```

##### 4.2.2 手动安装公钥Key

假设你使用上面的步骤生了SSH密钥对，使用以下命令在本地机器的终端打印你的公钥(id_rsa.pub)：

```shell
➜  ~ cat ~/.ssh/id_rsa.pub
```
将输入类似如下的内容：
```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCgukSOAjVfuN72VGE4rail4mscNm2oJmxNIsR0TZZXnLcA4z5AgCu5MnpFkjhTDjzKMlswEkzrLARJDq4mAbLQNGiA90D4OnI7+Oe6frmsZ2DaxIjMNBB3PXdM8M8t0z5P0j4VCUDqYdan6PS7B5wXG4qb0MAumItSv9r7doPWgRbDcZ6v5b7JiCB24jF52sfBvN7l88q0OKR1Wm+yqrlDJZUIAymt1OnF2WaE9Eg2HUmVvRR76Ay7jNUupZOLSDn8/E68VHWdJMYkmXRg1QakF2TGufG7/bO2wD7P8/dP3ruMwmkkXX/PqlACj4LWuECnLnnzdhf8uUdxKh3gR7hwuxVxuMA9TkNmsl/8k+WZxqFHE+InMuIoVr1yW16idmFNTx5RzCWbrrYcIhJgfDY8ids8eJn4UBnuWf0oVTWNudt4rArXN7AP5goVFynqMVVUhDpha8nJbiI5M6W0POXP16paQz4W4EGJnyWHPaYt0fNvXCkUSEWrnYJII01/UT4xHaEDUB2z0P/3xnMDe+Zlv8mMdVVoJXMP2EnJJBrlrvh9YC7WTncpSPSP3knFnIdbPzQcN5SwMUpR5Z0+tZZeBkco4yC4y6qr4UtqtZjqREQihckAGrBUXnn4wRTnW6BBpd9YCm9aSHjhM8+wdn+sDCbuEoj1XA4mIKHgp5FShw== localuser@machine.local
```
把以上内容复制到粘贴板。

**将公钥Key添加到新的远程用户**

让使用SSH密钥作为新的远程用户进行登录验证，你必须在用户的主目录添加特殊的公钥文件。

**在服务端**， 作为root用户，输入以下命令切换到新用户：

```shell
[root@VM_0_3_centos ~]# su - heqingbao
上一次登录：六 11月 17 15:11:17 CST 2018从 IP_ADDRESSpts/1 上
```
现在你将处在新用户的主目录。

创建一个名为`.ssh`的目录，并限制期权限：

```shell
[heqingbao@VM_0_3_centos ~]$ mkdir .ssh
[heqingbao@VM_0_3_centos ~]$ chmod 700 .ssh
```
在`.ssh`目录下创建`authorized_keys`文件，然后把公钥Key粘贴到里面并保存。

现在修改`authorized_keys`文件的权限：
```shell
[heqingbao@VM_0_3_centos ~]$ chmod 600 .ssh/authorized_keys
```
退出到root用户：
```shell
[heqingbao@VM_0_3_centos ~]$ exit
```

现在你可以使用本地机器上的私钥用SSH的方式登录到远程服务器。

### 五、配置SSH后台守护

现在我们已经有了新账户，我们可以修改SSH守护配置（让我们远程登录的程序）不允许远程SSH访问root账户，以增加服务器的安全性。

首先作为root用户，打开配置文件：
```
[root@VM_0_3_centos ~]# vim /etc/ssh/sshd_config
```
在这里，我们可以选择禁用通过ssh root登录。这通常是一个更安全的设置，因为现在可以在必要的时候通过普通用户登录和访问服务器升级特权。

要禁用远程root登录，我们需要找到这样的行：
```
#PermitRootLogin yes
```
取消注释，并且把值修改成`no`

强烈建议每个服务器禁用远程root登录。

#### 重新加载SSH

现在我们对SSH做了修改，我们需要重新启动SSH服务，这样它将使用我们的新配置。

```shell
[root@VM_0_3_centos ~]# systemctl reload sshd
```

现在，在我们退出服务器之前，我们应该测试新的配置。直接到我们可以确认可以成功建立新连接后才能断开。（避免丢失服务器）

打开一个新的本地终端窗口（比如我使用Mac），在新窗口中，我们需要开始一个新的连接到服务器。这一次不使用root账户，我们要豕新的账户创建连接：

```
➜  ~ ssh heqingbao@SERVER_IP_ADDRESS 
```

将提示你输入密码，之后你将会作为新用户登录服务器。

记住，如果需要使用root特权运行一个命令时，在前面输入`sudo`，类似这样：

```shell
$ sudo command_to_run
```

比如：
```shell
[heqingbao@VM_0_3_centos ~]$ sudo ls

我们信任您已经从系统管理员那里了解了日常注意事项。
总结起来无外乎这三点：

    #1) 尊重别人的隐私。
    #2) 输入前要先考虑(后果和风险)。
    #3) 权力越大，责任越大。

[sudo] heqingbao 的密码：
```

如果一切正常，你可以退出session通过以下命令：
```
exit
```

### 六、对于新的CentOS7系统的一些推荐配置

#### 6.1 配置防火墙

防火墙可以为服务器提供一些基本的安全配置。CentOS附带一个叫做`firewall-cmd`命令可以用来配置防火墙策略。我们的基本策略将锁定一切，我们没有一个很好的理由保持所有端口开放。

`firewalld`服务有能力在不断开当前连接的情况下进行修改，以上面的服务器为例：

```shell
[heqingbao@VM_0_3_centos ~]$ sudo systemctl start firewalld
```
现在服务已经启动并运行，我们可以使用`firewall-cmd`命令来获取和设置防火墙策略信息。`firewall`使用区域(zones)的概念标识网络上其它主机的可信度。这个标签可被分配的具体规则取决于我们有多信任网络。

在本文中，我们将只调整默认zone的策略。当我们策略加载防火墙，将会把zone应用到我们的接口。我们应该添加一些例外批准服务到服务到防火墙，期中最重要的是SSH，因它我们需要保留远程管理访问到服务器。

如果你没有修改ssh守护进程的默认端口，你可以直接使用名字来启动服务：
```shell
$ sudo firewall-cmd --permanent --add-service=ssh
```
如果你已经改变了SSH的默认端口，你需要显示地指定新的端口，还需要指明服务所使用的协议。

```shell
$ sudo firewall-cmd --permanent --remove-service=ssh
$ sudo firewall-cmd --permanent --add-port=4444/tcp
```

这是开启防火墙最低限制的配置，因为需要远程连接管理服务器。

如果你打算运行传统的HTTP web服务，你需要开启`http`服务：

```shell
$ sudo firewall-cmd --permanent --add-service=http
```

如果你计划运行一个启用了SSL/TSL的web服务黑咖啡，你应该允许流量为`https`：
```shell
$ sudo firewall-cmd --permanent --add-service=https
```
如果你需要启用SMTP邮件，你需要：
```shell
$ sudo firewall-cmd --permanent --add-service=smtp
```

查看所有可以开启的服务，可以输入：
```shell
$ sudo firewall-cmd --get-services
```
当配置完成以后，可以查看配置的所有例外服务列表：
```shell
$ sudo firewall-cmd --permanent --list-all
```
当你准备应用改变，重新加载防火墙：
```shell
$ sudo firewall-cmd --reload
```
如果在测试之后，一切工作如预期的那样，你应该确保防火墙服务在开机后自动开启。
```shell
$ sudo systemctl enable firewalld
```

记住，后面可能配置的服务，你必须显示地打开防火墙（服务和端口）。

#### 6.2 配置时区（timezone）和网络时间（NTP）同步

第一步将确保您的服务器运行在正确的时区。第二步将配置您的系统,其系统时钟同步标准时间由一个全球网络的NTP服务器维护。这将有助于防止因为时钟不同步导致的一些不一致的行为。

##### 6.2.1 配置时区

这是一个非常简单的过程，可以使用`timedatectl`命令完成。

首先，看一下有哪些可用的时区：
```shell
$ sudo timedatectl list-timezones
```

将列出可用的时区列表，选择并设置正确的时区：
```shell
$ sudo timedatectl set-timezone Asia/Shanghai
```
你的系统将使用所选的时区，可以使用如下命令确认：
```shell
[heqingbao@VM_0_3_centos ~]$ sudo timedatectl
      Local time: 六 2018-11-17 16:31:39 CST
  Universal time: 六 2018-11-17 08:31:39 UTC
        RTC time: 六 2018-11-17 08:31:38
       Time zone: Asia/Shanghai (CST, +0800)
     NTP enabled: no
NTP synchronized: yes
 RTC in local TZ: no
      DST active: n/a
```
##### 6.2.2 配置NTP同步

现在已经设置了时区，我们应该配置NTP，这将允许你的服务器与其它服务器保持同步。使依赖服务器时间的服务有正确的行为。

对于NTP同步，我们使用一个叫做`ntp`的服务，我们可以从CentOS的默认源里面安装：
```shell
$ sudo yum install ntp
```
接下来，你需要为当前session启动ntp服务，并且配置每次开机自动启动：
```shell
$ sudo systemctl start ntpd
$ sudo systemctl enable ntpd
```
现在，你的服务器将自动从global servers同步时钟。

#### 6.3 创建一个交换文件

Linux 下至少有两种方法可以配置系统的 swap. 一种是直接格式化一个分区, 用这个分区作为swap区. 另一种是创建一个文件, swap 的内容都丢到文件里面去。

这里使用文件的方式。

如果我们需要一个4GB的文件的，我们可以创建一个swap文件：/swapfile

```shell
$ sudo fallocate -l 4G /swapfile
```

创建文件后，我们需要限制访问权限，这样其它的用户和进程不能看到写了什么：

```shell
$ sudo chmod 600 /swapfile
```

告诉系统这是一个交换格式的文件：

```shell
$ sudo mkswap /swapfile
```

现在，告诉系统 它可以使用交换文件：
```shell
$ sudo swapon /swapfile
```

我们的系统正在使用这个会话的交换文件，但是我们需要修改系统文件以便我们的服务会开机自动启动，可以使用：
```shell
$ sudo sh -c 'echo "/swapfile none swap sw 0 0" >> /etc/fstab'
```
然后，你的系统 应该在每次启动的时候都会使用交换文件。