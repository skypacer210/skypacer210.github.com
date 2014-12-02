---
layout: post
title: "AES Acceleration in ARM"
categories:
- technology  
tags:
- ARM
- AES
- Acceleration


---

##目录
----

##正文
----
###1、特性分析

1.  Hardware-based cryptographic engines
2.  security accelerator(CESA)


####1.1、Cryptographic Engine Features
1.  Four engines, one at a time
2.  Encryption
    -   DES(Only ECB and CBC modes)
    -   3DES(ECB/CBC EDE and EEE modes)
    -   AES(128/192/256):  其中AES-CBC支持IV feedback，即加密完之后密文的最后16个字节拷贝至引擎的内部SRAM中的IV位置
3.  Authentication
    -   SHA or MD5
    -   continue mode, enables chaining between blocks   
对AES而言，首先将报文、配置信息以及密钥和初始化向量放入一连串的TDMA描述符当中，并配置chain模式，最后启动TDMA，即可连续加密。
    -   optimal external update of authentication and Encryption initial value    
 对于AES而言，IV是由用户配置的，后续的IV由硬件自己填充
    -   Support byte swap   
暂时没有设置改位段
    -   automatic activation
    -   authentication and encryption termination interrupts   
中断发生，会将对于位置位，通知软件取数
    -   support DES, OFB, and CFB modes with additional software  
比如CTR模式可以通过软件实现
    -   TDMA和CESA的配合各种方式见文档

####1.2、Security Accelerator Features
安全加速器的目的在于减少加解密中软件的干预，从而提高性能。  
####1.2.1、CPU职责    
1.一种方式是拷贝报文至加速器的本地SRAM中；另一种方式是利用TDMA引擎完成报文拷贝。  
2.准备一个描述符，声明所需要的操作，目前描述符在主存中分配和配置struct sa\_accel\_sram，并映射至硬件当中对应的镜像位置（偏移为SRAM\_CONFIG, SRAM\_KEY, SRAM\_DATA\_IN等）。  
3. 通过Sheeva CPU core，激活加速器。    
####1.2.2、加速器职责  
TDMA只是负责在主存和SRAM之间拷贝数据，CESA负责对数据进行操作，顺序如下：    

*  读取描述符
* 启动引擎
* 开始处理报文块，一次一个
* 等待完成
* 从引擎读取数据
* 存储数据值本地SRAM


###2、  CESA配置流程

####2.1、Using Cryptographic Engine

#####2.1.1、 Access

CPU和SEAS交替控制，但在一个时间点上只能有一个作为HOST，TDMA当中有owner位，指示出当前的HOST是TDMA设备还是CPU

#####2.1.2、 Control

若干特殊的命令控制器:   
     
* SHA-1/MD5 Auth cmd reg  

```  
\#define SHA1\_MD5\_CTRL\_REG 0x0003DD18 /\*Set when hash complete; Clear when any write to the auth engines\*/   
\#define SHA1\_MD5\_TERM\_MASK 0x1 << 31) /\*bit31\*/
```
*  DES cmd reg  
*  AES cmd reg

#####2.1.3、Input

1.Encryption engine: 64bit or 128bit   
2.Authentication: 512bit.  
3.Writing data reg: load input data(except DES, not support muti-tasking)

* key reg
* iv digest reg

#####2.1.4、output

1.  encryption engine return 64/128bits
2.  authentication engine return 4/5 words which are the hash signature.
3.  access output data: specific data in/out registers load input data

#####2.1.5、操作原则

1.  write cryptographic parameters
2.  write data(active the process)
3.  engine set a termination bit in one of the command reg
4.  active interrupt to notify the host that the operation is over.
5.  host read the result from reg.

####2.2、DOVE 88AP510 Security Acceleration Engine 

#### address maps

```
\* phys virt size       
\* c8000000 fdb00000 1M Cryptographic SRAM     
\* e0000000 @runtime 128M PCIe-0 Memory space 
\* e8000000 @runtime 128M PCIe-1 Memory space 
\* f1000000 fde00000 8M on-chip south-bridge registers 
\* f1800000 fe600000 8M on-chip north-bridge registers 
\* f2000000 fee00000 1M PCIe-0 I/O space 
\* f2100000 fef00000 1M PCIe-1 I/O space
```

```
\#define DOVE\_CRYPT\_PHYS\_BASE. . (DOVE\_SB\_REGS\_PHYS\_BASE | 0x30000) /\* Cryptographic Engine \*/
```

####2.2.1、AES Acceleration

####1、寄存器配置

实例： 修改DOVE寄存器   

```
575 . /* enable isolators */ 
576 . reg = MV_REG_READ(PMU_ISO_CTRL_REG); 
577 . reg &= ~PMU_ISO_VIDEO_MASK; 
578 . MV_REG_WRITE(PMU_ISO_CTRL_REG, reg);
```  

####3、加密流程  

1.  verify aes\_term = 1; 类似轮询操作
2.  configure AES\_CTRL\_REG: key size and date swap大小字节序的操作？
3.  write key registers: 总共8个，如果key不变，则无需配置     
4.  write block for 128-bit data: 配置AES ENC Data In/out register；当这些寄存器填写完毕（填写顺序无所谓），加密开始；如果小于128，自动补0。   
5.  轮询AES命令寄存器或者等待中断
    *  轮询模式: 如果为1， 说明结果已经准备好；  
    *  中断模式: 每次中断发生，结束位从0变为1，当中断发生后，主机向中断使能寄存器的Zint2写入0以重启它，写入1则无影响。  
6.  获取AES结果    
结果同样位于AES In/Out寄存器当中；引擎不会对字节序做任何修改；


####4、解密流程

当加密操作完成时，key发生了变化，必须根据最后的加密key生成key schedule；具体如下：  

*  当用一个指定的key解密数据时候，主机首先加载key至解密引擎，然后设置AES解密命令寄存器为1，产生key schedule。  
*  当从解密引擎中读取一个key的时候，主机必须等待AES命令寄存器的key schedule位为1，方可读取key寄存器

其中解密密钥生成步骤：

    1.  写入AES解密key n 寄存器
    2.  设置解密控制寄存器的aesDecMakekey位为1,
    3.  轮询直到该位设置为1
    4.  从对应的解密key寄存器中读取key
    5.  清空AES解密控制寄存器

主机可能会在内存中保存解密key，这样key计算可以在下次忽略（采用相同的key）:  
1.  轮询解密寄存器的终止位（类似加密过程）  
2.  为解密寄存器设置解密模式和大小端  
3.  写入生成的key  
4.  写入解密数据   
5.  轮询解密寄存器或者是等待中断发生  
6.  读取AES结果  
7.  读取key(主机必须加载一个raw key)

###3、实现

####3.1、设计原理

启用CESA，本质上是数据在DRAM和SRAM之间的拷贝，如果采用DMA，需要将DRAM和SRAM中数据的虚拟地址转化为对应的物理地址。   
由于CESA只能处理在CESA SRAM中的报文，最大支持2KB，因此在包含其他信息的前提下（SA描述符、key、IV），实际处理的数据最大是1.75KB。

1.  对于每个报文，至少需要1个SA描述符和4个TDMA描述符，如果数据在物理内存中有分片，则会需要更多的TDMA描述符
2.  驱动利用预定义的SA和TDMA描述符池以期获得最好的性能，池的大小可以匹配请求的size大小

####3.2、基本架构
#####3.2.1、应用请求分发方式

对于来自上层的应用请求（AES加解密）可以采用以下两种分发方式：

1.  同步回调模式：同步操作较为简单，但是当请求被提交时程序会阻塞；调用同步回调函数（实现数据由软件传至硬件）即可，一旦硬件处理完毕，即可返回，流程如下：
    1.  准备硬件请求；
    2.  发送至硬件，等待完成；
    3.  如果请求成功完成，用处理后的数据代替原先的数据
    4.  如果该数据需要验证，需要在MAC缓冲区存储一份MAC值
2.  异步回调模式：异步请求会被加入硬件队列，同时返回，当硬件完成请求时，硬件调用另外一个处理程序返回，并送入上层协议栈，通常情况下效率更高，除了同步回调函数，必须注册另外一个回调函数（通知回调函数）以方便硬件完成操作之后调用，该回调通知函数在上层实现，主要将结果传至上层协议栈。
	1.  准备硬件请求：包括所有数据和参数，实现通知回调函数
	2.  根据返回值操作，如果正确则用处理后的数据替换原始数据；获取MAC值；清空本地内存
	3.  如果失败则释放内存
	4.  如果有必要，准备一个包含原始数据的指针
	5.  发生请求值硬件
	6.  返回状态代码
	7.  当硬件完成操作后，调用通知回调函数

目前采用的是同步方式。

#####3.2.2、中间层
在硬件接口和上层接口层之间加入一个中间层，以此隔离上层对于底层调用的依赖，可以基于纯软件实现AES加解密，也可以基于硬件实现。  

#####3.2.3、流程控制
采用中断方式接收数据并进行处理。中断是硬件处理数据之后的动作，因此，触发中断需要有两级：  

* 使能中断----可以写对应中断掩码  
* 写入数据----开始硬件处理

具体的流程如下：  
1. 建立一个进程专门处理中断，通过读取中断寄存器判断是否有中断，如果中断上来就处理收发。  
2. 设置中断寄存器  
3. 本地中断寄存器和掩码寄存器，就包括了CESA单元的，可以在pic.c中为CESA定义中断向量  

####3.2.4、TDMA控制

#####3.2.4.1、 TDMA特性

1.  独立的TDMA引擎，无需CPU参与可以在DDR内存和内部SRAM之间搬移大规模数据, 通过TDMA从主存中拷贝报文至安全加速引擎的本地SRAM，最大支持2KB  
2.  一次最大支持2KB，可以运行于chain mode，该模式下为每个缓冲区分配一个唯一的描述符
3.  一旦TDMA激活，禁止任何软件读访问本地SRAM
4.  保存与安全加速引擎相关的加密参数值，保存在主存当中，但是必须映射至引擎的SRAM当中  
5.  在安全加速器的SRAM中准备描述符，开始加速引擎需要的操作和参数，


#####3.2.4.2、TDMA寄存器

这里指简要介绍TMDA描述符，主要是四个32位寄存器
    1.  Byte Count(0x800): 传输的数据字节数，最大支持64KB-1,随着数据从源地址向目的地址的拷贝递减，为0时，意味着TDMA传输完成或者终止;Own位表示CPU或者TMDA拥有该描述符:
    2.  Source Address(0x810)：TDMA源地址 必须为内部SRAM地址 可以直接将内部SRAM的固定偏移地址写入
    3.  Destination Address(0x820)：TDMA目的地址 必须为内部SRAM地址 提供了一种思路，遇到需要配置硬件地址的时候，可以定义一个固定偏移，这样软硬件填写方便
    4.  下一个描述符指针(0x830)：为了支持chain模式，必须是16字节对齐（bit[3:0]=0）

#####3.2.4.3、TDMA与CESA协同工作

1.  设置CESA配置寄存器（0xE08）的waitforTDMA和ActiveTDMA
2.  当TMDA和CESA协同工作的时候，CESA能独立操作DRAM，CESA有能力激活TDMA，决定其状态并提供单独的完成中断
3.  软件步骤（五步）：
    1.  在DRAM中配置TDMA描述符chain
    2.  初始化TDMA配置寄存器
    3.  在DRAM中或者SRAM中初始化CESA描述符和参数
    4.  初始化CESA配置寄存器
    5.  软件激活硬件加速

#####3.2.4.4、TDMA软件流程

TDMA的基本功能和软件流程综述如下：  

*  利用TDMA从主存DRAM中拷贝报文至CESA的本地SRAM中，对于大的报文，可以进行分片。
*  存储CESA所需的加密参数
*  为CESA的本地SRAM准备一个描述符（MV\_CESA\_DESC），设定需要的操作和参数，描述符总共8个双字长，由
*  通过配置CESA描述符指针寄存器（0xE04），将SRAM中MV\_CESA\_DESC数据域地址写入选定的session中（通过计算偏移得到）
*  通过配置命令寄存器（0xE00）合适的位激活该session
*  等待硬件完成，在轮询模式或者中断模式下的判断标志为寄存器（0xE0C）第0位，若为1则说明已经actived
*  利用TDMA将完成的报文拷贝回主存

1、CESA SRAM内存映射


```
/\*   
 \* CESA SRAM Memory Map: \*  
 \* +------------------------+ <= sc->sc_sram_base + CESA_SRAM_SIZE   
 \* |                        |   
 \* |          DATA          |   
 \* |                        |   
 \* +------------------------+ <= sc->sc_sram_base + CESA_DATA(0)    
 \* | struct cesa_sa_data    |   
 \* +------------------------+   
 \* | struct cesa\_sa\_hdesc |  
 \* +------------------------+ <= sc->sc_sram_base   
 */
```
其中CESA\_SRAM\_SIZE=2048， CESA\_DATA(0)=sizeof（struct cesa\_sa\_hdesc）

2、 大小端

利用宏直接转化

3、分片处理

可以暂时不支持，以简化设计 CESA 的SDK支持最大报文为64 \* 1024，而一次处理1600大小，因此需要分片。

4、获取SRAM地址

直接返回一个全局变量：MV\_U32 cesaCryptEngBase = 0;

5、SA配置

* 定义数据结构sa\_config
* 定义配置函数

###4、调试小记
####4.1、调试基本原则   
1.  如果有不同的错误可能，为了避免代码注释混乱，采用switch语句定义不同的dec\_key方法，并用全局变量用GUI配置，则无需重新编译即可完成正确方案的选择
2.  面对时而正确时而错误的加解密情况，加入打印语句
    1.  首先排除硬件返回超时的错误
    2.  排除CESA SRAM没有初始化的错误
    3.  排除状态机的锁处理问题
    4.  定位至状态机的处理----\>处理开始之前没有正确赋给初值


####4.2、数据乱码问题定位   

    
#####4.2.1、数据拷贝流程分析   
分解TDMA和CESA的交互过程，逐步调试，至比如数据从主存至TDMA，是否拷贝正确；数据处理完之后是否拷贝至主存？
    1.  从cache到主存
    2.  DMA读： 从主存至设备
    3.  DMA写： 从设备至主存
    4.  主存至cache
#####4.2.2、Cache vs DMA   

清除cache 

        1. Flush does write back the contents of cache to main memory,
        2. 意味着将cache中的数据强制写入主存,
        3. 即设备访问到了最新的内容，保证了设备访问CPU的数据正确性    

invalide cache    

        1. Invalidate does mark cache lines as invalid so that future reads go to main memory.
        2. 确保设备完成时，CPU可以从主存中读取到最新的内容
        3. 即保证了CPU访问设备的数据正确性
   
DMA要考虑页表连续性     

        1. 现象一：有错包出现，且大小无规律；现象二：memory trap，无规律  
        2. 定位：采用软件对比方法，对比几个方面，包括DMA-out/HW caculate,分析是错误点  
        3. 分析：硬件计算错误的原因（配置错误、密钥IV错误、源目的地址错误），排除了前二者，只能是传入了错误的物理地址给硬件。在进行虚实地址转化的时候，由于转化后的物理地址可能跨页，因此必须分页拷贝，即计算出距离页边界的偏移，判断是否跨页，如果跨了先拷贝当前页数据，在拷贝下一页数据。

DMA和CPU-cache   

        1.  这是不同的两个概念，DMA指的是一种硬件设备，能够无须使用CPU指令实现对内存的拷贝；后者是主存和CPU之间的关系
        2.  DMA在进行数据拷贝的时候，或许用到CPU-cache的参与，或许根本就不用

####4.3、随机问题定位
    
   
进程并发错误

        1.  确定问题：：由于在单个进程中调用时没有问题，在多个进程调用的时候出现问题，因此可以确定可能是进程并发的问题
        2.  定位：DEBUG发现，中断会在sleep之前发生，因此在需要禁止中断  

又是volitile   

        1.  确定问题：在调试版本中时没有问题，但是在release版本中出现问题，后者相对前者做了编译优化，导致发生不可预期的错误
        2.  定位：DEBUG发现，控制结构体中的一个描述状态的字段不太正常，因此需要在其定义出加入volitile关键字，问题得以解决。

###5、参考文献

1.  [Gallery of Processor Cache Effects](http://igoro.com/archive/gallery-of-processor-cache-effects/ "http://igoro.com/archive/gallery-of-processor-cache-effects/")
2.  [Memory alignment on modern processors?](http://stackoverflow.com/questions/1855896/memory-alignment-on-modern-processors "http://stackoverflow.com/questions/1855896/memory-alignment-on-modern-processors")
3.  [Will the cache line aligned memory allocation pay off?](http://stackoverflow.com/questions/4605030/will-the-cache-line-aligned-memory-allocation-pay-off "http://stackoverflow.com/questions/4605030/will-the-cache-line-aligned-memory-allocation-pay-off")
4.  [How correctly wake up process inside interrupt handlers](http://stackoverflow.com/questions/9809140/how-correctly-wake-up-process-inside-interrupt-handlers "http://stackoverflow.com/questions/9809140/how-correctly-wake-up-process-inside-interrupt-handlers")
5.  [race condition between wait\_event and wake\_up](http://stackoverflow.com/questions/11012406/race-condition-between-wait-event-and-wake-up?rq=1 "http://stackoverflow.com/questions/11012406/race-condition-between-wait-event-and-wake-up?rq=1")
6.  [Linux kernel interrupt handler mutex protection?](http://stackoverflow.com/questions/6570419/linux-kernel-interrupt-handler-mutex-protection?rq=1 "http://stackoverflow.com/questions/6570419/linux-kernel-interrupt-handler-mutex-protection?rq=1")
7.  [DSP DMA](http://www.noise.physx.u-szeged.hu/DigitalMeasurements/DSP/Sharc/Sharc/chap06.pdf "http://www.noise.physx.u-szeged.hu/DigitalMeasurements/DSP/Sharc/Sharc/chap06.pdf")