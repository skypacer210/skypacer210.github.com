---
layout: post
title: "基于ARM平台的AES硬件加速实现"
categories:
- technology  
tags:
- ARM
- AES
- Acceleration


---

## 1、硬件加密引擎特点 ##

硬件加速引擎的目的在于减少加解密中软件的干预，从而提高性能.该硬件加密引擎共有四个引擎，每次运行一个，支持的加密算法有：  

- DES(Only ECB and CBC modes)   
- 3DES(ECB/CBC EDE and EEE modes)    
- AES(128/192/256):  其中AES-CBC支持IV feedback，即加密完之后密文的最后16个字节拷贝至引擎的内部SRAM中的IV位置  

对AES而言，首先将报文、配置信息以及密钥和初始化向量放入一连串的TDMA描述符当中，并配置chain模式，最后启动TDMA，即可连续加密。  IV是由用户配置的，后续的IV由硬件自己填充，CTR模式可以通过软件实现。    
中断发生，会将对于位置位，通知软件取数。     

支持的验证算法为SHA或者MD5。

在硬件加速实现中，CPU和加速引擎精密协作但具体分工又不尽相同。  
  
对CPU而言，主要完成三件事：

- 直接拷贝报文至加速器的本地SRAM中（没有采用DMA的情况下）；  
- 准备一个描述符，声明所需要的操作，目前描述符在主存中分配和配置`struct sa_accel_sram`，并映射至硬件当中对应的镜像位置（偏移为`SRAM_CONFIG`, `SRAM_KEY`, `SRAM_DATA_IN`等）。  
- 通过Sheeva CPU core，激活加速器。    

而加速引擎的职责包括计算、与CPU交互和数据搬运，具体来说，包括CESA和TDMA两个子引擎，其中CESA负责对数据进行操作，主要流程如下：    

1. 读取描述符  
2. 启动引擎 
3. 开始处理报文块，一次一个
4. 等待完成
5. 从引擎读取数据
6. 存储数据值本地SRAM    

而TDMA只是负责在主存和SRAM之间拷贝数据。

## 2、CESA配置流程 ##

### 2.1、引入加密引擎 ###
 
**Access**

CPU和SEAS交替控制，但在一个时间点上只能有一个作为HOST，TDMA当中有owner位，指示出当前的HOST是TDMA设备还是CPU。  
 
**Control**

若干特殊的命令控制器，比如SHA-1/MD5相关的命令寄存器：    


{% highlight objc %}
#define SHA1_MD5_CTRL_REG 0x0003DD18 /	\*Set when hash complete; Clear when any write to the auth engines\*/   
#define SHA1_MD5_TERM_MASK 0x1 << 31) /	\*bit31\*/
{% endhighlight %}

**输入**  

1. 输入数据长度为64bit或者128bit 
2. 输入带验证数据长度为512bit  
3. 写数据寄存器，加载输入数据(只有DES支持多任务)
4. 写密钥寄存器和IV寄存器


**输出**

1. 加密引擎返回数据长度也为64bit或者128bit
2. 验证引擎返回相应的签名长度.
3. 通过指定的数据寄存器访问输出数据  

**操作原则**

1. 写入加密参数
2. 写入数据，激活引擎
3. 在指定的命令寄存器上设置终止位
4. 使能中断通知CPU操作完毕
5. CPU从寄存器读取结果  

### 2.2、88AP510安全加密引擎 ### 

**地址映射**   

{% highlight objc %}
\* phys virt size       
\* c8000000 fdb00000 1M Cryptographic SRAM     
\* e0000000 @runtime 128M PCIe-0 Memory space 
\* e8000000 @runtime 128M PCIe-1 Memory space 
\* f1000000 fde00000 8M on-chip south-bridge registers 
\* f1800000 fe600000 8M on-chip north-bridge registers 
\* f2000000 fee00000 1M PCIe-0 I/O space 
\* f2100000 fef00000 1M PCIe-1 I/O space

#define DOVE_CRYPT_PHYS_BASE. . (DOVE_SB_REGS_PHYS_BASE | 0x30000) /	\* Cryptographic Engine \*/
{% endhighlight %}



**寄存器配置**

实例： 修改DOVE寄存器   

{% highlight objc %}
575 . /* enable isolators */ 
576 . reg = MV_REG_READ(PMU_ISO_CTRL_REG); 
577 . reg &= ~PMU_ISO_VIDEO_MASK; 
578 . MV_REG_WRITE(PMU_ISO_CTRL_REG, reg);
{% endhighlight %}  
    
**加密流程**

- 检查 `aes_term` 是否为1，类似轮询操作；
- 配置 `AES_CTRL_REG`: 密钥长度和数据大小字节序；
- 写入密钥寄存器，总共8个，如果key不变，则无需配置；     
- 每次写入128bit的数据块: 配置AES ENC Data In/out register； 当这些寄存器填写完毕（填写顺序无所谓），加密开始；如果小于128，自动补0。   
- 轮询AES命令寄存器或者等待中断：在轮询模式下如果为1，说明结果已经准备好； 在中断模式下每次中断发生，结束位从0变为1，当中断发生后，主机向中断使能寄存器的Zint2写入0以重启它，写入1则无影响。  
- 获取AES结果：结果同样位于AES IO寄存器当中；引擎不会对字节序做任何修改。


**解密流程**

当加密操作完成时，key发生了变化，必须根据最后的加密key生成key schedule；具体如下：  

- 当用一个指定的key解密数据时候，主机首先加载key至解密引擎，然后设置AES解密命令寄存器为1，产生key schedule。
- 当从解密引擎中读取一个key的时候，主机必须等待AES命令寄存器的key schedule位为1，方可读取key寄存器。  

其中解密密钥生成步骤：  

1. 写入AES解密key n 寄存器；
2. 设置解密控制寄存器的aesDecMakekey位为1；
3. 轮询直到该位设置为1；
4. 从对应的解密key寄存器中读取key；
5. 清空AES解密控制寄存器。

主机可能会在内存中保存解密key，这样key计算可以在下次忽略（采用相同的key）:  
  
1.  轮询解密寄存器的终止位（类似加密过程）；  
2.  为解密寄存器设置解密模式和大小端；  
3.  写入生成的key；  
4.  写入解密数据；   
5.  轮询解密寄存器或者是等待中断发生；  
6.  读取AES结果；  
7.  读取key(主机必须加载一个raw key)。

## 3、实现 ##

### 3.1、设计原理 ###

启用CESA，本质上是数据在DRAM和SRAM之间的拷贝，如果采用DMA，需要将DRAM和SRAM中数据的虚拟地址转化为对应的物理地址。   
由于CESA只能处理在CESA SRAM中的报文，最大支持2KB，因此在包含其他信息的前提下（SA描述符、key、IV），实际处理的数据最大是1.75KB。  

- 对于每个报文，至少需要1个SA描述符和4个TDMA描述符，如果数据在物理内存中有分片，则会需要更多的TDMA描述符。
- 驱动利用预定义的SA和TDMA描述符池以期获得最好的性能，池的大小可以匹配请求的size大小。  


### 3.2、基本架构 ###  

**1、应用请求分发方式**

对于来自上层的应用请求（AES加解密）可以采用以下两种同步回调模式和异步回调模式。  

对于同步回调模式而言，同步操作较为简单，但是当请求被提交时程序会阻塞；调用同步回调函数（实现数据由软件传至硬件）即可，一旦硬件处理完毕，即可返回，流程如下：

1. 准备硬件请求；
2. 发送至硬件，等待完成；
3. 如果请求成功完成，用处理后的数据代替原先的数据；
4. 如果该数据需要验证，需要在MAC缓冲区存储一份MAC值。

而在异步回调模式下，异步请求会被加入硬件队列，同时返回，当硬件完成请求时，硬件调用另外一个处理程序返回，并送入上层协议栈，通常情况下效率更高，除了同步回调函数，必须注册另外一个回调函数（通知回调函数）以方便硬件完成操作之后调用，该回调通知函数在上层实现，主要将结果传至上层协议栈。  

1. 准备硬件请求，包括所有数据和参数，实现通知回调函数；
2. 根据返回值操作，如果正确则用处理后的数据替换原始数据；获取MAC值；清空本地内存。
3. 如果失败则释放内存；
4. 如果有必要，准备一个包含原始数据的指针；
5. 发生请求值硬件；
6. 返回状态代码；
7. 当硬件完成操作后，调用通知回调函数

目前采用的是同步方式。

**2、中间层**

在硬件接口和上层接口层之间加入一个中间层，以此隔离上层对于底层调用的依赖，可以基于纯软件实现AES加解密，也可以基于硬件实现。  

**3、流程控制**  

采用中断方式接收数据并进行处理。中断是硬件处理数据之后的动作，因此，触发中断需要有两级：  

- 使能中断，此时可以写对应中断掩码  
- 写入数据，此时开始硬件处理  

具体的流程如下：   
 
- 建立一个进程专门处理中断，通过读取中断寄存器判断是否有中断，如果中断上来就处理收发。  
- 设置中断寄存器；  
- 本地中断寄存器和掩码寄存器，就包括了CESA单元的，可以在pic.c中为CESA定义中断向量。     

**4、DMA**

在该平台的硬件引擎中，专门实现了DMA，称之为TDMA。TDMA的基本特点如下：

- 独立的TDMA引擎，无需CPU参与可以在DDR内存和内部SRAM之间搬移大规模数据, 通过TDMA从主存中拷贝报文至安全加速引擎的本地SRAM，最大支持2KB。
- 一次最大支持2KB，可以运行于chain mode，该模式下为每个缓冲区分配一个唯一的描述符。
- 一旦TDMA激活，禁止任何软件读访问本地SRAM；
- 保存与安全加速引擎相关的加密参数值，保存在主存当中，但是必须映射至引擎的SRAM当中；  
- 在安全加速器的SRAM中准备描述符，开始加速引擎需要的操作和参数。

TDMA最主要的寄存器是TMDA描述符，由四个32位寄存器组成：  

- Byte Count(0x800): 传输的数据字节数，最大支持64KB-1,随着数据从源地址向目的地址的拷贝递减，为0时，意味着TDMA传输完成或者终止;Own位表示CPU或者TMDA拥有该描述符;  
- Source Address(0x810)：TDMA源地址 必须为内部SRAM地址 可以直接将内部SRAM的固定偏移地址写入;
- Destination Address(0x820)：TDMA目的地址 必须为内部SRAM地址 提供了一种思路，遇到需要配置硬件地址的时候，可以定义一个固定偏移，这样软硬件填写方便;
- 下一个描述符指针(0x830)：为了支持chain模式，必须是16字节对齐（bit[3:0]=0）。

**5、TDMA与加密引擎协调工作**

TDMA作为一种硬件内置的DMA方式，与其他硬件加速引擎配合工作，较为显著的提高了吞吐率，基本流程如下：  

首先设置CESA配置寄存器（0xE08）的waitforTDMA和ActiveTDMA，当TMDA和CESA协同工作的时候，CESA能独立操作DRAM，CESA有能力激活TDMA，决定其状态并提供单独的完成中断；其中软件则需要步骤（五步）：  

- 在DRAM中配置TDMA描述符chain；
- 初始化TDMA配置寄存器；
- 在DRAM中或者SRAM中初始化CESA描述符和参数；
- 初始化CESA配置寄存器；
- 软件激活硬件加速。

TDMA的基本功能和软件流程综述如下：  

- 利用TDMA从主存DRAM中拷贝报文至CESA的本地SRAM中，对于大的报文，可以进行分片。
- 存储CESA所需的加密参数；
- 为CESA的本地SRAM准备一个描述符（MV\_CESA\_DESC），设定需要的操作和参数，描述符总共8个双字长；
- 通过配置CESA描述符指针寄存器（0xE04），将SRAM中MV\_CESA\_DESC数据域地址写入选定的session中（通过计算偏移得到）；
- 通过配置命令寄存器（0xE00）合适的位激活该session；
- 等待硬件完成，在轮询模式或者中断模式下的判断标志为寄存器（0xE0C）第0位，若为1则说明已经actived；
- 利用TDMA将完成的报文拷贝回主存。

其中CESA中的SRAM内存映射如下：  

{% highlight objc %}
/\*   
 \* CESA SRAM Memory Map: \*  
 \* +------------------------+ <= sc->sc_sram_base + CESA_SRAM_SIZE   
 \* |                        |   
 \* |          DATA          |   
 \* |                        |   
 \* +------------------------+ <= sc->sc_sram_base + CESA_DATA(0)    
 \* | struct cesa_sa_data    |   
 \* +------------------------+   
 \* | struct cesa_sa_hdesc |  
 \* +------------------------+ <= sc->sc_sram_base   
 */
{% endhighlight %} 
  

其中`CESA_SRAM_SIZE=2048`， `CESA_DATA(0)=sizeof（struct cesa_sa_hdesc）`，字节序则利用宏直接转化。  

对于分片报文，可以暂时不支持，以简化设计 CESA 的SDK支持最大报文为64 \* 1024，而一次处理1600大小，因此需要分片。

SRAM地址可以直接返回一个全局变量：`MV_U32 cesaCryptEngBase = 0`;

对于SA配置，可以定义数据结构`sa_config`和相应的配置函数。


### 4、调试小记 ###  


### 4.1、调试基本原则 ###   
   
在开发前期，为了尽快试错以及避免代码注释混乱，采用switch语句定义不同的dec\_key方法，并用全局变量用GUI配置，则无需重新编译即可完成正确方案的选择。而在面对时而正确时而错误的加解密情况，加入打印语句：  
  
- 首先排除硬件返回超时的错误；
- 排除CESA SRAM没有初始化的错误；
- 排除状态机的锁处理问题；
- 定位至状态机的处理，很可能在处理开始之前没有正确赋给初值。

基于以上分析思路，解决了数据乱码问题，定位过程如下：  
    
**1、数据拷贝流程分析**  
   
分解TDMA和CESA的交互过程，逐步调试，至比如数据从主存至TDMA，是否拷贝正确； 数据处理完之后是否拷贝至主存?
  
- 从cache到主存；
- DMA读： 从主存至设备；
- DMA写： 从设备至主存；
- 主存至cache。  

**2、Cache相关**   

这里涉及两个问题：clean cache和Invalide cache:  

- clean cache意味着将cache中的数据强制写入主存, 即设备访问到了最新的内容，保证了设备访问CPU的数据正确性。    
- invalide cache用来确保设备完成时，CPU可以从主存中读取到最新的内容，即保证了CPU访问设备的数据正确性。
   
**3、DMA相关**  
   
DMA要考虑页表连续性和CPU-cache。  

现象：  

- 有错包出现，且大小无规律；
- 现象二：memory trap，无规律。  

采用软件对比方法，对比几个方面，包括DMA-out/HW caculate,分析出错误点。硬件计算错误的原因（配置错误、密钥IV错误、源目的地址错误），排除了前二者，只能是传入了错误的物理地址给硬件。在进行虚实地址转化的时候，由于转化后的物理地址可能跨页，因此必须分页拷贝，即计算出距离页边界的偏移，判断是否跨页，如果跨了先拷贝当前页数据，在拷贝下一页数据。其中DMA和CPU-cache是两个不同的两个概念：  

- DMA指的是一种硬件设备，能够无须使用CPU指令实现对内存的拷贝；后者是主存和CPU之间的关系；
- DMA在进行数据拷贝的时候，或许用到CPU-cache的参与，或许根本就不用。

**4、随机问题定位**
    
在实际的调试中在单个进程中调用时没有问题，在多个进程调用的时候出现问题，因此可以确定可能是进程并发的问题。经过调试发现，中断会在休眠之前发生，因此在需要禁止中断。    

另外一个随机问题是在释放版本中发现的，硬件加速引擎偶尔出现超时，但是在开发版本中从不会超时，控制结构体中的一个描述状态的字段不太正常，后者相对前者做了编译优化，导致发生不可预期的错误，因此需要在其定义出加入volitile关键字，问题得以解决。  

## 5、参考文献 ##

1.  [Gallery of Processor Cache Effects](http://igoro.com/archive/gallery-of-processor-cache-effects/ "http://igoro.com/archive/gallery-of-processor-cache-effects/")
2.  [Memory alignment on modern processors?](http://stackoverflow.com/questions/1855896/memory-alignment-on-modern-processors "http://stackoverflow.com/questions/1855896/memory-alignment-on-modern-processors")
3.  [Will the cache line aligned memory allocation pay off?](http://stackoverflow.com/questions/4605030/will-the-cache-line-aligned-memory-allocation-pay-off "http://stackoverflow.com/questions/4605030/will-the-cache-line-aligned-memory-allocation-pay-off")
4.  [How correctly wake up process inside interrupt handlers](http://stackoverflow.com/questions/9809140/how-correctly-wake-up-process-inside-interrupt-handlers "http://stackoverflow.com/questions/9809140/how-correctly-wake-up-process-inside-interrupt-handlers")
5.  [race condition between wait\_event and wake\_up](http://stackoverflow.com/questions/11012406/race-condition-between-wait-event-and-wake-up?rq=1 "http://stackoverflow.com/questions/11012406/race-condition-between-wait-event-and-wake-up?rq=1")
6.  [Linux kernel interrupt handler mutex protection?](http://stackoverflow.com/questions/6570419/linux-kernel-interrupt-handler-mutex-protection?rq=1 "http://stackoverflow.com/questions/6570419/linux-kernel-interrupt-handler-mutex-protection?rq=1")
7.  [DSP DMA](http://www.noise.physx.u-szeged.hu/DigitalMeasurements/DSP/Sharc/Sharc/chap06.pdf "http://www.noise.physx.u-szeged.hu/DigitalMeasurements/DSP/Sharc/Sharc/chap06.pdf")