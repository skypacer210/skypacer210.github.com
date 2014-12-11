---
layout: post
title: "Web Application Layer"
categories:
- technology  
tags:
- Docker


---

#1 HTTP Basic  

##1、HTTP资源

HTTP用于获取服务器的资源，资源的种类称为MIME，资源的表达方式URI（统一资源标识符）。URI有两种形式，URL和URN：    
* URL精确说明了资源的位置及如何访问它，包括方案（HTTP、FTP），服务器因特网地址和WEB服务器上的某个资源。  
* URN（统一资源名）目的是为了使用同一个名字通过多种网络协议来访问资源。
URI几乎都是URL。

##2、HTTP事务  

一个事务包含一条请求加响应的响应。HTTP方法：  
* GET：从服务器向客户端发送命令资源     
* POST：将客户端数据发送至一个服务器网关应用程序；  
* PUT：将来自客户端的数据存储至一个命名的服务器资源中去；  
* DELETE：从服务器张删除命名资源。  

HTTP客户端的应用程序为了完成某项任务可以发布多个HTTP事务，而请求的资源可以位于不同的HTTP服务器端。比如JAVA小程序、图像等。

##3、HTTP报文   

HTTP报文都是纯文本，不是二进制，包含三部分：   
* 起始行（start lines）： 典型首部字段比如Content-Type: text/html; charset=utf-8    
* 首部 (Header)   
* 主体 (entity body)

##4、HTTP报文传输

基于TCP/IP传输，因此需要目的IP地址和端口号。HTTP通过URL告诉TCP层IP地址和端口号。其中URL可以直接包含IP地址或者通过DNS查询得到，端口号默认为80。  
TCP为HTTP提供了一条可靠的比特传输通道，从TCP一端填入的字节会从另外一端以原有顺序、正确的传送出来。  

![图片](/assets/images/tcp_http.jpg)
 
HTTP事务的性能在很大程度上决定上取决于底层TCP通道的性能。HTTP时延主要由TCP时延引起。需要考虑如下因素：  
* TCP连接建立握手；  
* TCP慢启动拥塞控制；  
* 数据汇聚的Nagle算法；  
* 用于捎带确认的TCP延迟确认算法；    


##5、HTTP连接管理
在HTTP层的优化方法：    
* 并行化连接：HTTP客户端打开多个连接，并行执行HTTP事务，即每个事务对应自己的。  
* 持久化连接：通过头部的connection字段完成，client填充keep-alive，如果server同意，就会回复同样的字段和数据内容。  
* 管道化连接  

##6、HTTP验证框架  
HTTP的验证框架基于Challenge/Response验证。具体而言，基本的HTTP验证通过GET方法，在首部利用WWW-Authentication字段（GET Response）和Authorization字段（GET Request）完成交互。 

![图片](/assets/images/http_auth.jpg)
   
根据client在Authorization字段的对身份凭证（比如用户名密码）的处理方式不同，可以引申出诸多验证方式。    
* 基于BASE64编码的基本认证；
* 基于HASH算法的摘要认证；
* 另外，为了防止重放攻击，可以再客户端和服务器端都引入随机数。  

但是其其缺点也很明显：
* 缺乏对客户端的认证；
* 缺乏对服务器端的认证；
* 无法保证客户端和服务器端数据的完整性；
* 无法保证客户端和服务器端之间数据的私密性；

HTTPS应运而生。

#2 Web应用的层次结构  

Web的基本原理即client向server发起HTTP Requst, server向client回应HTTP Response。client不仅仅局限于浏览器，也不限于人，更一般的情况下client向第三方发起HTTP Requst,得到第三方的HTTP Response。

##1、Static website

nginx 1.5 + modsecutity + openssl + bootstrap 2

##2、Background workers  

Phython 3.0 + celery + pyedis + libcurl + ffmpeg + libopencv + nodejs + phantomjs

##3、User DB  

postgresql + pgv8 + v8  

##4、Anaytics DB  

hadoop + hive + thrift + OpenJDK  

##5、Queue  

随着HTTP交互的数据量增长，不可避免会出现请求无法响应的情况，例如Twitter的在宕机时会出现fail whale图片。为了解决这个问题，引入消息队列。  
* 可以向第三方应用发送HTTP Request;  
* 可以更快的响应浏览器；  
* 如果第三方应用失败，不会影响到client；  
* 还可以发起重试；  
* 可以向对个第三方应用推送数据。  

目前的消息队列库的实现可以基于多种语言，包括Ruby、Python、Java、Erlang、Scala、C#、Lua、 PHP 等等。  

消息队列遵循的协议有STOMP、AMQP和Memcache。  
	
消息队列相关的软件包括RabbitMQ、Stompserver、ZeroMQ、Morbid、ApacheMQ、Starling、Beanstalk、Kestrel以及MemcacheQ。  

Redis + redis-sentinel

1.	Kafka: A Distributed Messaging System
2.	Redis: A Simple Key-Value Store

##6、API endpooint  

Python 2.7 + Flask + pyredis + celery + psycop + postgresql-client
