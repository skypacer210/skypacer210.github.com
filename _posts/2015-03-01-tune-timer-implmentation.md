---
layout: post
title: "关于timer实现的优化"
categories:
- technology
tags:
- Timer
- Event Loop
- RedBlack Tree


---


## 1. Timer ##

Timer是底层库实现中重要的一部分，其作用主要是实现超时处理，应用的场景有：  

- EAP状态机实现
- 事件驱动型模型实现
- SSL握手超时处理

为了支持实时处理，将Timer的内部实现由原来的单向链表变为红黑树。  


### 2. 为什么是红黑树 ###    

红黑树是一种平衡树，但却不是一种基于高度的平衡树，在STL map和Linux中都作为平衡树来应用，当搜索的要求高或者修改频繁时，其表现效果更是好于AVL树，因为后者是基于高度的，所以在每次修改后都要进行rebalance，开销很大。而红黑树的插入只要两次旋转，删除最多三次旋转，虽然搜索稳定性不及AVL，但是仍不失为一种折中的好办法。


为保持红黑树的平衡，需要对树进行一些调整，好比为了保证一个黑客集团内部的黑帽子和红帽子的平衡，在成员之间的帽子问题上进行调整，假定只有两种帽子，黑帽子和红帽子，而且等级森严，按照辈分排：  

- 根节点必须为黑色，就是说老大必须是黑帽子；
- 每个成员非黑即红；
- 每个红帽子下面有两个黑帽子；
- 从任何一个成员自己数起到自小的下线，黑帽子个数都是一致的。


当一个新的成员（帽子为暂定为红色）加入后，对家族成员帽子颜色的调整有下面几种情况：  

- CASE1：新加入成员的Uncle辈是个红帽子，调整波及Uncle、Parent和Granparent，全部调整帽子颜色；
- CASE2：新加入成员的Uncle是个黑帽子，而且该成员是Uncle父亲的左孩子的右孩子或者是右孩子的左孩子，调整波及Uncle和Granparent，以Uncle为轴旋转，交换了Uncle和Granparent的位置。
- CASE3：新加入成员的Uncle是个黑帽子，而且该成员是Uncle父亲的左孩子的左孩子或者是右孩子的右孩子，调整波及很大，不仅涉及Uncle和Granparent的调整（以Uncle为轴旋转，交换了Uncle和Granparent的位置），还需将Uncle和Granparent的帽子颜色换掉。

上述规则可用下图表示：  

![图片](/assets/images/rb_rule.png)

依次插入6个节点123456，建立一颗红黑树如图所示：  

![图片](/assets/images/rb_insert.png)

## 3. 参考资料 ##

1. [RB Tree Howto](https://www.youtube.com/watch?v=PhY56LpCtP4)  
1. [RB Tree Github](http://kakack.github.io/2014/04/%E9%9D%A2%E8%AF%95%E7%BC%96%E7%A8%8B%EF%BC%9A%E7%BA%A2%E9%BB%91%E6%A0%91/)  

