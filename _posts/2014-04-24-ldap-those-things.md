---
layout: post
title: "LDAP those things"
categories:
- technology  
tags:
- LDAP
- AD
- database


---


## LDAP 概述  

由DAP发展而来，相比DAP，其基于TCP/IP，在功能上有所减少。LDAP本质上就是一种定义了如何访问目录数据的方法的协议，同时定义了在目录服务中数据是如何表达的。  
对于数据操作，LDAP只定义了数据如何加载至目录服务和如何从目录服务导出数据，并未定义如何存储数据和操作，厂家可以有自己的back-end的具体实现，比如OpenLDAP提供了选择back-end数据库的支持。事实上很多商业DB都提供了LDAP视图。  
  
###1. LDAP与DB  

LDAP可以看做是一种读优化的数据库。LDAP Server可以基于multi-master架构，也可以基于Multi-master架构。  
1.	LDAP提供了一种标准化的远程和本地数据的访问方法。可以在不影响外部数据访问接口的前提下替换LDAP的实现。  
2.	由于标准化了数据访问，LDAP客户端和服务器端可以独立开发。  
3.	可以不影响外部数据访问的前提下转移数据至多个地点。  

###2. LDAP信息模式  

LDAP是面向对象的数据库，定义了基础原语（read/delete/modify），以继承性对象的方式表示数据。  
  
###3. LDAP与AD  

LDAP提供了一种可以查询和修改目录服务的应用协议。AD本质就是一种数据库的系统，在window环境中提供验证、目录、策略和其他服务。   
LDAP是一个标准，AD则是微软的基于目录服务器的一中LDAP实现。

