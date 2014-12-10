---
layout: post
title: "Web Application Layer"
categories:
- technology  
tags:
- Docker


---

# Web应用的层次结构  

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
