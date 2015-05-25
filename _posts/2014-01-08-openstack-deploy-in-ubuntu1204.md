---
layout: post
title: "Ubuntu下OpenStack配置小记"
categories:
- technology  
tags:
- deploy
- openstack


---
本文由[skypacer](http://skypacer210.github.io/)根据[OpenStack Installation Guide for Ubuntu 12.04 (LTS)](http://docs.openstack.org/trunk/install-guide/install/apt/content/)在自己的环境中实际部署情况整理而成。转载请注明出处！


###1、安装准备  

#### 网络设置  

-	安装双网卡：通过vsphere client安装两个网卡，一个内网，一个外网。  
	修改文件/etc/network/interfaces如下：    

<pre><code>
auto lo
iface lo inet loopback 
# The primary network interface
# Internal network
auto eth0
#iface eth0 inet dhcp
iface eth0 inet static
address 10.151.122.138
netmask 255.255.255.0
gateway 10.151.122.1
dns-nameservers 10.151.122.5 10.151.120.21 10.140.2.48
dns-search wyse.com
# This is an autoconfigured IPv6 interface
iface eth0 inet6 auto

# External network
auto eth1
iface eth1 inet static
address 10.151.120.190
netmask 255.255.255.0
gateway 10.151.120.1
dns-nameservers 10.151.122.5 10.151.120.21 10.140.2.48
dns-search wyse.com  
 
</code></pre>  

-	修改host  
修改/etc/hostname文件如下：   
<pre><code>  
controller
</code></pre>     

修改/etc/hosts文件如下：  

<pre><code>  
127.0.0.1       localhost
10.151.122.138  controller
10.151.122.139  compute1

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
</code></pre> 


-	重启网络：service networking restart  
-	测试网络：通过hostname命令检查主机名设置；通过ping检查主机解析  

####1.2、安装数据库    

-	apt-get install python-mysqldb mysql-server
-	修改/etc/mysql/my.cnf：将mysql的绑定地址改为内网IP：1
	bind-address            = 10.151.122.138  
-	重启MYSQL：service mysql restart
-	删除匿名用户：   
<pre><code>  
	# mysql_install_db
	# mysql_secure_installation  
</code></pre> }

####1.3、安装时间服务器  
    
-	安装NTP服务器用于同步时间：apt-get install ntp
-	创建各自密码：暂时不考虑安全性，为了方便暂时保存在.bashrc中的环境变量中

###2、Identity services配置

<a id='single_res' name='single_res'> </a>

单一责任原则，即一个类只能有一种原因去驱动改变，该原则的是同open-closed原则有异曲同工之妙：尽量避免现有代码的修改。当违反了单一原则的时候，一个模块有多种原因需要去修改。更糟的是，多个不同的责任集中于一个模块当中会耦合在一起，使得测试与修改变得非常复杂。  

单一责任原则本质是集中性，便于进行多层抽象，而不仅仅是一个过程上下文，简单地讲类替换为函数将有利于我们在此原则基础上分析算法。  

####2.1、验证机制  

####2.2、定义用户、租户和角色  

由于在/etc/keystone/keystone.conf中定义了admin_token，这是一个keystone和other openstack services的预共享密钥，在创建租户时会用到。  

-	通过设置两个环境变量指定token和identity service运行的路径  
	export OS_SERVICE_TOKEN=ADMIN_TOKEN  
	export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
-	创建租户  

keystone tenant-create --name=admin --description="Admin Tenant"  
<pre><code>  
 
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |           Admin Tenant           |
|   enabled   |               True               |
|      id     | 5e7178983104482a92433f55b09c6dcd |
|     name    |              admin               |
+-------------+----------------------------------+  
</code></pre> }

keystone tenant-create --name=service --description="Service Tenant"    

<pre><code> 
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |          Service Tenant          |
|   enabled   |               True               |
|      id     | 2b1ee1230ed34dcf9b73941f9e6b89c1 |
|     name    |             service              |
+-------------+----------------------------------+
</code></pre> }  

-	创建管理员用户  

keystone user-create --name=admin --pass=ADMIN_PASS --email=yangyongbupt168@gmail.com  
  
<pre><code>  
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |    yangyongbupt168@gmail.com     |
| enabled  |               True               |
|    id    | d9308be3ba924b94ac051eb8d0750cd6 |
|   name   |              admin               |
+----------+----------------------------------+  
</code></pre> }    

-	创建角色：角色由各种OpenStack服务对应的policy.json文件指定，缺省的策略文件赋予admin角色访问大多数服务的权利。    

<pre><code> 
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|    id    | 372478ee2d2647858d864ce9066acccd |
|   name   |              admin               |
+----------+----------------------------------+  
</code></pre> }    

-	为用户添加角色：用户总是以租户身份登录，角色通过租户被赋予给用户。例如，当用户admin以admin租户的身份登录时，赋予其admin角色。
	keystone user-role-add --user=admin --tenant=admin --role=admin    

####2.3、定义services和API endpoints  

-	为Identity services创建一个服务实体    

keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"    

<pre><code>   
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |    Keystone Identity Service     |
|      id     | 7d36ad0a2c0f4ec893b1fedc95d5e9de |
|     name    |             keystone             |
|     type    |             identity             |
+-------------+----------------------------------+  
</code></pre> }    
  
-	利用返回的service ID（随机产生）创建API endpoint，其实就是创建内外网API等URL。
	
	keystone endpoint-create \
	--service-id=7d36ad0a2c0f4ec893b1fedc95d5e9de \
	--publicurl=http://controller:5000/v2.0 \
	--internalurl=http://controller:5000/v2.0 \
	--adminurl=http://controller:35357/v2.0  

<pre><code> 
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |   http://controller:35357/v2.0   |
|      id     | 42eb21997c244fa480320fa8bd4102dc |
| internalurl |   http://controller:5000/v2.0    |
|  publicurl  |   http://controller:5000/v2.0    |
|    region   |            regionOne             |
|  service_id | 7d36ad0a2c0f4ec893b1fedc95d5e9de |
+-------------+----------------------------------+
</code></pre> }   

####2.4、验证Identity Services安装  
  
本质上是利用user/token验证绑定user/password的过程，期望通过user/password 查找token。  

-	删除临时环境变量

unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT  

-	基于用户的验证：用admin的user/passord请求验证token  

keystone --os-username=admin --os-password=ADMIN_PASS \
	--os-auth-url=http://controller:35357/v2.0 token-get  
  
<pre><code>

+----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Property |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
+----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 2014-01-10T03:29:38Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|    id    | MIIC8QYJKoZIhvcNAQcCoIIC4jCCAt4CAQExCTAHBgUrDgMCGjCCAUcGCSqGSIb3DQEHAaCCATgEggE0eyJhY2Nlc3MiOiB7InRva2VuIjogeyJpc3N1ZWRfYXQiOiAiMjAxNC0wMS0wOVQwMzoyOTozOC4xNTMwNDAiLCAiZXhwaXJlcyI6ICIyMDE0LTAxLTEwVDAzOjI5OjM4WiIsICJpZCI6ICJwbGFjZWhvbGRlciJ9LCAic2VydmljZUNhdGFsb2ciOiBbXSwgInVzZXIiOiB7InVzZXJuYW1lIjogImFkbWluIiwgInJvbGVzX2xpbmtzIjogW10sICJpZCI6ICJkOTMwOGJlM2JhOTI0Yjk0YWMwNTFlYjhkMDc1MGNkNiIsICJyb2xlcyI6IFtdLCAibmFtZSI6ICJhZG1pbiJ9LCAibWV0YWRhdGEiOiB7ImlzX2FkbWluIjogMCwgInJvbGVzIjogW119fX0xggGBMIIBfQIBATBcMFcxCzAJBgNVBAYTAlVTMQ4wDAYDVQQIDAVVbnNldDEOMAwGA1UEBwwFVW5zZXQxDjAMBgNVBAoMBVVuc2V0MRgwFgYDVQQDDA93d3cuZXhhbXBsZS5jb20CAQEwBwYFKw4DAhowDQYJKoZIhvcNAQEBBQAEggEAAS-Fdhv74-kaGYHDuVnJo3ZHhfQ0sdXtdJE68V86gB3TnRMBvDizYaaBlUEDkVLEsOTf-SwO+2fVDnEQ82eARFwXi5RgGUW-rSPX-DU9iJob0ajc4Pn7tRioEzjUByON+8kYge7V5hmowc4e+qM7vvbUcLkTC3eKkF4b4xNg7JSCEMeRCIPSDlqOR3a-S5jODUSdQlfkcZyjocQjcQdaEBW8Rm+bv4TTVYi2PotbqoZQHGvVPg45cjeFnLFSs9XVCkFMx9rUgXmj4+FdV61t6wH0goJ5LriIHKOlKD0dp6V8RX0p1hpmFoD1dI+yDFD68dmmUV5t6gtluyZA5dJNBw== |
| user_id  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           d9308be3ba924b94ac051eb8d0750cd6                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
+----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

</code></pre> }   

 

这是一个包含用户ID的token paired。  

keystone --os-username=admin --os-password=ADMIN_PASS \
	--os-tenant-name=admin --os-auth-url=http://controller:35357/v2.0 token-get  
  
<pre><code>

+-----------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|  Property |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  Value                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
+-----------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|  expires  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           2014-01-10T03:41:25Z                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
|     id    | MIIEsgYJKoZIhvcNAQcCoIIEozCCBJ8CAQExCTAHBgUrDgMCGjCCAwgGCSqGSIb3DQEHAaCCAvkEggL1eyJhY2Nlc3MiOiB7InRva2VuIjogeyJpc3N1ZWRfYXQiOiAiMjAxNC0wMS0wOVQwMzo0MToyNS45NjI4MzUiLCAiZXhwaXJlcyI6ICIyMDE0LTAxLTEwVDAzOjQxOjI1WiIsICJpZCI6ICJwbGFjZWhvbGRlciIsICJ0ZW5hbnQiOiB7ImRlc2NyaXB0aW9uIjogIkFkbWluIFRlbmFudCIsICJlbmFibGVkIjogdHJ1ZSwgImlkIjogIjVlNzE3ODk4MzEwNDQ4MmE5MjQzM2Y1NWIwOWM2ZGNkIiwgIm5hbWUiOiAiYWRtaW4ifX0sICJzZXJ2aWNlQ2F0YWxvZyI6IFt7ImVuZHBvaW50cyI6IFt7ImFkbWluVVJMIjogImh0dHA6Ly9jb250cm9sbGVyOjM1MzU3L3YyLjAiLCAicmVnaW9uIjogInJlZ2lvbk9uZSIsICJpbnRlcm5hbFVSTCI6ICJodHRwOi8vY29udHJvbGxlcjo1MDAwL3YyLjAiLCAiaWQiOiAiNzMzOWEwZDY1NDBlNGM0MGI4YWE4NGI1NDEzODNlMTYiLCAicHVibGljVVJMIjogImh0dHA6Ly9jb250cm9sbGVyOjUwMDAvdjIuMCJ9XSwgImVuZHBvaW50c19saW5rcyI6IFtdLCAidHlwZSI6ICJpZGVudGl0eSIsICJuYW1lIjogImtleXN0b25lIn1dLCAidXNlciI6IHsidXNlcm5hbWUiOiAiYWRtaW4iLCAicm9sZXNfbGlua3MiOiBbXSwgImlkIjogImQ5MzA4YmUzYmE5MjRiOTRhYzA1MWViOGQwNzUwY2Q2IiwgInJvbGVzIjogW3sibmFtZSI6ICJhZG1pbiJ9XSwgIm5hbWUiOiAiYWRtaW4ifSwgIm1ldGFkYXRhIjogeyJpc19hZG1pbiI6IDAsICJyb2xlcyI6IFsiMzcyNDc4ZWUyZDI2NDc4NThkODY0Y2U5MDY2YWNjY2QiXX19fTGCAYEwggF9AgEBMFwwVzELMAkGA1UEBhMCVVMxDjAMBgNVBAgMBVVuc2V0MQ4wDAYDVQQHDAVVbnNldDEOMAwGA1UECgwFVW5zZXQxGDAWBgNVBAMMD3d3dy5leGFtcGxlLmNvbQIBATAHBgUrDgMCGjANBgkqhkiG9w0BAQEFAASCAQA3-Ce1g6lmH707ibg+NDRTn2xiTds9SkWUvpHNTPSDvbnW5-eoSSE+Ty7EVMb5XFkCZlDYHHKuDRfdG9reFfQq9qVcXzS4R8FAxJF8V9Nfx4z76l1yGyZxRbn566uKPYc2BUlp3ZydeP7eX+k54GCNkPj+dMgCcr4weSCfuIXaVejKVsKaz1fJu7AmypeDa24fTjseJUTzqUNkCLp4+RkVMsb2SaQDgNSSJf890m+TB70zPJN63IasJvH8DnAzZ2G2F2MIw+zBAQ7kTl7lskRKOGVtUIZxxxb8TGWP+Y34VrAHLoUoXsP+fQhIlv-dKfuNt5qoWe0AlnBkrwQl3HK1 |
| tenant_id |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     5e7178983104482a92433f55b09c6dcd                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
|  user_id  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     d9308be3ba924b94ac051eb8d0750cd6                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
+-----------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

</code></pre> }   
  

期望返回一绑定租户的token。  


-	脚本操作：上述手动操作的替代方法  
	
<pre><code> 
#!/bin/bash

export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://controller:35357/v2.0  

</code></pre> }   

-	确保admin账户已经被验证

keystone user-list

<pre><code> 
+----------------------------------+-------+---------+---------------------------+
|                id                |  name | enabled |           email           |
+----------------------------------+-------+---------+---------------------------+
| d9308be3ba924b94ac051eb8d0750cd6 | admin |   True  | yangyongbupt168@gmail.com |
+----------------------------------+-------+---------+---------------------------+
</code></pre> 

###3、OpenStack clients安装与配置  

通过命令行客户端进行API调用，可以设置在脚本中自动执行。在OpenStack内部，可以运行cURL命令执行内嵌API请求。OpenStack是一种基于HTTP协议REST风格的API，包括方法、URI、介质类型和响应代码。  
每个Openstack服务都有自己的命令行客户端。

####3.1、基于python的开源客户端  
   
-	ceilometer：度量openStack
-	cinder：创建和管理volume
-	glance:创建和管理图片
-	heat：从模板登录和管理stack
-	keystone:创建管理用户、租户、角色和密钥等
-	neutron：为客户端server配置网络
-	nova：管理图片、句柄和flavors
-	swift：统计Object Storage服务的元数据
-	充电


###4、验证Identity Services安装     

####4.1、安装Image Service  

-	注册Image服务和Identity服务，这样其他的OpenStack服务可以定位到它  

keystone service-create --name=glance --type=image \
	--description="Glance Image Service"

<pre><code> 
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |       Glance Image Service       |
|      id     | 21b15dc9b1d846149883e953af08d906 |
|     name    |              glance              |
|     type    |              image               |
+-------------+----------------------------------+
</code></pre>  
  
-	利用serveice id创建endpoint

keystone endpoint-create --service-id=21b15dc9b1d846149883e953af08d906 --publicurl=http://controller:9292 --internalurl=http://controller:9292 \
	--adminurl=http://controller:9292

<pre><code> 
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |      http://controller:9292      |
|      id     | 9cf77dd344514215812b1391dbf2b498 |
| internalurl |      http://controller:9292      |
|  publicurl  |      http://controller:9292      |
|    region   |            regionOne             |
|  service_id | 21b15dc9b1d846149883e953af08d906 |
+-------------+----------------------------------+
</code></pre>  

####4.2、安装Image Service
  
验证安装：  

-	上传Image

root@controller:/home/fyang/project/images# glance image-create --name="CirrOS 0.3.0" --disk-format=qcow2 --container-format=bare --is-public=true < cirros-0.3.0-x86_64-disk.img   

root@controller:/home/fyang/project/images# glance image-create --name="CirrOS 0.3.0" --disk-format=qcow2 --container-format=bare --is-public=true < cirros-0.3.0-x86_64-disk.img  
-	显示Image列表  

glance image-list

<pre><code> 
+--------------------------------------+--------------+-------------+------------------+---------+--------+
| ID                                   | Name         | Disk Format | Container Format | Size    | Status |
+--------------------------------------+--------------+-------------+------------------+---------+--------+
| 59c63869-ce77-4647-90b5-782d258e3354 | CirrOS 0.3.0 | qcow2       | bare             | 9761280 | active |
| 05f2ae82-0926-4b8a-9d11-dfc99d8ee845 | CirrOS 0.3.1 | qcow2       | bare             | 9761280 | active |
+--------------------------------------+--------------+-------------+------------------+---------+--------+
</code></pre>  

###5、计算控制服务安装与配置  

####5.1、概述  

####5.1、安装Image Service  

-	创建计算服务表：

nova-manage db sync  

<pre><code>
	2014-01-09 17:45:46.814 7918 INFO migrate.versioning.api [-] 215 -> 216... 
2014-01-09 17:45:46.837 7918 INFO migrate.versioning.api [-] done
</code></pre>  

-	创建用户

keystone user-create --name=nova --pass=NOVA_PASS --email=nova@example.

<pre><code>
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |          nova@example.           |
| enabled  |               True               |
|    id    | 26e1951aa2264a11800c68cc58dbce59 |
|   name   |               nova               |
+----------+----------------------------------+
</code></pre>  

-	充电

root@controller:/home/fyang/project/images# keystone service-create --name=nova --type=compute --description="Nova Compute service"  
  
<pre><code>
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |       Nova Compute service       |
|      id     | 0d67dd130ebc4de2ba8e54ac241e92d0 |
|     name    |               nova               |
|     type    |             compute              |
+-------------+----------------------------------+

</code></pre>  

-	创建endpoint

root@controller:/home/fyang/project# keystone endpoint-create --service-id=0d67dd130ebc4de2ba8e54ac241e92d0 --publicurl=http://controller:8774/v2/%\(tenant_id\)s --internalurl=http://controller:8774/v2/%\(tenant_id\)s --adminurl=http://controller:8774/v2/%\(tenant_id\)s  

<pre><code>
+-------------+-----------------------------------------+
|   Property  |                  Value                  |
+-------------+-----------------------------------------+
|   adminurl  | http://controller:8774/v2/%(tenant_id)s |
|      id     |     21f036edca9547e196b5976a4c31f561    |
| internalurl | http://controller:8774/v2/%(tenant_id)s |
|  publicurl  | http://controller:8774/v2/%(tenant_id)s |
|    region   |                regionOne                |
|  service_id |     0d67dd130ebc4de2ba8e54ac241e92d0    |
+-------------+-----------------------------------------+  
</code></pre>  
