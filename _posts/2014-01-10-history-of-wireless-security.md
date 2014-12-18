---
layout: post
title: "history of wireless security"
categories:
- technology
tags:
- wireless
- security


---

# 无线安全之前世今生

------

本文将对wireless security的发展做一个粗略介绍，从open system到WPA、WPA2和802.1X，权当做抛砖引玉~

注：任何一次技术变革不仅仅是技术方案的较量，更多的是市场的博弈。原文链接[history of wireless security](http://securityuncorked.com/2008/08/history-of-wireless-security/)

------

##1、Open and SKA

802.11在发展早期主要采用两类验证方法-- Open System Association (OSA) 和Shared Key Authentication (SKA).

* Open = no authentication
* Shared Key = WEP

Open systems 相当于没有任何验证手段，任何人可以连接至无线网络。

SKA基于WEP加密，我们会在4个预共享密钥中选择一个作为所有无线站点连接AP的密钥，虽然后来WEP被发现存在安全漏洞。让我们来看看WEP何罪之有？
WEP本身有诸多问题，在wired介质当中是一个足够强健的算法，但是并不适合无线介质，下面是WEP的若干罪状：

- 采用基于流的算法
- 每包重用master key 
- 在AP之间采用相同的组播密钥
- 缺乏对client端的有效认证机制
- 易于遭受重放攻击和完整性攻击  


##2、 从WEP的改造到WPA的崭露头角  

###2.1、 “i”之初创  

随着WEP越来越多的漏洞被发现，IEEE决定成立802.11i工作组以改进无线安全。作为一个安全网络的过渡期，WPA应运而生，其目的在于开发一个强健的安全网络。  
802.11i工作组面临诸多困难，其中之一便是如何在仍旧使用现有硬件的基础上对WEP进行安全加密流程进行改造。作为一个过渡性解决方案，WPA在解决WEP现有不足的基础上同时考虑长期的无线安全。  

###2.2、 WPA与TKIP  

Wi-Fi联盟(Wi-Fi Alliance)的目的在于创建一个在主流厂家之间互通的安全平台；而WPA源于‘Wi-Fi Protected Access’其本身不是一个标准甚至不能称之为框架，仅仅是一个由无线厂商为达到相互之间的互通而形成的ad-hoc联合。  
WPA作为802.11i的一个准标准，既采用了特定的密钥管理，也遵循了802.11i中的验证架构-802.1X。由于当时的硬件不支持AES算法，WPA采用TKIP((Temporal Key Integrity Protocol))作为替代算法。

###2.3、WPA中的密钥建立机制  

WPA采用PMK建立PTKs，即会话密钥，其目的在于保护每个会话。伪随机数函数PSF (pseudo-random function)确保PTKs包含nonces、MAC地址计数、时间以及随机数。密钥建立机制的重新设计目的在于最大程度地降低主密钥被窃取的可能性。见图1。  

![图片](/assets/images/wireless_brief_fig_1.png)  

<center>Figure 1. WEP vs WPA</center>  

###2.4、WPA的其他改进点  

WPA在安全性方面的改进之处主要有以下几点:

 - IV扩展至56bit（对于弱密钥可以允许8bit的密钥扩展空间）
 - 采用PTKs（再一次扩展了有效的密钥空间）
 - 指定IV rotation，避免了密钥重用
 - 采用MICHAEL算法进行完整性校验
 - 通过IEEE802.1X进行网络验证
 - 增加了序列号以防止重放攻击
 - 报文头和负载的完整性检查以防止重定向攻击

###2.5、WPA小结  
WPA作为一种安全架构主要包含两类组件：第一类是加密组件TKIP；第二类是验证组件，可以是PSK也可以是RADIUS。除了安全性相比WEP有提升之外，更重要的是保证了向前兼容。

##3、 WPA2一统天下   

2008年WPA的漏洞被发现，即TKIP exploit,该漏洞通过改变验证组件无法修正，唯一的解决方法是用AES替代TKIP。WPA2应运而生，这就要求硬件进行改变。TKIP可以作为WPA2的可选项。   
802.11i和WPA2指的是同一件事情，这样一来，在现有的硬件上可以用统一的的加密方式-AES，这正是IEEE标准下所谓的强健网络。
作为当前无线安全的标准，802.11i/WPA2又像极了当年的WPA准标准，仅在一些细微处有所不同。随着802.11i完成最终的业界应用，现有的硬件将AES加密算法作为内置算法。
WPA2相比WPA有如下两大改变：

 1. AES加密算法替代WEP被内置进硬件
 2. 加强了完整性检查  
     
###3.1、802.11i与802.1x     

802.11i采用802.1x保证密钥的建立与验证。EAPoL虽然本身是基于有线设计的，但是很容易适用于无线。802.1X相比之前的验证方法的最大不同之处在于验证过程发生在station和RADIUS server之间，而不是station和AP之间。由于AP不参与密钥产生，所以session key（由PTK导出）和每包密钥需要和AP共享，这些密钥由一个安全通道TLS传输。  

图2描述了多种通信流，用不同的颜色予以标记。EAPoL通信由黑色标记，而EAP握手由蓝色区分。RADIUS采用绿色，TLS则使用了橙色。从该图可以清晰地看到station、AP以及RADIUS server之间的交互流程。

![图片](/assets/images/wireless_brief_fig_2.png)  

<center>Figure 2. 802.1X in Wireless</center>

###3.2、源自RADIUS的TLS隧道    

802.11i利用802.1X不仅建立密钥同时也完成了验证， 相应地，RADIUS服务器也通过该途径向client验证其自身。这条从RADIUS服务器端到客户端的隧道提供了安全通信的机制，从而保证了验证过程中的密钥交互。

###3.3、WPA2的安全性  

WPA2也同样包含两类组件，一是基于AES的加密组件，二是基于PSK或者是RADIUS的验证组件。但并不意味着WPA2就没有漏洞，比如可以通过错误配置PEAP验证组件进行攻击。
   

##4、 后记

尽管业界对于802.1X仍然有一些争议，但其已经成为主流的安全无线验证机制并被广泛接受。总之，802.1X囊括了安全密钥交换、密钥轮转、主密钥保护以及AES加密等，成为目前一种具有单项优势的技术。



  [1]: http://securityuncorked.com/2008/08/history-of-wireless-security/

---