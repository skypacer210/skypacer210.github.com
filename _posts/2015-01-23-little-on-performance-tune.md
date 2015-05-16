---
layout: post
title: "浅谈VDI优化"
categories:
- technology
tags:
- performance tune
- TLS
- memory pool


---


### 概述 ###

VDI优化涉及到诸多系统的方面，主要包括：  

![图片](/assets/images/perf/perf_framework.jpg)


### TLS层的优化点 ###

TLS连接的建立和关闭过程如图：  

![图片](/assets/images/perf/SSL_connect.png)

在TLS中频繁分配释放的内存主要有以下几类：   
- 哈希描述符和哈希值;  
- 密钥，比如masterkey为48字节;  
- 典型的小块内存，一般为64字节和96字节;    

Memory Pool的主要设计思路：  
- 每块内存都看做一个对象，预先分配好一定数量的对象，即所谓的内存池；  
- 定义get和put方法，一般成对出现，这样保证了预分配的对象数量可控；    
- 内存池内的每个对象用单向链表维护，具体操作见图；  

![图片](/assets/images/perf/mem_pool.jpg)


对SSL的握手过程进行对比测试，结果表明在使用mem pool的性能提升为8%左右，似乎并不理想，看来在SSL握手中此类内存的分配释放并不是影响性能的主要因素。 


