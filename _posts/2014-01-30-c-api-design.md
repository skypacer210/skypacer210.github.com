---
layout: post
title: 《APIs with an Appetite》读后感  
categories:
- technology
tags:
- Design Pattern


---
##问题提出  

设计一个数据库的API，可供不同的WEB前端服务存储和获取数据。一些存储图像，一些存储文本、声音、图像等，如何控制API对数据的上述访问方法？  

##解决方案  

###1、瘦API  

void *do_as_i_say(...)  ，其参数是void *  

函数的目的在于提高访问复杂软件的接口，通过隐藏复杂性，提高软件的易用性, 但是该API并不能直观的告诉调用者如何使用。

###2、Fat APIs  

{% highlight c %}  

int store_jpeg_image(string name, image data);
image get_jpeg_image(string name); 
int delete_jpeg_image(string name); 
int store_gif_image(string name, image data); 
image get_gif_image(string name); 
int delete_gif_ image(string name); 
int store_tiff_image(string name, image data); 
image get_tiff_image(string name); 
int delete_tiff_image(string name); 
int store_text(string name, string data); 
string get_text(string name); 
int delete_text(string name);  

{% endhighlight %} 

这里走向了另外一个极端，这里有几个问题：

-	很多信息应该作为参数，而不是一个个罗列为API
-	尽量提供少的名字和用法，因为人类的记忆有限，不可靠记住如此多的用法

###3、Good Example
Unix的文件IO操作为我们提供了一个良好的示例，该接口只有五个API：open、close、read、write、ioctl，其中最重要的API是ioctl,它为程序提供了访问系统底层的能力，以备为设计者预先没有想到的需要，相当于提供了一个后门。  
  

##奇淫巧计  

###1、instance  

该概念类似面向对象设计中的对象，用于维护一个实体，即可以创建、查找、销毁一个Instance。

###2、Cookie  

cookie的含义很广，在分层设计中，cookie可以用于保存上层的数据，包括配置数据和配置行为(往往通过回调函数定义)，这样底层可以无需知道上层的配置细节，直接调用其配置方法即可。  

###3、Callback  
  
-	Callback可以实现底层调用上层，这里无需赘述  

-	Callback可以用于实现异步通知机制，比如在一个异步事件通知系统中，可以将事件处理方法和ID组成事件对象加入事件队列，其中事件处理方法采用callback函数，在事件注册时定义，实现处理系统轮询该事件队列，逐个调用，当然也可以加入优先级以改变响应次序。

  

##结论  

API设计应该是一个迭代的过程，首先你提供你认为用户需要的，同时流出后门。然后观察用户在使用过程中利用后门所做的操作，再在API全集中添加给API，依次类推。