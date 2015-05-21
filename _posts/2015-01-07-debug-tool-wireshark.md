---
layout: post
title: "调试工具杂记之Wireshark篇"
categories:
- technology
tags:
- WPA
- PSK
- Wireshark


---

## 1. 概述 ##

Wireshark作为一个使用频率极高的工具，在日常工作中发挥着不可替代的作用，下面主要结合工作中涉及的无线和安全相关部分展开说明。  
  
## 2. WPA/WPA2-PSK抓包解密 ##  

对于基于PSK验证方法的网络报文，可以直接利用抓包工具进行解密。

### 2.1. 获取解密密钥 ###  

由于我们的目的在于分析加密报文，因此需要获取加密数据的密钥。简单的方法可以利用Wireshark的WPA Pre-shared Key Generator，原理如下：     
 
* 根据PSK和SSID经过伪随机函数进行哈希计算得到PMK   
* 根据PMK和四步握手第一步AP发过来的随机数算出PTK      
* PTK则包含了最终用于加密数据的密钥    

![图片](/assets/images/psk/psk_1.png)

### 2.2. 利用Wireshark解密报文 ###  

原始数据：   
![图片](/assets/images/psk/psk_2.png)

在Wireshark中针对IEEE802.11选择解密数据，添加上步计算所得密钥保存即可：   

![图片](/assets/images/psk/psk_3.png)

这时候我们可以看到数据已经变为明文：         
![图片](/assets/images/psk/psk_4.png)


## 3. SSL/TLS抓包解密 ##

### 3.1. 前提 ### 
  
为了能够对SSL/TLS中的数据报文进行解密，要保证一下几个前提：    

第一：Cipher suit只能选择RSA相关，即必须强制SSL客户端使用RSA Key。如果选择了其他Cipher，比如Ephemeral Diffie-Hellman(称之为DHE) ，Elliptic Curve Diffie-Hellman(称之为TLS_ECDH)。RSA密钥仅仅用于协商而没有用在整个数据传输当中。对于自己的客户端，直接增加对应接口即可；而对于window客户端，可以通过修改注册表进行修改：  

![图片](/assets/images/tool/win_cipher_config.png)  

第二：能够拿到服务器证书对应的私钥，然后将其转化为PEM格式，因为只有私钥才能解密其对应公钥加密的报文。    

第三：确保客户端和服务器之间有一系列完整的通信报文。   
 
### 3.2. Wireshark设置 ###

Wireshark的设置很简单，见下图所示：   

![图片](/assets/images/tool/ssl_decrypt.png)  

配置好，之后就可以对所抓报文进行解密。  


## 4. 数据重放 ##  

在实际工作中，往往遇到无法直接连接服务器的情形，只能通过抓包进行分析。在其中尤以证书验证问题居多，利用Wireshark报文，既可以将server证书串直接导出，也可以以二进制文件格式导出再辅以其他解析工具导出程序进行调试。

这其实涉及到了数据重放的概念，将实际环境当中的运行数据进行本地化调试，具体步骤如下：   

第一步：获取PCAP，保存证书相关字节流。  

![图片](/assets/images/tool/extract_cert_bin.png) 

第二步：编写脚本将hex stream转为数组，并存在一个头文件当中，该python脚本如下:  


{% highlight c %}
  1 #!/usr/bin/python
  2 
  3 def usage():
  4 .       print 'Usage:'
  5 .       print '\thex2c.py filename'
  6 
  7 import sys
  8 import re
  9 
 10 if sys.argv[1:]:
 11 .       try:
 12 .       .       infile = open(sys.argv[1], 'r')
 13 .       except IOError, msg:
 14 .       .       print 'Can\'t open "%s":' % sys.argv[1], msg
 15 .       .       sys.exit(1)
 16 
 17 .       imagename = sys.argv[1]
 18 .       imagename = sys.argv[1].replace('.', '_')
 19 
 20 .       outfile = open(imagename + '.c', 'w')
 21 
 22 .       content = infile.read()
 23 .       size = len(content)
 24 
 25 .       outfile.write('/* content of file "%s" */\n' % sys.argv[1]);
 26 .       outfile.write('static unsigned char ' + imagename + '[%d] = {\n'%size)
 27 .       process = 0
 28 
 29 .       while 1:
 30 .       .       if size <= 0:
 31 .       .       .       break
 32 
 33 .       .       outstring = '\t'
 34 
 35 .       .       for i in range (0, 8):
 36 .       .       .       if ((size - i * 2) <= 1) :
 37 .       .       .       .       break
 38 .       .       .       else:
 39 .       .       .       .       outstring += '0x'
 40 .       .       .       .       outstring += content[process + 2 * i]
 41 .       .       .       .       outstring += content[process + 2 * i + 1]
 42 .       .       .       .       outstring += ', '
 43 
 44 .       .       outstring += '\n'
 45 .       .       outfile.write(outstring)
 46 .       .       process += 16
 47 .       .       size -= 16
 48 
 49 .       outfile.write('};\n')
 50 
 51 .       infile.close()
 52 .       outfile.close()
 53 
 54 .       print 'Complete.'
 55 else:
 56 .       usage()
 57 
 58 sys.exit(0)
{% endhighlight %}


第三步：修改源代码，在文件中包含该头文件，使得证书验证的数据流指向该数据即可。 
