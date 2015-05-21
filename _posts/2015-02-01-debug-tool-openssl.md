---
layout: post
title: "调试工具杂记之OpenSSL篇"
categories:
- technology
tags:
- OpenSSL
- debug


---


## 1. 概述 ##

OpenSSL，可以用来公私密钥对产生，证书查看（查看证书整体或者是单个信息），证书格式转化以及网络抓包，很多时候可以用于验证。

### 2. 利用OpenSSL进行验证 ###    


**利用OpenSSL验证证书**  

由于OpenSSL在验证证书时基于PEM格式，因此首先将非PEM格式转化为PEM格式。root证书由对方发送过来，但是server没有给，到是可以从网络抓包中export出来如图所示：  

![图片](/assets/images/tool/extract_cert.png)  

- 转化root证书   
  `openssl x509 -inform der -in FZROOTCA.cer -out FZROOTCA.pem`
- 查看证书  
  `openssl x509 -in rsa_server.pem -noout –text`  
- 转化sever证书    
  `openssl x509 -inform der -in rsa_server.der -out rsa_server.pem`  
- 利用root证书验证server证书：  
  `openssl verify -verbose -CAfile FZROOTCA.pem  rsa_server.pem`     
结果显示：  
  `rsa_server.pem: OK`，但貌似并不对签名进行验证，继续。

**利用OpenSSL验证签名**   

为了验证签名，需要将public key从证书中单独抓出来：  

  `openssl x509 -pubkey -noout -in rsa_server.pem > server_pubkey.pem`

同时也需要将Signature从证书中抓出来：  

  `openssl asn1parse -in rsa_server.pem -out sig -noout -strparse 614`  

利用抓出来的signature和public key进行该证书的签名验证：  

  `openssl rsautl -in sig -verify -asn1parse -inkey server_pubkey.pem –pubin`  

结果显示：  

  `RSA operation error`
  `3177:error:0407006A:rsa routines:RSA_padding_check_PKCS1_type_1:block type is not 01:rsa_pk1.c:100:`
  `3177:error:04067072:rsa routines:RSA_EAY_PUBLIC_DECRYPT:padding check failed:rsa_eay.c:699:`  
  
对应代码如下：    

{% highlight c %}
  1 int RSA_padding_check_PKCS1_type_1(unsigned char *to, int tlen,
  2 .       .       const unsigned char *from, int flen, int num)
  3 {
  4 .       int i,j;
  5 .       const unsigned char *p;
  6 
  7 .       p=from;
  8 .       if ((num != (flen+1)) || (*(p++) != 01))
  9 .       { 10 .       .       RSAerr(RSA_F_RSA_PADDING_CHECK_PKCS1_TYPE_1,RSA_R_BLOCK_TYPE_IS_NOT_01);
 11 .       .       return(-1);
 12 .       }
 13 
 14 .       /* scan over padding data */
 15 .       j=flen-1; /* one for type. */
 16 .       for (i=0; i<j; i++)
 17 .       {
{% endhighlight %}  


从上面的结果可以得知公钥未能正确对签名解密，严重怀疑server将私钥弄错了。  

### 开发新功能 ###   

为了在原有安全库的基础上支持PSSRSA，需要理解此类算法的封装格式，利用OpenSSL直接可以查看此类证书的ASN1编码格式：   

  `openssl asn1parse -in pss_rsa_server.pem`    

当然OpenSSL的功能远不止这些，后续会陆续添加。


### 参考资料 ###

1. [How to Extract an SSL Certificate from a Network Packet Trace File in Wireshark](http://support.citrix.com/article/CTX132907)  
1. [openssl rsautl](https://www.openssl.org/docs/apps/rsautl.html)  
1. [OpenSSL Command-Line HOWTO 1](https://www.madboa.com/geek/openssl/#digest-verify)  
1. [OpenSSL Command-Line HOWTO 2](https://www.madboa.com/geek/openssl/)  

