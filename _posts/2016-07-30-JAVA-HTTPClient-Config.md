---
layout: post
title: "JAVA HTTP Client Configration"  
image:   
categories:
- JAVA

tags:
- HTTPClient
- keep-alive
- SSLConnectionSocketFactory

---


## 1. JAVA 网络相关的类、接口与异常

**类**

- ConnectionManager
- RegistryBuilder

**接口**

- ConnectionSocketFactory

创建和连接socket；

可以实现为非加密socket:   

```
public class PlainConnectionSocketFactory implements ConnectionSocketFactory {

``` 

也可以实现为加密socket;


也可以继续扩展增加新的方法：

```
public interface LayeredConnectionSocketFactory extends ConnectionSocketFactory {

``` 

**异常**

- ConnectException
- ProtocolException


## 2. 长连接修改


将基于HttpURLConnection类的HTTPClient修改为HttpClient4.3.2： 

- 重写了HttpRequest类的getHttpConnection()方法和HttpResponse类的getResponse()方法，主要将HttpURLConnection类的操作接口替换为HttpClient4.3.2；
- 增加ConnectionManager类，主要基于PoolingHttpClientConnectionManager类完成KeepAlive策略配置、连接池管理与socket超时配置，对于HTTPS服务端验证，目前缺省验证通过；
- 为Client接口增加setConnConfig()方法，客户端可以基于该接口配置如下参数：
	- maxConnNumber：总的最大并发连接数；
	- maxConnPerRoute：每个路由的并发连接数，如果不使用代理，该限制实际上就是单个主机的连接数限制，缺省为2；
	- maxKeepAliveTimeout：设置KeepAlive超时时间，单位为秒；

## 3. 短连接修改


### 3.1 连接重用策略

在httpClient中可以自定义连接重用的策略，定义了一个接口类ConnectionReuseStrategy：

```
public interface ConnectionReuseStrategy {
    boolean keepAlive(HttpResponse var1, HttpContext var2);
}
```

其缺省的实现类为org.apache.http.impl.DefaultConnectionReuseStrategy:   

![DefaultConnectionReuseStrategy](/assets/images/javaclient/default_reuse_impl.png)
<center>Figure 1: DefaultConnectionReuseStrategy </center>   


其中该类似实现了接口类的对外方法keepAlive。当然我们可以自定义一个连接重用策略：

### 3.2 调用连接重用

调用者为org.apache.http.impl.execchain.MainClientExec：

![call_reuse_conn](/assets/images/javaclient/call_reuse_conn.png)
<center>Figure 2: 连接重用调用过程 </center>   
  
- 如果用户没有自定义重用策略，这里会调用缺省的策略，会将connHolder设置为可重用；
- 否则设置为不可重用，解析完毕response之后就会关闭该连接;

连接关闭的调用条件：    

![call_release_conn](/assets/images/javaclient/call_release_conn.png)
<center>Figure 3: 连接释放条件 </center> 


其中内部实现如下：

![release_conn_impl](/assets/images/javaclient/release_conn_impl.png)
<center>Figure 4: 连接释放实现 </center> 


## 3. SSL Cipher Suite配置

![SSLConnectionSocketFactory](/assets/images/javaclient/SSLConnectionSocketFactory.png)    
<center>Figure 5: SSLConnectionSocketFactory结构 </center>

SSLConnectionSocketFactory类可以基于SSLSocketFactory、支持的协议、支持的密码套件进行构造：

![SSLConnectionSocketFactory Flow](/assets/images/javaclient/SSLConnectionSocketFactory_code.png)
<center>Figure 6: SSLConnectionSocketFactory流程 </center>


其中supportCipherSuites作为自定义密码套件在握手过程中检查。