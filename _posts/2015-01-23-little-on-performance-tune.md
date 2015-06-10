---
layout: post
title: "浅谈VDI优化"
categories:
- technology
tags:
- performance tune
- TLS
- memory pool


---


## 概述 ##

VDI优化涉及到诸多系统的方面，整个系统的大框架如图所示：  

![图片](/assets/images/perf/VDI_framework.png)

### VDI APP层 ###  

APP内部实现架构选择，比如是采用多进程模型还是单进程模型；  

### HTTP层 ###   

目前整体流程基于过程式交互方式，可以改为基于事情驱动的状态机实现方式；  

### SSL层 ###  

首先评估SSL的overhead到底有多大，以及主要瓶颈，针对性的进行优化；  
SSL双方采用的版本也影响overhead，目前SSL的版本号有SSL2.0 SSL3.0，TLS1.0(SSL3.1)，TLS1.0(SSL3.2)和TLS1.0(SSL3.3)。版本号的协商基于如下公式：  

`min(max. client supported version, max. server supported version)`  


### TCP/IP层 ###

**关于序列号**   
  
1. TCP中双方的握手和分手过程，所有报文的`TCP Segment Len`都为0。  
2. `Window`就是滑动窗口，用来解决流控问题，即如何提高网络吐吞率，主要涉及到拥塞控制的参数调整。    
3. `Sequence Number`用来解决乱序问题，即我发送了多少加多少（对于三次握手阶段，以报文个数为单位，即在对方`Acknowledgment Number`基础上递增1；对于数据发送阶段，以字节数为递增单位，即在自己现有的`Sequence Number`上递增`TCP Segment Len`）。
4. `Acknowledgment Number`用于确认收包，解决丢包问题，即我收到了多少加多少（对于三次握手阶段，同样以报文个数为递增单位，即在对方`Sequence Number`的基础上上递增1；对于数据发送阶段，也同样以字节数为递增单位，即在自己现有的`Acknowledgment Number`上递增`TCP Segment Len`）。如图所示：

![图片](/assets/images/perf/tcp_seq.png)

其中客户端192.168.69.2和服务器192.168.69.1的三次握手中，服务器的`seq`是自己产生的，`ack`则是在收到客户端的SYN报文后递加1（客户端的`seq + 1`），表示收到了客户端的连接请求；客户端的`seq`也是自个产生的，发送了一个SYN，就递加1，`ack`则是在收到服务器的SYN/ACK报文后在服务端的`seq + 1`。  
在TCP连接建立后，客户端向服务端发送一个`TCP Segment Len`为445字节的分片#4，当前`seq`为1（相对序列号），不管是否送达下一个分片#6发送时，其`seq`会忠实地递增445字节，变为446。服务端收到分片#4后，将`ack`递增445字节。

总之，`Sequence Number`和`Acknowledgment Number`像极了人的奋斗过程，前者好比付出，后者好比收获，但是付出不一定有收获（如果得到ACK确认则成功，否则失败）。  
  

**网络层：**    

主要涉及到网络接口驱动的实现优化，比如中断轮询模式选择，是否启用DMA等。

### 加入LRU Cache ###

LRU即Least Recently Used，是一种在cache中广泛应用的算法，保持经常访问的数据节点在cache中，如果某块节点很少被访问，在达到某种阈值之后就把它交换出去，这就要求查找和插入删除都要快。本系统采用双向链表和哈希表实现，主要利用了双向链表的插入和删除操作和哈希表的查找操作，数据结构定义如下:   

{% highlight c %}
  6 typedef struct __cacheNode {
  7 .       ubyte4. .       .       key;
  8 .       int.    .       .       value;
  9 .       struct list_head.       node;
 10 } cacheNode;
 11 
 12 typedef struct __LRUCache {
 13 .       struct list_head.       first;		//指向链表头部，代表最新访问节点
 14 .       struct list_head.       last;		//指向链表尾部，代表最旧访问节点
 15 .       hashTableOfPtrs..       *cacheMap;	//哈希表
 16 .       int.    .       .       capacity;	//cache容量
 17 .       int.    .       .       curr_size;	//当前节点个数
 18 } LRUCache;
{% endhighlight %}

对应的操作主要就是get和put。    

- 创建LRU cache。

{% highlight c %}

 59 MSTATUS LRU_create(LRUCache **ppNewLRUCache, int capacity)
 60 {
 61 .       MSTATUS..       status = OK;
 62 .       LRUCache.       *pNewLRU = NULL;
 63 .       hashTableOfPtrs.*pCacheMap;
 64         ubyte4. .       remain;
 65         ubyte4. .       count = 0;
 66 
 67 .       if (NULL == (pNewLRU = MALLOC(sizeof(LRUCache)))) {
 68 .       .       status = ERR_MEM_ALLOC_FAIL;
 69 .       .       goto exit;
 70 .       }
 71 
 72 .       MOC_MEMSET((ubyte *)(pNewLRU), 0x00, sizeof(LRUCache));
 73 
 74 .       /*Align capacity*/
 75 .       remain = capacity;
 76 .       while (remain > 0) {
 77 .       .       remain = remain >> 1;
 78 .       .       count++;
 79 .       }
 80 
 81 .       if (OK > (status = HASH_TABLE_createPtrsTable(&pCacheMap, (1 << count) - 1, NULL, LRU_allocHashPtrElement, LRU_freeHashPtrElement)))
 82 .       .       goto exit;
 83 
 84 .       pNewLRU->capacity = capacity;
 85 .       pNewLRU->curr_size = 0;
 86 .       pNewLRU->cacheMap = pCacheMap;
 87 .       pNewLRU->first.prev = pNewLRU->first.prev = &pNewLRU->first;
 88 .       pNewLRU->last.prev = pNewLRU->last.prev = &pNewLRU->last;
 89 .       *ppNewLRUCache = pNewLRU;
 90 exit:
 91 .       return status;
 92 }

{% endhighlight %}

- 销毁LRU cache。

{% highlight c %}
 94 MSTATUS LRU_release(LRUCache *pLRUCache)
 95 {
 96 .       MSTATUS status = OK;
 97 
 98 .       if (NULL == pLRUCache) {
 99 .       .       status = ERR_NULL_POINTER;
100 .       .       goto exit;
101 .       }
102 
103 .       HASH_TABLE_removePtrsTable(pLRUCache->cacheMap, NULL);
104 .       FREE(pLRUCache);
105 .       pLRUCache = NULL;
106 exit:
107 .       return status;
108 }
{% endhighlight %}

- 插入LRU node。

基本思路：如果待插入数据在cache中没有（cache miss）且没有超出cache容量,就为其分配节点，插入至链表头部，表示最近访问，同时插入哈希表当中；如果待插入数据已经存在且cache已经满了，那么首先找到最后一个元素，意味着最近没有访问，从哈希表中删除，然后从双链表中删除。

{% highlight c %}
209 extern MSTATUS
210 LRU_setNode(LRUCache *pCache, ubyte4 key, int value)
211 {
212 .       MSTATUS..       status = OK;
213 .       cacheNode.      *pNode = NULL, *pNewNode = NULL, *pLastNode = NULL;
214 .       ubyte4. .       hashValue = 0;
215 .       intBoolean.     foundHashValue;
216 .       int.    .       hashData;
217 
218 .       hashData = key;
219 .       HASH_VALUE_hashGen(&hashData, sizeof(hashData), INIT_HASH_VALUE, &hashValue);
220 
221 .       if (OK > (status = HASH_TABLE_findPtr(pCache->cacheMap, hashValue, (void *)&hashData, 
222 .       .       .       .       .       LRU_testKey, (void **)pNode, &foundHashValue)))
223 .       .       goto exit;
224 
225 .       if ((TRUE != foundHashValue) || (NULL == pNode)) {
226 .       .       if (pCache->curr_size == pCache->capacity) {
227 .       .       .       /*Remove last one*/
228 .       .       .       pLastNode = LIST_ENTRY(pCache->last.next, cacheNode, node);
229 .       .       .       hashData = (ubyte4)pLastNode->key;
230 .       .       .       HASH_VALUE_hashGen(&hashData, sizeof(hashData), INIT_HASH_VALUE, &hashValue);
231 
232 .       .       .       HASH_TABLE_deletePtr(pCache->cacheMap, hashValue, (void *)&hashData,
233 .       .       .       .       .       LRU_testKey, (void *)pNode, &foundHashValue);
234 .       .       .       LRU_removeLast(pCache);
235 .       .       .       if (pNode)
236 .       .       .       .       FREE(pNode);
237 .       .       } else {
238 .       .       .       pCache->curr_size++;
239 .       .       }
240 .       }
241 .              
242 .       pNewNode = MALLOC(sizeof(cacheNode));
243 .       if (NULL == pNewNode)
244 .       .       status = ERR_MEM_ALLOC_FAIL;
245 
246 .       pNewNode->key = key;
247 .       pNewNode->value = value;
248 .       pNewNode->node.prev = pNewNode->node.prev = &pNewNode->node;
249 
250 .       LRU_moveToHead(pCache, pNewNode);
251 
252 .       hashData = pNewNode->key;
253 .       HASH_VALUE_hashGen(&hashData, sizeof(hashData), INIT_HASH_VALUE, &hashValue);
254 .       if (OK > (status = HASH_TABLE_addPtr(pCache->cacheMap, hashValue, (void *)pNewNode)))
255 .       .       goto exit;
256 
257 exit:
258 .       return status;
259 }
{% endhighlight %}


- 查找LRU node。

如果cache miss也就罢了，如何找到了，在返回该节点之前因此需要将其移到链表的头部，因为查找某个节点意味着对该节点有了最新的访问。

{% highlight c %}

181 extern MSTATUS
182 LRU_getNode(LRUCache *pCache, ubyte4 key, int *pValue)
183 {
184 .       MSTATUS..       status = OK;
185 .       cacheNode.      *pNode = NULL;. /*AppData*/
186 .       ubyte4. .       hashValue = key;
187 .       intBoolean.     foundHashValue;
188 .       int.    .       hashData;
189 
190 .       hashData = key;
191 .       HASH_VALUE_hashGen(&hashData, sizeof(hashData), INIT_HASH_VALUE, &hashValue);
192 
193 .       if (OK > (status = HASH_TABLE_findPtr(pCache->cacheMap, hashValue, (void *)&hashData, 
194 .       .       .       .       .       LRU_testKey, (void **)&pNode, &foundHashValue)))
195 .       .       goto exit;
196 
197 .       if ((TRUE != foundHashValue) || (NULL == pNode)) {
198 .       .       status = -1;
199 .       .       goto exit;
200 .       }
201 
202 .       LRU_moveToHead(pCache, pNode);
203 
204 .       *pValue = pNode->value;
205 exit:
206 .       return OK;
207 }

{% endhighlight %}  

实际上两个辅助函数`LRU_moveToHead`和`LRU_removeLast`的实现值得注意：   

其中`LRU_moveToHead`的基本实现思路就是先促成当前节点前后两个节点的牵手，然后再往前坐链表头，最后修改`first`和`last`节点。  

{% highlight c %}
134 static 
135 void LRU_moveToHead(LRUCache *pCache, cacheNode *pNode)
136 {
137 .       struct list_head.       *first = &pCache->first;
138 .       struct list_head.       *last  = &pCache->last;
139 .       struct list_head.       *curr = &pNode->node;
140 
141 .       if (first == curr)
142 .       .       return;
143 
144 .       if (NULL != curr->prev)
145 .       .       curr->prev->next = curr->next;
146 
147 .       if (NULL != curr->next)
148 .       .       curr->next->prev = curr->prev;
149 
150 .       if (curr == last)
151 .       .       last = curr->prev;
152 
153 .       if (NULL != first) {
154 .       .       first->prev = curr;
155 .       .       curr->next = first;
156 .       }
157 
158 .       first = curr;
159 .       curr->prev = NULL;
160 
161 .       if (NULL == last)
162 .       .       last = first;
163 }
{% endhighlight %}

### TLS层的优化点 ###

TLS连接的建立和关闭过程如图：  

![图片](/assets/images/perf/SSL_handshake.png)

在TLS中频繁分配释放的内存主要有以下几类：   
- 哈希描述符和哈希值;  
- 密钥，比如masterkey为48字节;  
- 典型的小块内存，一般为64字节和96字节;    

Memory Pool的主要设计思路：  
- 每块内存都看做一个对象，预先分配好一定数量的对象，即所谓的内存池；  
- 定义get和put方法，一般成对出现，这样保证了预分配的对象数量可控；    
- 内存池内的每个对象用单向链表维护，具体操作见图；  

![图片](/assets/images/perf/mem_pool.jpg)


对SSL的握手过程进行对比测试，结果表明在使用mem pool的性能提升为8%左右，似乎并不理想，看来在SSL握手中此类内存的分配释放并不是影响性能的主要因素。 


### 充分利用硬件加速 ###  

对于计算密集型的加解密运算，首先想到的就是利用硬件进行加速，这里有两个方面的考虑：  

- 主存和硬件加速单元之间的IO： 数据从主存搬移至硬件，硬件将结果返回给主存  
- 对硬件处理结果的查询方式选择： 轮询还是中断？    

针对上述考虑，采取的措施如下：    

- 分析分析硬件加速的实现代码，发现其涉及到非常多固定大小内存的分配与释放，依然采用上文中的cache机制，需要注意的是硬件读写时要求地址对齐，因此对cache实现稍作修改，加入内存对齐处理。测试结果显示，后者能将性能提升50%左右(见下图)。   

![图片](/assets/images/perf/SSL_mcache_test.png)
  
- 利用DMA进行数据拷贝避免动用CPU在主存和硬件加速单元之前拷贝，测试结果表明两者的性能差异大约在2倍左右。
- 分别实现中断模式和轮询模式，对比后发现轮询模式下效率更高，因此选择了轮询模式。  

整体而言，采用硬件加速就是在充分利用压榨硬件计算性能的基础上尽可能减少内存拷贝和IO消耗，最后性能平均提升大约十倍左右。  

