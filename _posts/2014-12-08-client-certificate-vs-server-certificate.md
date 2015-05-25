---
layout: post
title: "客户端证书与服务端证书"
categories:
- technology
tags:
- Certificate


---

## Server Certificate ##  

服务器端证书用于标识一个Server，典型地被签发给主机名，比如机器名或者host-header(www.baidu.com)。对Server证书而言，”Issued to”域意味着选择哪个签发的hostname。  

![图片](/assets/images/server_certificate.png)    

1.	目的：加密和解密内容
2.	真实世界里只有ROOT CA有自签名证书
3.	存储：对于window IIS，Server certificates are stored in the Personal store of the “Local Computer” account
4.	证书可以存储在用户账户对应的个人存储目录下，也是IIS缺省的搜索路径
	
## Client Certificate ##  

服务器端证书用于标识一个Client或者user，意味着向Server验证Client，”Issued to”域采用用户的名字。  

![图片](/assets/images/client_certificate.png)      


1.	目的：仅用于标识身份
2.	真实世界里只有ROOT CA有自签名证书
3.	Client certificate and private key的角色：
- Client证书的证书和私钥作用就是为了在Client authentication中提供Client的数字签名。
- crt后缀名文件仅仅包含证书，证书和私钥一般包含在.p12或者.pfx文件中(PKCS12格式)。
- 私钥如果单独存放，一般为.pem后缀名。
- Server端需要client端的数字签名，然后发送至Server，Server利用client的证书（即公钥）解密，以确认client端的身份。其中Client端的数字签名生成过程如下：首先对SSL交互中的随机值进行HASH，然后用client的私钥进行加密。  


## 中间人攻击与证书 ##  

SSL中间人攻击的形式主要有三种：SSL downgrading，SSL stripping和fake of SSL certs。
在SSL协议中，证书是用来进行身份验证的，这里有两个关键的检查点：  

- 第一个是CA验证，对于客户端而言，验证服务器端证书依赖与本地的trust store，如果本地的trust  store被人篡改，那么避免中间人攻击则无从谈起，事实上这也正是PKI体系的弱点所在。  
- 第二个是hostname检查，用来确保所连接的服务器就是其声称的，如果有人劫持了DNS服务器，即利用DNS cache poisoning攻击让你的地址栏看起来很正常，这个时候劫持者就可以让此项检查通过。  

## 参考链接 ##  


1. [Understanding Man-In-The-Middle Attacks - Part 4: SSL Hijacking](http://www.windowsecurity.com/articles-tutorials/authentication_and_encryption/Understanding-Man-in-the-Middle-Attacks-ARP-Part4.html )
2. [Introducing Strict SSL](https://blog.cloudflare.com/introducing-strict-ssl-protecting-against-a-man-in-the-middle-attack-on-origin-traffic/)