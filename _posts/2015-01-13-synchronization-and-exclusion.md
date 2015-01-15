---
layout: post
title: "Synchronization and Exclusion"
categories:
- technology
tags:
- Spinlock
- Process
- Synchronization
- Exclusion


---

##1. 进程之间为什么需要同步

在顺序程序设计中，由于代码指令都是顺序执行，重复执行会得到相同的结果，即程序与计算一一对应。  
并发程序意味着一个程序未执行完，而另外一个程序已经开始执行，这就是所谓的并发性。并发性从宏观上反映了一个时间段内有几个进程都处于运行态但运行尚未结束的状态。从微观上看，任一时刻仅有一个进程的一个操作在处理器上运行。反过来看，并发其实就是处理器在几个进程之间的多路复用器，是对有限物理资源的强制进行多用户共享，消除计算机部件之间的互等，提供系统资源的利用率。  
由于交互的并发进程共享某些资源，一个进程的执行可能会影响其他进程的执行结果，即交互的进程之间**相互制约**。因此，必须对进程的交互过程进行控制，引入了各种同步机制。  

##2. 互斥与同步的关系  

###2.1 互斥
互斥又称为竞争，并发的引入使得原本没有竞争关系的进程在访问共享资源时发生了冲突，进程之间存在**间接制约关系**。在这种关系下，一个进程获得资源，另一个进程不得不阻塞等待，因此可能会导致两个严重问题：死锁与饥饿。防止不公平，或者饿死某些低优先级的进程，是调度系统必须考虑的问题。    
###2.2 同步  
进程同步是指为完成共同任务的并发进程基于某个条件来协调其活动，因为需要在某些位置上排定执行的**先后次序**而等待、传递信号或则消息所产生的协作制约关系。这种有先后次序的协作是进程之间的直接制约关系。          
同步是为了几个具备一些依赖性的任务一起协同工作，通信则是为了同步的手段，可以基于共享内存，也可以基于消息传递。进行之间的协作可以是比知道对方名字的协作，比如通过共享内存进行松散式协作；或者进程双方知道对方名字，通过消息机制进行紧密协作。      
竞争关系从某种意义上可以看出是同步，因为存在竞争关系的进程需要互斥的访问资源，也遵循**互斥的访问次序**。


##3. 实现同步的方式

实现同步的方式有多种，可以基于软件，也可以基于硬件。历史上，其演变史大致由自旋锁至信号量，再到互斥锁。

##4. 自旋锁 
自旋锁可以通过CPU轮询机制实现同步。Spinlock作为一种临界区保护机制，在单处理器和多处理器下的实现不尽相同：    
*  对于单处理器系统，实现互斥的最简单办法就是在进程进入临界区之前关闭中断，在进程退出临界区时开中断。因为进程的上下文切换都是由中断引起的，这样进程的执行就会被打断，因此关掉中断可以保证进行互斥的进入临界区。但不适宜作为通用的互斥机制，关中断事件过程会导致系统性能和效率下降，而且在多处理器中不适用，因为在一个处理器上关闭中断，并不能防止进程在其他处理器上执行同样的临界段代码。  
*  对于多处理器系统，可以基于xchag指令和Test and Set Lock (TSL) instruction: TSL指令这两个指令实现spinlock，因为二者都是原子操作，一个处理器在处理的时候，其余处理器只有等到处理完毕才可获取访问权，实现了对临界区的互斥访问。

XCHGB的实现：  
<pre><code>
       .globl. xchgb
 xchgb:
       movl.   4(%esp), %edx   //edx = *((int32 *) (esp + 4));将最后一个参数赋值给ESP
       movl.   8(%esp), %eax   //eax = *((int32 *) (esp + 8));将第一个参数赋值给寄存器eax
       lock                    //锁住总线
       xchgb.  %al, (%edx)     //交换寄存器edx和寄存器al的值
       ret
</code></pre>

spinlock的实现：  
<pre><code>  
bool lock = false;
bool keyi = true;

spin_lock()
{   
    
    do {
        xchgb(&keyi, lock); //xchg保证原子性操作，中间不会有中断进入
    } while (keyi);
}

spin_unlock()
{
    xchgb(&keyi, lock);
}
</code></pre>  

自旋锁的适用场合与缺点：适用与临界区代码执行时间较短的场合；由于自旋锁采取忙式等待，白白浪费了CPU的时间，将能否进入临界区的责任推给了各个竞争的进程，而且**只能解决竞争问题，而不能解决进程之间的协作问题**。

##5. 信号量 

信号量于1965年由Edsger Dijkstra提出，主要是为了解决并发编程中的竞争问题，其实质是二元信号量，后来Scholten在此基础上提出了通用信号量，也称为计数信号量。不管是哪一种信号量，都加入了进程调度，CPU不再大量的忙等待。

###5.1 Linux中信号量的实现      
* **semaphore定义和初始化**    

<pre><code>
struct semaphore {
	raw_spinlock_t		lock;	//自旋锁，用于互斥保护
	unsigned int		count;	//信号量值，若count为正值，代表进入临界区之前可用的资源总数；若count为负值，意外着在该信号量队列当中注册等待的进程个数
	struct list_head	wait_list;	//信号量队列，即等待队列
};

static inline void sema_init(struct semaphore *sem, int val)
{
	static struct lock_class_key __key;
	*sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);	//初始化各个字段
	lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}
</code></pre>  

* **semaphore的P操作**  

P操作也称为down操作，即请求一个资源。
<pre><code>  
void down(struct semaphore *sem)
{
	unsigned long flags;

	raw_spin_lock_irqsave(&sem->lock, flags);	//互斥保护信号量计数
	if (likely(sem->count > 0))		//如果资源个数大于0，递减
		sem->count--;
	else
		__down(sem);	//如果资源数不大于0，则将此进程放入该信号量的等待队列。
	raw_spin_unlock_irqrestore(&sem->lock, flags);
}

static noinline void __sched __down(struct semaphore *sem)
{
	__down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}

static inline int __sched __down_common(struct semaphore *sem, long state,
								long timeout)
{
	struct task_struct *task = current;
	struct semaphore_waiter waiter;

	list_add_tail(&waiter.list, &sem->wait_list);	//加入信号量的等待队列
	waiter.task = task;
	waiter.up = false;	//设置资源释放标志位up为假

	for (;;) {
		if (signal_pending_state(state, task))
			goto interrupted;	
		if (unlikely(timeout <= 0))
			goto timed_out;
		__set_task_state(task, state);	//设置当前进程状态为TASK_UNINTERRUPTIBLE
		raw_spin_unlock_irq(&sem->lock);
		timeout = schedule_timeout(timeout);	//当前进程sleep，超时时间为MAX_SCHEDULE_TIMEOUT
		raw_spin_lock_irq(&sem->lock);
		if (waiter.up)	//一旦资源释放标志位up为真，返回0
			return 0;
	}

 timed_out:
	list_del(&waiter.list);		//超时发生，将当前进程从等待队列移除，返回-ETIME
	return -ETIME;

 interrupted:		//中断发生，将当前进程从等待队列移除，返回-EINTR
	list_del(&waiter.list);
	return -EINTR;
}

</code></pre>

* **semaphore的V操作，即up操作，尝试释放一个资源**
<pre><code>
void up(struct semaphore *sem)
{
	unsigned long flags;

	raw_spin_lock_irqsave(&sem->lock, flags);
	if (likely(list_empty(&sem->wait_list)))		//如果等待队列为空，只需将可用资源递加即可
		sem->count++;
	else
		__up(sem);		//如果在该信号量上有等待队列，以某种调度算法唤醒一个进程。比如遵循FCFS算法
	raw_spin_unlock_irqrestore(&sem->lock, flags);
}

static noinline void __sched __up(struct semaphore *sem)
{
	struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
						struct semaphore_waiter, list);
	list_del(&waiter->list);		//将当前进程从等待队列中移除
	waiter->up = true;		//设置资源释放标志up为真
	wake_up_process(waiter->task);		//唤醒一个进程
}

</code></pre>
 
信号量潜在的问题如下，其中死锁由三种情形：  
* **Accidental release**   
* **Recursive deadlock**  
* **Task-Death deadlock**  
* **Cyclic Deadlock (Deadly Embrace)**
* **Priority inversion**  
* **Semaphore as a signal**  

###5.2 Accidental release    
比如没有进程P操作就进行V操作，会导致资源访问错误。  
###5.3 Recursive deadlock      
所谓死就是进程都在等一个永远不会为真的条件，进程试图获取一个已经lock的信号量，比如如下锁实现会存在此问题。
<pre><code>
365 void
366 InitializeCriticalSection(CRITICAL_SECTION *cs)
367 {
368 .       *cs = 0;	//互斥锁初始化为0, 类型为char
369 }

382 void
383 EnterCriticalSection(CRITICAL_SECTION *cs)
384 {
385 .       int.    s = spl7();	//关中断
386 
387 .       while(xchgb(cs, 0xff) != 0) {	//若cs已经不是0，说明已经有进程持有，则转入休眠，注意此时cs = -1;若cs还是0，说明没有进程持有，继续执行。
388 .       .       sleep(cs, PZERO);	//1.将调用进程置成等待互斥锁cs的状态（sleep）；2.放入该互斥锁的队列当中；3.转向进程调度
389 .       }
390 .       if (xchgb(cs, 1) != 0)		//执行前，cs为-1，再次修改cs为0xff
391 .       .       *cs = 0xff;
392 .       splx(s);	//开中断
393 }
394 
395 void
396 LeaveCriticalSection(CRITICAL_SECTION *cs)
397 {
398 .       CRITICAL_SECTION old;
399 
400 .       old = xchgb(cs, 0);		//返回cs旧值，并赋予新值0	
401 .       if (old < 0) 
402 .       .       wakeup(cs);		//若旧值小于0，说明有进程在该锁上等待，因此需要：1.从该锁上队列中唤醒一个； 2.放入运行队； 3.自己则继续执行
403 }
</code></pre>
如果一个进程已经获取该锁，当该进程尝试再次获取该锁的时候，会再次将自己置于等待状态而无法释放。
  
###5.4 Task-Death deadlock  
如果一个拥有信号量的task死亡了或者被终止了，会有什么后果？如果不能检测这种情况，所有正在等待的的task将永远都无法获得信号量从而进入死锁。为了一定程度上解决这个问题，普遍的做法是在获得信号量的函数调用中指定一个可选的超时时间。比如前文提到的Linux实现就指定了超时机制。
###5.5 Cyclic Deadlock (Deadly Embrace)
###5.5 Priority Inversion  
大部分RTOS使用了优先级驱动的抢占调度算法。每个task拥有一个优先级，抢占调度中，低优先级的task释放CPU，使高优先级的task得以运行，这是构建一个实时操作系统的核心理念。优先级反转是指高优先级的task被低优先级的task挂起。
###5.5 Semaphore as a signal  

同步（Synchronization）这个词经常被错误地用于表示互斥（mutual exclusion）。根据定义，同步是：

**To occur at the same time; be simultaneous**  
一般来说，task之间的同步是指一个task在继续执行前，等待另外一个task的通知。还有一种情况是每个task都可能进入等待状态。互斥是一种保护机制，与此有很大不同。但是，这种错误的使用导致计数信号量可以被用于单向同步：初始化一个信号量，计数值为0。

需要注意的是，**P和V并不是在同一个task中成对出现的**。在这个例子中，假设Task1调用了P(S)它将被挂起。当Task2之后调用V(S)时，单向同步发生了，两个task都进入就绪状态（高优先级的task先运行）。不过，这种对信号量的误用是存在问题的，会导致调试起来非常困难，并可能导致accidental release类型的问题，因为单独的V(S)调用（不与P(S)配对）现在被编译器认为是合法的。

##6. 互斥体  

为了解决信号量存在的问题，1980年提出了一种新的概念——互斥（Mutual Exclusion的缩写）。互斥与二元信号量在原理上是相似的，但有一个很大的不同：属主，这就意味着如果一个task获得了互斥体，只有这个task可以释放这个互斥体。如果某个task试图释放另一个task的互斥体，将会触发错误导致操作失败。一个没有属主的“互斥体”不能被称为互斥体。


##7. 参考文献 
[semaphore_mutex](https://github.com/windflyer/windflyer.github.io/blob/master/_posts/2013/semaphore_mutex.md)  
[semaphore_mutex2](https://blog.feabhas.com/2009/09/mutex-vs-semaphores-%E2%80%93-part-1-semaphores/)