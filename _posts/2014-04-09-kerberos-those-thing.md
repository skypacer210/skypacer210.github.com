---
layout: post
title: "Kerberos那些事"
categories:
- technology 
tags:
- ACCESS CONTROLS


---
 

本文将对kerberos的协议细节做一个粗略介绍~  

原文链接[explain-like-im-5-kerberos](http://www.roguelynn.com/words/explain-like-im-5-kerberos/)  

------ 

##1、Nutshell  

Kerberos源自古希腊神话中的三头狗，对应引入了第三方验证系统。

Kerberos有如下特点：  

* 一种验证协议
* 采用ticket进行验证
* 密码通过网络进行传递，避免了本地存储
* 需要引入第三方机构
* 基于对称加密算法  


一个ticket意味着用特定服务请求所对应的key加密后的身份证明，如果其有效，你可以在一个kerberos域内访问该服务。

Ticket由很多信息组成，以保证特定服务授权的针对性和实效性。
Keberos其目的仍是保证client和client所请求的服务或者主机之间的session key。  

典型实例：
访问内网的工资系统，每次重新输入user/password的时候或者在终端运行kinit USER的时候，用户的ticket会更新。  

##2、Kerberos Realm 

管理员创建realm，囊括了所有可能的访问，即定义了不同服务和主机的访问策略和授权，其本质为依据谁能够访问什么的原则定义了Kerberos的管理内容。  

用户主机位于该realm,同时也包括了所期望访问的服务、KDC(Key Distribution Center)。  

KDC可以分为Authentication Server 和Ticket Granting Server。  

![图片](/assets/images/kerberos/Kerb.001.jpg)

##3、Kerberos那些事  

当访问一个服务或者主机时，存在着三种交互：  

* Client 和Authentication Server
* Client 和Ticket Granting Server
* Client和服务器

其他要点如下：

* 每次交互client可以收到两条消息，一条是可以解密的，一条是无法解密的；
* Client期望访问的服务或者主机从不直接与KDC通信；
* KDC存储了其数据库下所有主机和服务的密钥；
* 密钥由密码加上一组随机数的哈希值，哈希算法由Kerberos的具体实现选择。对于服务和主机而言，其本身是没有密码的。一个密码实际上是由管理员的初始启动时产生和存储于服务和主机中的；
* 所有密钥存储于KDC数据库；
* KDC本身由主密钥加密；
* 已经存在Kerberos的配置和实现采用公钥加密。
 
上述定义的消息和内容并未反映他们在TCP、UDP发送中的实际顺序。
下面描述如果访问一个内部的HTTP服务，将会发生什么。

###3.1、You and the Authentication Server  


在访问HTTP服务之前，client必须向authentication server介绍自己。当登陆系统或者运行kinit USERNAME的时候，通过一个明文请求TGT(Ticket Granting Ticket)。其具体内容包括：

* Name/ID
* 服务ID(有很多的服务ID？)
* 网络地址
* 合法TGT的请求生命周期

![图片](/assets/images/kerberos/Kerb.002.jpg)

Authentication server仅仅检查是否存在于KDC数据库，无需密码。

![图片](/assets/images/kerberos/Kerb.003.jpg)

Authentication server回复两条消息：  
消息一，即用TGS的密钥(TGS和Authenticator都知道各自的密钥？)加密的ticket, 即TGT，这个TGT包括如下：
  
* Client name/ID；
* TGS name/ID,即ticket服务器也自报家门；
* 时间戳；
* Client的网络地址；
* TGT生命周期，即ticket的生命周期；
* TGS session Key，即会话密钥。  

消息二，即client密钥加密的TGS会话密钥等：  

* TGS name/ID;
* Timestamp;
* 生命周期；
* TGS session key;

TGS session key就是client和TGS之间的共享密钥。  

![图片](/assets/images/kerberos/Kerb.004.jpg)

Client 密钥通过提示client输入密码，再加上一个随机串，最后对整个进行哈希得到。可以用该密钥解密TGS发送过来的第二条消息，以获取TGS 会话密钥。显然，如果密码错误，client 将无法解密该条消息。实际上是完成了对client password的隐性验证。   

![图片](/assets/images/kerberos/Kerb.005.jpg) 

由于不知道TGS 密钥，无法解密TGT，将此加密的TGT存储于credential cache当中。


###3.2、You and the Ticket Granting Server

此时，client拥有TGS session key，但是没有TGS密钥。
现在轮到client发送两条消息了：  

第一条消息为Authenticator 准备，用TGS会话密钥加密如下信息：  
Client name/ID + timestamp  

第二条消息是明文，包含请求的HTTP服务名称和HTTP服务的ticket生命周期同时还有加密的Authenticator和TGT。   

![图片](/assets/images/kerberos/Kerb.006.jpg)

TGS收到这两个消息后，首先检查KDC数据库中是否存在所请求的HTTP服务。  

![图片](/assets/images/kerberos/Kerb.007.jpg)

如果存在，TGS用TGS密钥解密TGT。因为解密后的TGT包含TGS会话密钥，因此TGS可以解密client所发送的Authenticator。    

![图片](/assets/images/kerberos/Kerb.008.jpg)

TGS然后将作如下操作：  

* 比较authenticator中包含的client ID和TGT中包含的；
* 比较两者之间的时间戳，误差一般可容忍2分钟；
* 通过生命周期字段检查TGT是否过期；
* 检查authenticator已经不在TGS的cache当中。
* 如果原始请求中的网络地址不为NULL，比较其中的IP地址与TGT中的；

此时，TGS开始随机产生client所请求的HTTP服务的会话密钥，准备HTTP服务ticket，包含如下信息：  
* Client name/ID;
* HTTP服务name/ID;
* Client的网络地址；
* 时间戳；
* Ticket的生命周期；
* HTTP服务session key;  

上述信息由HTTP服务密钥加密。  

![图片](/assets/images/kerberos/Kerb.009.jpg)

然后TGS向client发送两条信息：  

第一条是由HTTP服务密钥加密的HTTP服务ticket；  

第二条是由TGS session key加密的信息：  
  
* HTTP服务name/ID;
* 时间戳；
* Ticket的生命周期；
* HTTP服务session key。  

Client利用cached的TGS session key解密第二条消息，以获取HTTP服务session key。而无法解密第一条消息，因为其没有HTTP服务密钥。

![图片](/assets/images/kerberos/Kerb.010.jpg)

###3.3、 You and the HTTP Service  

为了访问HTTP服务，client准备了另外的Authenticator消息，包括：  
  
* Client name/ID;
* 时间戳。  

用HTTP服务session key加密后发送给HTTP服务，同时也发送未解密的HTTP服务ticket。  

![图片](/assets/images/kerberos/Kerb.011.jpg)

HTTP服务在收到消息后用HTTP服务密钥解密消息二，获取HTTP服务session key。然后用该Key解密client发送的第一条authenticator消息。    

![图片](/assets/images/kerberos/Kerb.012.jpg)

类似于TGS，HTTP服务器也要做如下校验：    

* Client ID;
* 时间戳；
* ticket是否过期；
* 避免反重放攻击，查看HTTP服务器的cache是否包含authenticator；
* 网络地址比较。  
 
如果通过校验，HTTP服务然后发送一条Authenticator消息，该消息包含其服务ID及时间戳，由HTTP服务session key加密。  

![图片](/assets/images/kerberos/Kerb.013.jpg)

Client收到HTTP服务发送的加密信息，用cache的HTTP 服务session key解密，由此接收到一条HTTP服务ID和时间戳的明文信息。  

此时client已经验证通过使用HTTP服务，将来的请求利用cached的HTTP服务ticket，前提是在定义的生命周期内。    

![图片](/assets/images/kerberos/Kerb.014.jpg)

这里有个前提，HTTP服务本身必须支持Kerberos。同时，client必须要有一个支持SPNEGO/Negotiate的浏览器。  

![图片](/assets/images/kerberos/Kerb.015.jpg)


##4、后续

token VS ticket
 