---
layout: post
title: "Token VS Cookie"
categories:
- technology
tags:
- token
- cookie
- http authentication


---

#1 概述  
Client请求server端应用时，server端的验证方法有两种：  
1.  最常见的cookie-based Authentication：  Serve端发送给client的cookie，这样client每次登陆的时候server只需验证cookie即可。Cookie可以存在内存或者硬盘中。  
2.  基于token的验证：
client收到签发的token之后，以后每次登陆需要发送至server端。
  

#2 Token相比于cookie的优点
* Cross-domain/CORS:  
Cookie + CORS 在不同的域之间工作的不是很好，token方法允许你利用AJAX调用至任何server和任何域，因为你利用HTTP头传递用户信息。
比如说SAML Tokens, SAML(Security Assertion Markup Language) tokens是一种标准化的基于XML的token，可以提供跨平台交互，而且client和server无需再同一个安全域内。  
* 无状态：  
Server端无需维护一个session的存储，token本身是一个包含所有用户信息的实体。Cookie则存在状态，并且必须保存在client端。
* CDN：  
Sever端可以提供所有的APP特性，只需通过API访问CDN即可。
* 解耦：  
无需绑定某个特定的验证方案，token可以在任何地方产生，因为你能够在任何地方以一种方式调用这些API。  
* 性能  
计算HMAC-SHA256似乎效率更高
* 标准化：  
你的API可以接受标准的JWT(JSON Web Token)。


#3 Token存储  
Token可以存储在三个地方：session storage/local storage/client side cookie。 如果存储在cookie中，利用了cookie的存储机制，而不是验证机制。  

#4 Token的生命周期  
Token和cookie都有生命周期。  
1.	对于cookie而言，控制器生命周期有如下选择：  
* Session cookie在浏览器关闭后被销毁；  
* 可以实现一个server端的检查，避免过期；  
* Cookie可以在生命周期内是永久的，即浏览器关闭后不销毁。  
2.	对Token而言，一旦token过期，你需要一个新的，可以用如下的更新机制：  
* 验证留的token；  
* 检查用户是否仍旧存在或者访问权限被吊销；  
* 签发一个更新了有效期的token

#5 Token缺点
1.	对于大公司而言，token可能变得很大  
2.	开发者的APP或者API变得更加复杂  
3.	维护的时间成本较高    
#6 Token VS Certificate VS Ticket  
三者都是引入了第三方机构,以window的验证模式为例：  
1.	Token：SAML Token，引入STS(Security Token Service)作为第三方机构  
2.	Certificate：引入CA，即X.509 PKI架构  
3.	Ticket：基于Kerberos的验证架构，引入TGS和AS作为KDC  
