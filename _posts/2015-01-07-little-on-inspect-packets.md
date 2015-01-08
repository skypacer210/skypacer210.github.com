---
layout: post
title: "Little on Inspect Packets"
categories:
- technology
tags:
- WPA
- PSK
- Wireshark


---
  
## WPA/WPA2-PSK抓包解密  

对于基于PSK验证方法的网络报文，可以直接利用抓包工具进行解密。

###1. 获取解密密钥  

由于我们的目的在于分析加密报文，因此需要获取加密数据的密钥。简单的方法可以利用Wireshark的WPA Pre-shared Key Generator，原理如下：    
* 根据PSK和SSID经过伪随机函数进行哈希计算得到PMK   
* 根据PMK和四步握手第一步AP发过来的随机数算出PTK      
* PTK则包含了最终用于加密数据的密钥    

![图片](/assets/images/psk/psk_1.png)

###2. 利用Wireshark解密报文  

原始数据：   
![图片](/assets/images/psk/psk_2.png)

在Wireshark中针对IEEE802.11选择解密数据，添加上步计算所得密钥保存即可：   

![图片](/assets/images/psk/psk_3.png)

这时候我们可以看到数据已经变为明文：         
![图片](/assets/images/psk/psk_4.png)

## SSL/TLS抓包解密

  

  