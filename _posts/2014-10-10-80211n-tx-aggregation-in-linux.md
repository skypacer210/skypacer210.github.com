---
layout: post
title: "802.11n TX aggregation in Linux"
categories:
- technology  
tags:
- Linux
- TX Aggregation
- mac80211


---

##小引
----

##正文
----  

本文研究了Linux下802.11n TX aggregation过程。

###1、MAC80211 Framework  

Linux协议栈中mac80211的整体架构如下图所示：	  

![图片](/assets/images/mac80211_framework.png)  
<center>Figure 1: mac80211 framework</center>

###2、Transmission: kernel->mac80211->iwlwifi  

kernel->mac80211->iwlwifi的具体层次如下图所示：	  

![图片](/assets/images/tx_agg_1.png)  
<center>Figure 1: kernel->mac80211->iwlwifi flow</center>


###3、Transmission: iwlwifi->hardware  

iwlwifi->hardware的具体层次如下图所示：	  

![图片](/assets/images/tx_agg_1.png)  
<center>Figure 1: kernel->mac80211->iwlwifi flow</center>