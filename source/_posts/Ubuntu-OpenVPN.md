title: Ubuntu 14.04配置OpenVPN
date: 2015-03-01 17:51:42
tags: OpenVPN
---
### 第一步 安装和配置OpenVPN环境

#### OpenVPN配置

在安装任何包之前，首先我们将更新Ubuntu的库列表。

```bash
apt-get update
```

```bash
apt-get install openvpn easy-rsa
```

```bash
gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz > /etc/openvpn/server.conf
```

在`server.conf`文件中，我们将做一些修改。

首先找到类似如下的块：

```
# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh1024.pem 1024
# Substitute 2048 for 1024 if you are using
# 2048 bit keys.
dh dh1024.pem
```

把`dh1024.pem`改成`dh2048.pem`，这样当生成server和client key的时候，这将使用两倍的RSA key长度。

然后找到这一段：

```
# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
;push "redirect-gateway def1 bypass-dhcp"
```

取消最后一行的注释`push "redirect-gateway def1 bypass-dhcp"`。

再找到这个区域：

```
# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"
```

取消最后两行的注释，并改成：

```
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
```

最后找到这个区域：

```
# You can uncomment this out on
# non-Windows systems.
;user nobody
;group nogroup
```

取消最后两行的注释，改完后看起来长这样：

```
user nobody
group nogroup
```

#### 配置包转发

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

```bash
vim /etc/sysctl.conf
```

在这个文件顶部附近，我将看到：

```
# Uncomment the next line to enable packet forwarding for IPv4
#net.ipv4.ip_forward=1
```

取消注释`net.ipv4.ip_forward`

#### 简单的防火墙ufw

```
ufw allow ssh
```

```
ufw allow 1194/udp
```

打开`/etc/default/ufw`，找到`DEFAULT_FORWARD_POLICY="DROP"`，把**DROP**改成**ACCEPT**。

接下来需要配置网络地址转换和IP伪装，打开`/etc/ufw/before.rules`，添加`# START OPENVPN RULES`到`# END OPENVPN RULES`之间的内容：

```
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0] 
# Allow traffic from OpenVPN client to eth0
-A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES

# Don't delete these required lines, otherwise there will be errors
*filter
```

```
ufw enable
```

检查ufw的主要防火墙规则

```
ufw status
```

会看到类似的输出：

```
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
1194/udp                   ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
1194/udp (v6)              ALLOW       Anywhere (v6)
```

### 第二步 创建一个证书频发机构和服务端证书/Key


#### 配置和编译证书频发机构

```bash
cp -r /usr/share/easy-rsa/ /etc/openvpn
```

```bash
mkdir /etc/openvpn/easy-rsa/keys
```

编辑`/etc/openvpn/easy-rsa/vars`，然后改变相应的内容：

```
export KEY_COUNTRY="US"
export KEY_PROVINCE="TX"
export KEY_CITY="Dallas"
export KEY_ORG="My Company Name"
export KEY_EMAIL="sammy@example.com"
export KEY_OU="MYOrganizationalUnit"
```

为了方便，我们将使用`server`作为key的名字，紧临着上面这一段修改：

```
export KEY_NAME="server"
```

我们需要生成`Diffie-Hellman`参数，这可能会花几分钟时间：
```
openssl dhparam -out /etc/openvpn/dh2048.pem 2048
```

```
cd /etc/openvpn/easy-rsa
```

初始化PKI(Public key Infrastructure)。(注意点和空格)

```
. ./vars
```

```
./clean-all
```

```
./build-ca
```

如果不需要改变，直接回车即可。

#### 生成服务端证书和key


仍然在`/etc/openvpn/easy-rsa`目录：
```
./build-key-server server
```

跟`./build-ca`有相似的输出，如果不需要修改，直接回车即可。但最后两步需要回复`y`确认。

#### 移动服务端证书和key

OpenVPN期望在`/etc/openvpn`目录里看到服务端的CA。
```
cp /etc/openvpn/easy-rsa/keys/{server.crt,server.key,ca.crt} /etc/openvpn
```

到此，OpenVPN服务端做好了，可以开始了：

```
service openvpn start
service openvpn status
```

### 第三步 为客户端生成证书和key

#### Key和证书绑定

仍然在`/etc/openvpn/easy-rsa`目录。

```
./build-key client1
```

如果不需要修改直接回车，最后两项需要明确的`y`回应。


复制例子配置到我们的目录，再修改。

```
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/easy-rsa/keys/client.ovpn
```

#### 将证书和密钥转移到客户端设备

需要将以下这些文件转移到客户端设备：

```
/etc/openvpn/easy-rsa/keys/client.ovpn
/etc/openvpn/easy-rsa/keys/client1.crt
/etc/openvpn/easy-rsa/keys/client1.key
/etc/openvpn/ca.crt
```

可以使用`scp`或者`pscp`拷贝：

```
scp root@your-server-ip:/etc/openvpn/easy-rsa/keys/client1.key .
```

拷贝完成后，最终客户端将得到这四个文件：

- client1.crt
- client1.key
- client.ovpn
- ca.crt


### 第四步 为客户端创建一个统一的OpenVPN属性文件

打开`client.ovpn`文件，找到`;remote my-server-2 1194`这行，取消注释，并且使用你自己的VPN IP。

对`user nobody`和`group nogroup`取消注释：

```
# Downgrade privileges after initialization (non-Windows only)
user nobody
group nogroup
```

然后需要把ca.crt,client1.crt和client1.key的内容粘贴到这个.ovpn的文件的末尾，使用基本的xml格式语法，类似这种：

```
<ca>
(insert ca.crt here)
</ca>
<cert>
(insert client1.crt here)
</cert>
<key>
(insert client1.key here)
</key>
```

当粘贴完成，这个文件的末尾应该类似这样的：

```
<ca>
-----BEGIN CERTIFICATE-----
. . .
-----END CERTIFICATE-----
</ca>

<cert>
Certificate:
. . .
-----END CERTIFICATE-----
. . .
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
. . .
-----END PRIVATE KEY-----
</key>
```

保存并退出，现在我们已经有一个为client1生成的统一的OpenVPN客户端配置文件。


### 第五步 安装客户端配置文件

#### Window

Windows可以直接安装[OpenVPN](https://openvpn.net/index.php/open-source/downloads.html)。

安装完成后，把前面生成的client.ovpn拷贝到安装目录下的`config`文件夹下。

然后当你启动OpenVPN，它将自动识别这个配置文件。之后可以使用图形化的界面连接和断开等操作。

#### OS X

OS X可以使用免费的[Tunnelblick](https://code.google.com/p/tunnelblick/)，安装完成后，双击client.ovpn文件，Tunnelblick将自动安装这个客户端配置。

#### Android

Android端可以使用[Android OpenVPN Connect](https://play.google.com/store/apps/details?id=net.openvpn.openvpn)

#### iOS

iOS设备可以使用[OpenVPN Connect](https://itunes.apple.com/us/app/id590379981)，不过可能需要进入美区AppStore才可以搜索得到，或者使用一些助手类工具安装。然后可以使用QQ简单地把client.opvpn发送到手机端，再选择使用OpenVPN打开即可。

-------------

然后就可以妥妥地翻墙了。