---
layout: post
title: "Authorization Code flow in OAuth 2.0"
categories:
- technology
tags:
- OAuth
- Authorization


---


# OAuth解决什么问题  

##1、场景  

某孕妇Mrs.Lily向她的私人医生Dr.Michael申请做胎心监护，Dr.Michael了解到Mrs.Lily在医院A.Hospital做过类似的检查，因此Dr.Michael希望访问A.Hospital的在线系统获取Mrs.Lily的相关数据，最直接的方法就是向Mrs.Lily索要其在A.Hospital在线系统用户名和密码。  
那么问题来了，该方法会造成哪些潜在安全威胁？    

* Dr.Michael会保存用户Mrs.Lily的密码，很不安全  
* Dr.Michael拥有了获取用户Mrs.Lily在A.Hospital在线系统的所有资料的权利，这样用户无法限制Dr.Michael的授权范围和时间  
* Mrs.Lily只有修改其密码，才能收回赋予Dr.Michael的权力，但同时也使得其他获得Mrs.Lily授权的访问应用失效  

  
##2、OAuth的特点  

OAuth解决了上述问题，即定义了一种允许一个APP访问用户相关的其他APP的验证机制。基本思路如下： 
比如用户可以利用QQ账号登陆其他APP。  
1.	OAuth在“客户端”与“服务提供商”之间，设置了一个授权层（authorization layer）。 “客户端”不能直接登陆和访问“服务提供商”，只能登陆授权层，这样就把用户和客户端分隔开来。  
2.	“客户端”不在使用密码，而是令牌(token),可以在登陆的时候指定授权范围和有效期限。  
3.	“客户端”登陆授权层之后，“服务提供商”根据令牌的授权范围和有效期限，以决定向客户开放哪些信息。  


# OAuth工作流程

待续。

# 参考文献  

1.	[oauth-20-for-my-ninth-grader](http://architecture-soa-bpm-eai.blogspot.com/2012/08/oauth-20-for-my-ninth-grader.html)  
2.	[liboauth in C](http://liboauth.sourceforge.net/)



------