---
layout: post
title: "Issue about TCP Read Timeout in SSL session"
categories:
- technology 
tags:
- SSL/TLS
- select
- HashTable


---

## 从SSL会话中出现TCP超时说起  

最近将工作中碰到的一个问题记录下，以备查找。该问题的现象是程序在连接某一个APP时失败，日志显示TCP读超时，但是连接其他APP都成功。    

初步调试发现，此时client与server已经建立SSL连接，但是在读取APP数据时select超时，但为什么单单在连接这个APP的时候出现超时呢？ 

首先看SSL连接的建立过程，如图1所示：    
![图片](/assets/images/SSL/SSL_exchange.png)

1.	第一步服务器端创建监听socket等待客户端连接。
2.	第二步客户端创建SSL上下文数据结构，包括支持的cipher种类，证书以及SSL版本等。
3.	第三步完成TCP端口的建立。客户端调用connect()发起TCP握手，即SYN，SYN/ACK和ACK这三次握手）。服务器端调用accept()响应客户端的请求。
4.	第四步客户端和服务器端都会创建一个session，该对象基于SSL上下文，并绑定其至socket。具体实现就是以socket为索引建一张表，该表维护当前所有的SSL session。既然是session，那就是有状态的，一般是CONNECT_DISABLED、CONNECT_CLOSED、CONNECT_NEGOTIATE和CONNECT_OPEN。
5.	第五步完成SSL层的握手。该过程由客户端发起，主要是负责建立一个安全通道，协商出一个共享秘钥以供后续使用。
6.	在第六步中数据经由前面建好的加密隧道收发。该过程是密文交互。
7.	第七步完成SSL连接关闭，客户端和服务器端都可以通过alert类型的记录报文发起关闭请求。
8.	在正常情况下，TCP关闭由四次握手完成，即FIN/ACK，ACK，FIN/ACK和ACK完成，该过程是明文交互。
9.	客户端和服务器端释放SSL session资源。
10.	服务器返回第三步接受另外的socket。  

注：  
1.	第一、第三和第八步是标准的TCP握手和终止流程，与是否进行SSL连接无关。  
2.	第二、第四、第五、第七和第八步仅仅在SSL连接建立的情况下发生的流程。  
3.	第六和第九步则SSL相关。   


让我们回到数据读取，这里需要说明的是程序采用的事件处理框架。作为SSL客户端，为了能够处理多个SSL连接建立后对多个socket的并发事件处理，引入了select机制，事实上最多支持32个SSL连接。Select作为一种非阻塞IO接口，在这里用于轮询建立的TCP socket (SSL连接基于TCP socket，当然例外情况是DTLS基于UDP) 事件，结果发现没有数据可读。     

进一步查看已经建立的SSL连接表，结构如图2所示。SSL连接表本身由一个数组存储，为了加快SSL连接的查找又引入了一张哈希表，该哈希表采用数组作为散列的桶，每个桶是一个单向链表以解决哈希冲突。      
  
![图片](/assets/images/SSL/SSL_data.png)

当前的SSL连接表显示多个SSL连接都挂在了一个桶的链表上，看起来是发生了哈希冲突，但细想这概率似乎有点高，那就看看为啥这几个SSL连接都搞出来一个key， 发现这里是以TCP socket作为查找key，让我们看看在哈希表插入的时候发生什么。哈希表的插入发生在新建一个SSL连接的时候（每一个SSL连接对应一个TCP socket），当TCP连接建立之后，SSL就以socket作为key插入哈希表当中，很明显这里的socket都弄成一样了，看代码发现程序错将文件描述符当成SSL连接表的查找键，而文件描述符在一些应用中很容易重复获得，直接导致在哈希表中向同一个桶插入SSL连接描述符，进而引发TCP在创建socket之后对应错误，最终导致APP在一个错误的socket上轮询读写事件而超时。  
