---
layout: post
title: "802.11n TX aggregation in Linux"
categories:
- 80211
tags:
- Linux
- TX Aggregation


---

##小引
----

##正文
----  

本文研究了Linux下802.11n TX aggregation过程。

###1、Transmission: kernel->mac80211->iwlwifi  

kernel->mac80211->iwlwifi的具体层次如下图所示：	  

![图片](/assets/images/tx_agg_1.png)  
<center>Figure 1: kernel->mac80211->iwlwifi flow</center>


###2、Transmission: iwlwifi->hardware  

iwlwifi->hardware的具体层次如下图所示：	  

![图片](/assets/images/tx_agg_1.png)  
<center>Figure 1: kernel->mac80211->iwlwifi flow</center>