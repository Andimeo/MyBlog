title: 在Google Cloud上搭建VPN
date: 2016-12-25 15:34:20
categories: VPS
tags:
- VPN
- Google Cloud
- IPsec
- StrongSwan
---
之前在Google Cloud上申请过一个VPS，在Free trial期间使用了一个月，后来就关掉了。当时发现自己搭建VPN好容易啊。
直到这次，想重新激活上次那个instance来做VPN的时候，VPN已链接，但是data plane的traffic却始终发送不出去。
于是重新搜索了教程，才搞定这件事情。这里主要针对Google Cloud总结一下一些重要步骤。

关于StrongSwan的配置部分主要参考下面这篇文章：
[使用Strongswan搭建IPSec/IKEv2 VPN](http://hjc.im/shi-yong-strongswanda-jian-ipsecikev2-vpn/)

# Google Cloud配置
建立好Compute Engine后，申请一个永久静态地址，然后配置SSH登录证书等内容。
这里不再赘述，Google自己的文档里已经交代的很清楚怎么做。

<!-- more -->

主要配置是在Google的Firewall rules里，这里决定了什么包可以访问到内部的instance，其后才是instance内部的iptables的配置。

这里我们建立一条新的rule，source filter填写`0.0.0.0/0`，代表任何地方发出的包，除非你只想在特定IP上访问，否则就填写这个。
然后在Allowed protocols and ports里添加`udp:500;udp:4500;esp`。

* 这里`udp:500`是`IKE`协议指定的端口，会通过这个端口来发送Control plane的包
* `udp:4500`是当source或者destination在`NAT`后时转而使用的端口，另外当`NAT`被检测到后，Data plane的数据包也会被使用端口，这个时候`ESP`的数据包会被包含在`UDP`包中，端口也是这个
* 最后`esp`就是允许Data plane的数据包通过

另外需要勾选允许`ip forwarding`。

至此Google Cloud的配置就完成了。

# 安装StrongSwan
这里假设使用的是Ubuntu作为操作系统，其他OS请参考相应教程。

## 准备工作
安装PAM库和SSL库，以及make和gcc
```
apt-get update
apt-get install libpam0g-dev libssl-dev make gcc
```
## 下载StrongSwan的源码并编译
```
#下载
wget http://download.strongswan.org/strongswan.tar.gz
tar xzf strongswan.tar.gz
cd strongswan-*

#编译
./configure  --enable-eap-identity --enable-eap-md5 \
--enable-eap-mschapv2 --enable-eap-tls --enable-eap-ttls --enable-eap-peap  \
--enable-eap-tnc --enable-eap-dynamic --enable-eap-radius --enable-xauth-eap  \
--enable-xauth-pam  --enable-dhcp  --enable-openssl  --enable-addrblock --enable-unity  \
--enable-certexpire --enable-radattr --enable-tools --enable-openssl --disable-gmp --enable-kernel-libipsec
make

#安装
make install # if not root please prepend sudo

```

完成后执行`ipsec version`查看是否安装成功。

# 生成证书

## 生成CA私钥
```
ipsec pki --gen --outform pem > ca.pem  
```

## 利用私钥签名CA证书
```
ipsec pki --self --in ca.pem --dn "C=com, O=myvpn, CN=VPN CA" --ca --outform pem >ca.cert.pem
```

## 生成server端私钥
```
ipsec pki --gen --outform pem > server.pem
```

## 用CA证书签发server端证书
这里需要将下面的地址更换为我们刚才申请到的永久IP地址。

```
SERVER_ADDR="[REPLACE_WITH_YOUR_OWN_IP_ADDRESS]"
ipsec pki --pub --in server.pem | ipsec pki --issue --cacert ca.cert.pem \
--cakey ca.pem --dn "C=com, O=myvpn, CN=$SERVER_ADDR" \
--san="$SERVER_ADDR" --flag serverAuth --flag ikeIntermediate \
--outform pem > server.cert.pem
```
## 生成client端私钥
```
ipsec pki --gen --outform pem > client.pem
```
## 利用CA证书签发client端证书
```
ipsec pki --pub --in client.pem | ipsec pki --issue --cacert ca.cert.pem --cakey ca.pem --dn "C=com, O=myvpn, CN=VPN Client" --outform pem > client.cert.pem
```

## 生成client端p12证书
```
openssl pkcs12 -export -inkey client.pem -in client.cert.pem -name "client" -certfile ca.cert.pem -caname "VPN CA"  -out client.cert.p12
```

## 安装证书
```
# if not root probably you need to prepend sudo in front of the following commands
cp -r ca.cert.pem /usr/local/etc/ipsec.d/cacerts/
cp -r server.cert.pem /usr/local/etc/ipsec.d/certs/
cp -r server.pem /usr/local/etc/ipsec.d/private/
cp -r client.cert.pem /usr/local/etc/ipsec.d/certs/
cp -r client.pem  /usr/local/etc/ipsec.d/private/
```

# 配置StrongSwan

## 配置ipsec.conf

将`/usr/local/etc/ipsec.conf`替换为如下内容：
```
config setup
    uniqueids=never

conn iOS_cert
    keyexchange=ikev1
    # strongswan version >= 5.0.2, compatible with iOS 6.0,6.0.1
    fragmentation=yes
    left=%defaultroute
    leftauth=pubkey
    leftsubnet=0.0.0.0/0
    leftcert=server.cert.pem
    right=%any
    rightauth=pubkey
    rightauth2=xauth
    rightsourceip=10.31.2.0/24
    rightcert=client.cert.pem
    auto=add

conn android_xauth_psk
    keyexchange=ikev1
    left=%defaultroute
    leftauth=psk
    leftsubnet=0.0.0.0/0
    right=%any
    rightauth=psk
    rightauth2=xauth
    rightsourceip=10.31.2.0/24
    auto=add

conn networkmanager-strongswan
    keyexchange=ikev2
    left=%defaultroute
    leftauth=pubkey
    leftsubnet=0.0.0.0/0
    leftcert=server.cert.pem
    right=%any
    rightauth=pubkey
    rightsourceip=10.31.2.0/24
    rightcert=client.cert.pem
    auto=add

conn windows7
    keyexchange=ikev2
    ike=aes256-sha1-modp1024!
    rekey=no
    left=%defaultroute
    leftauth=pubkey
    leftsubnet=0.0.0.0/0
    leftcert=server.cert.pem
    right=%any
    rightauth=eap-mschapv2
    rightsourceip=10.31.2.0/24
    rightsendcert=never
    eap_identity=%any
    auto=add
```
## 配置strongswan.conf
将`/usr/local/etc/strongswan.conf`替换为如下内容：
```
charon {
         load_modular = yes
         duplicheck.enable = no
         compress = yes
         plugins {
                 include strongswan.d/charon/*.conf
         }
         dns1 = 8.8.8.8
         dns2 = 8.8.4.4
         nbns1 = 8.8.8.8
         nbns2 = 8.8.4.4
 }
 include strongswan.d/*.conf
```

## 配置ipsec.secrets
将`/usr/loca/etc/ipsec/ipsec.secrets`内容替换为如下内容：
```
: RSA server.pem
: PSK "mykey"
: XAUTH "mykey"
[用户名] %any : EAP "[密码]"
```
注意将PSK、XAUTH处的"mykey"编辑为唯一且私密的字符串，并且将[用户名]改为自己想要的登录名，[密码]改为自己想要的密码（[]符号去掉），可以添加多行，得到多个用户。

# 配置iptables

## 修改sysctrl.conf
打开`/etc/sysctrl.conf`，然后uncomment包含`net.ipv4.ip_forward=1`的这一行。

保存后，执行`sysctrl -p`。

## 修改iptables
将`INF`替换为自己的网络接口.
```
INF="Your own network interface"
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s 10.31.2.0/24  -j ACCEPT
iptables -A INPUT -i $INF -p esp -j ACCEPT
iptables -A INPUT -i $INF -p udp --dport 500 -j ACCEPT
iptables -A INPUT -i $INF -p tcp --dport 500 -j ACCEPT
iptables -A INPUT -i $INF -p udp --dport 4500 -j ACCEPT
# for l2tp
iptables -A INPUT -i $INF -p udp --dport 1701 -j ACCEPT
# for pptp
iptables -A INPUT -i $INF -p tcp --dport 1723 -j ACCEPT
iptables -A FORWARD -j REJECT
iptables -t nat -A POSTROUTING -s 10.31.2.0/24 -o $INF -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.31.2.1/24 -o $INF -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.31.2.2/24 -o $INF -j MASQUERADE
```

## 保存iptables且开机自动启动

```
iptables-save > /etc/iptables.rules
cat > /etc/network/if-up.d/iptables<<EOF
#!/bin/sh
iptables-restore < /etc/iptables.rules
EOF
chmod +x /etc/network/if-up.d/iptables
```

## 重启ipsec服务
```
ipsec restart
```

至此大功告成。

WP8.1手机安装`ca.cert.pem`，进入设置`VPN`添加`IKEv2`连接，地址为证书中的地址或IP，通过用户名-密码连接。
Windows连接也是一样，但注意将证书导入本地计算机而不是当前用户的“受信任的证书颁发机构”。
iOS/Android/Mac OS X设备添加`Cisco IPSec PSK`验证方式，预共享密钥是`/usr/local/etc/ipsec.secrets`中PSK后的字符串（不含引号），用户名密码同上，可以通过任意域名或IP连接，不需要证书.

接下来有时间我会试图把整个安装过程配置为一个Docker Build file，这样以后配置新的instance就跟方便了。
