---
layout: post
title: "Anyconnect" 
date:   2017-09-25 10:35:06
categories: anyconnect
---

<!-- more -->

### 2022年已停用ocserv
```
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
# 选择登陆方式，这里使用密码登陆并且从 /etc/ocserv/ocpasswd 文件中读取用户名和密码
# 使用证书登录则启用 auth="certificate"
 
tcp-port = 443
udp-port = 443
# 这个代表 TCP和UDP监听的端口 默认443，可以更换其他端口，端口号可分开；或者禁用UDP
# ocserv默认使用基于UDP的TLS协议（DTLS）进行加速，但是UDP无法提供可靠的传输。TCP比UDP慢，但是可以提供可靠的传输。一个优化技巧是禁用DTLS，使用标准TLS（通过TCP），然后启用TCP BBR以提高TCP速度

compression = true
# 开启压缩
try-mtu-discovery = true
# 开启以后可以增强VPN性能

# cert-user-oid = 2.5.4.3
# 让服务器读取用户证书,适用于证书登录

server-cert = /etc/pki/ocserv/public/server.crt
server-key = /etc/pki/ocserv/private/server.key
# ca-cert = /etc/pki/ocserv/cacerts/ca.crt
# 服务器证书、私钥和CA证书的位置，这里的CA指的是签发登录证书的CA;用户名和密码登录方式不需要CA证书
# Strongswan则不一样，其服务器证书的CA根证书（包括中间证书）必须同时存在服务器和客户端的相关位置
# 注意这里的路径最后的位置不能有空格
max-clients = 16
# 允许同时连接的总客户端数量
max-same-clients = 2
# 同账号连接VPN最大客户端数量，0是不作限制
cookie-timeout = 172800
# 连接中断的时间在cookie-timeout数值内可以自动重连

ipv4-network = 172.17.0.0
ipv4-netmask = 255.255.0.0
 
dns = 8.8.8.8
dns = 1.1.1.1
# 服务端的DNS

cisco-client-compat = true
# 使ocserv兼容AnyConnect
```
添加路由表。AnyConnect限制200条路由表。
将私网地址添加到no-route，并在客户端勾选 Allow local(LAN) access when using VPN (if configured)，即可让私网地址不走VPN；客户端若配置双网卡，需确认路由问题，使用tracert分析判断。

```
#no-route = #全部注释掉no-route选项，启用 route；不能同时启用

route = 0.0.0.0/5
route = 8.0.0.0/7
route = 11.0.0.0/8
route = 12.0.0.0/6
route = 16.0.0.0/4
route = 32.0.0.0/3
route = 64.0.0.0/2
route = 128.0.0.0/3
route = 160.0.0.0/5
route = 168.0.0.0/6
route = 172.0.0.0/12
route = 172.32.0.0/11
route = 172.64.0.0/10
route = 172.128.0.0/9
route = 173.0.0.0/8
route = 174.0.0.0/7
route = 176.0.0.0/4
route = 192.0.0.0/9
route = 192.128.0.0/11
route = 192.160.0.0/13
route = 192.169.0.0/16
route = 192.170.0.0/15
route = 192.172.0.0/14
route = 192.176.0.0/12
route = 192.192.0.0/10
route = 193.0.0.0/8
route = 194.0.0.0/7
route = 196.0.0.0/6
route = 200.0.0.0/5
route = 208.0.0.0/4
```

用户管理
```
ocpasswd -c /etc/ocserv/ocpasswd name      #创建用户名为name的用户，会提示创建密码
ocpasswd -c /etc/ocserv/ocpasswd -d name   #删除用户名为name的用户，无任何提示
ocpasswd -c /etc/ocserv/ocpasswd -l name   #锁定用户名为name的用户，无任何提示
ocpasswd -c /etc/ocserv/ocpasswd -u name   #解锁用户名为name的用户，无任何提示
```
用户证书登录：只需 ocserv 信任 CA 即可，因此使用自签发证书。使用ocserv自动生成的CA证书签发客户端证书未能成功登录，所以使用自签发CA。

生成CA根证书的私钥，再用这个私钥自颁发根证书
```
certtool --generate-privkey --outfile ca-key.pem
```
创建根证书的tmpl模板ca.tmpl
```
cn = "BCOC CA"
organization = "BCOC"
serial = 1
expiration_days = 3650
ca
signing_key
cert_signing_key
crl_signing_key
```
颁发根证书
```
certtool --generate-self-signed --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem
```

生成一个私钥，再用这个私钥参与用户证书的签发
```
certtool --generate-privkey --outfile user-key.pem
```
创建用户证书的tmpl模板user.tmpl
```
cn = "harveymei"
unit = "standard"
expiration_days = 3650
signing_key
tls_www_client
```
颁发用户证书并将证书和私钥合成PKCS12格式,会提示创建证书名字和密码。安装证书时需要提供密码
```
certtool --generate-certificate --load-privkey user-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template user.tmpl --outfile user-cert.pem
certtool --to-p12 --load-privkey user-key.pem --pkcs-cipher 3des-pkcs12 --load-certificate user-cert.pem --outfile user.p12 --outder
```

再修改登录方式为证书验证。无论哪种方式都要启用系统的IP转发功能，否则无法访问网络。OpenVZ需要开启TUN，否则Anyconnect无法登录。

anyconnect客户端均使用p12或pfx这种包含密钥的证书，但是安卓端某些型号导入证书可能有bug，而且安卓端的anyconnect无法加载no-route路由表。若启用route，则苹果的客户端连接即断开。

openconnect客户端的证书和密钥分开导入，不需要合成p12格式。但是都不支持no-route路由，一般只用安卓端。安卓端都只支持route分流，但启用route的话，苹果客户端无法连接。

### 吊销证书

如果要禁止已经颁发出去的证书进行登录，可以吊销证书，或者通过登录脚本控制。

在ocserv.conf配置中有connect-script选项，填写脚本路径：
```
connect-script=/etc/ocserv/connectscript
```
然后创建该脚本，并允许执行：
```
#!/bin/sh
case $USERNAME in
    userA|userB|userC)
        exit 1
        ;;
        
    *)  
        ;;
esac
exit 0
```
脚本中的用户名称即为要吊销的证书的名称；只要返回值不为0，就不能连接成功。
