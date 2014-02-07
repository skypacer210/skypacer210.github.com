---
layout: post
title: 《APIs with an Appetite》读后感  
categories:
- Read
tags:
- 


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

##结论  

API设计应该是一个迭代的过程，首先你提供你认为用户需要的，同时流出后门。然后观察用户在使用过程中利用后门所做的操作，再在API全集中添加给API，依次类推。