---
layout: post
title: "线程安全函数与可重入函数"
categories:
- technology 
tags:
- thread
- reentrance
- mutex


---
  

### 线程安全 ###  

线程安全是多线程程序中的编程理念，一段代码线程安全意味着在多个线程执行的时候能够运行正确。因此，它必须满足：

- 多线程访问共享数据；
- 在任一时刻只能被一个线程访问。

保证的手段如下：  

- 编写可重入代码

一个任务执行该代码的一部分，此时另外一个任务进来，最后再恢复到原先任务。这就需要保存每一个任务的本地局部变量，一般保证在自己的栈当中，而不是static或者全局变量。  

- Mutual exclusion  

如果采用某种机制使得序列化访问共享数据，保证在任意时刻只有一个线程读写数据。由此引发条件竞争、死锁、活锁、饥饿等问题。


使用静态数据或任何其他共享资源的函数（比如文件或终端）必须通过锁使得对这些资源的访问串行化，以便函数变为线程安全。例如，以下函数是线程不安全的：     
 
{% highlight c %}  
/* thread-unsafe function */
int increment_counter()
{
	static int counter = 0;

	counter++;
    return counter;
}
{% endhighlight %}  

要使其变为线程安全的，那么必须通过静态锁将静态变量 counter 保护起来，如以下例子所示：  

{% highlight c %}
/* pseudo-code threadsafe function */
int increment_counter();
{
	static int counter = 0;
    static lock_type counter_lock = LOCK_INITIALIZER;

    pthread_mutex_lock(counter_lock);
    counter++;
    pthread_mutex_unlock(counter_lock);
    return counter;
}
{% endhighlight %}  

在使用线程库的多线程应用程序中，应使用 mutex 来对共享资源进行串行化。独立的库可能需要在线程的上下文以外工作，因此，请使用其他种类的锁。  

- 采用TLS(Thread-local storage)  

其实就是变量本地化，即确保每个thread拥有自己的私有拷贝。

{% highlight c %}
thread_t {
	int	tls;
	void	*t_tls_data;
	…
}
{% endhighlight %}

- 原子操作  

采用原子操作保证其他thread无法中断当前操作，通常通过特殊的机器指令，比如xchag和TSL。原子操作构成了多线程锁机制的基础。  

**线程安全函数**

线程安全函数通过锁来保护并发访问中的共享资源。线程安全只与函数的实现有关，并不会影响它的外部接口。

在C语言中，局部变量是在堆栈中动态分配的。因此，任何不使用静态数据或其他共享资源的函数一般都是线程安全的。  

全局数据的使用是线程不安全的。全局数据应针对每个线程保存或被封装起来，这样可以使它的访问串行化。线程可以读取对应于由另一个线程引起的错误的错误代码。

在多线程程序中，所有被多个线程调用的函数必须是线程安全的。但是，对于在多线程程序中使用线程不安全子例程有一个变通方法。虽然非重入函数通常都是线程不安全的，但是将它们变为重入常常也使它们变为线程安全。

**使函数成为重入函数**  

在多数情况下，必须用带有已修改的将要重入的函数来替代非重入函数。非重入函数不能由多个线程使用。此外，可能也无法使非重入函数变为线程安全。  

许多非重入函数会返回一个指向静态数据的指针。可以用以下方法来避免这种情况：  

- 返回动态分配的数据。在这种情况下，调用程序将负责释放存储量。好处在于无需对接口进行修改。但是，向后兼容性就无法保证了；现有的使用已修改函数的单线程程序在不更改的情况下不会释放存储量，这将导致内存泄漏。
- 使用调用程序提供的存储量。虽然必须修改接口，但是推荐使用这种方法。

{% highlight c %}
/* non-reentrant function */
char *strtoupper(char *string)
{
    static char buffer[MAX_STRING_SIZE];
    int index;

    for (index = 0; string[index]; index++)
        buffer[index] = toupper(string[index]);
        buffer[index] = 0

    return buffer;
}
{% endhighlight %}

该函数不是重入函数（也不是线程安全的函数）。要通过返回动态分配的数据来使该函数重入，那么该函数应类似于以下代码段：  

{% highlight c %}
/* reentrant function (a poor solution) */
char *strtoupper(char *string)
{
    char *buffer;
    int index;

    /* error-checking should be performed! */
    buffer = malloc(MAX_STRING_SIZE);
    
    for (index = 0; string[index]; index++)
        buffer[index] = toupper(string[index]);  
	buffer[index] = 0

    return buffer;
}
{% endhighlight %}

较好的解决方案是修改接口。调用程序必须为输入和输出字符串提供存储量，如以下代码段所示：  
   
{% highlight c %}
/* reentrant function (a better solution) */
char *strtoupper_r(char *in_str, char *out_str)
{
	int index;

    for (index = 0; in_str[index]; index++)
    	out_str[index] = toupper(in_str[index]);
   
	out_str[index] = 0

	return out_str;
}
{% endhighlight %}  

使用调用程序提供的存储量使非重入标准 C 库子例程重入。


**在连续调用中保存数据**  

在连续调用中将不保存任何数据，因为不同的线程可能连续地调用该函数。如果函数必须在连续调用中保存某些数据，比如工作缓存或指针，那么调用程序应提供该数据。  

如下示例，函数返回了字符串中连续的小写字符。该字符串只在第一次调用时提供，就像strtok子例程。函数在到达字符串的结尾处时返回 0。该函数可通过以下代码段来实现：

{% highlight c %}
/* non-reentrant function */
char lowercase_c(char *string)
{
	static char *buffer;
    static int index;
	char c = 0;

    /* stores the string on first call */
    if (string != NULL) {
    	buffer = string;
        index = 0;
	}

    /* searches a lowercase character */
    for (; c = buffer[index]; index++) {
    	if (islower(c)) {
        	index++;
        break;
        }
	}
    return c;
}
{% endhighlight %}   


该函数不是重入函数。要使其变为重入函数，那么调用程序必须保存静态数据和变量 index。该函数的重入版本可通过以下代码段来实现：  

{% highlight c %}
/* reentrant function */
char reentrant_lowercase_c(char *string, int *p_index)
{
	char c = 0;
    /* no initialization - the caller should have done it */

    /* searches a lowercase character */
    for (; c = string[*p_index]; (*p_index)++) {
    	if (islower(c)) {
			(*p_index)++;
			break;
		}
	}
    return c;
}
{% endhighlight %}  


该函数的接口和用法都发生了改变。调用程序必须向每次调用提供该字符串，且在首次调用前，必须将索引初始化为 0，如以下代码所示：   

{% highlight c %}
char *my_string;
char my_char;
int my_index;
...
my_index = 0;
while (my_char = reentrant_lowercase_c(my_string, &my_index)) {
	...
}
{% endhighlight %}  

**TLS库的线程安全**

在TLS实现中，常见的需要保证现场安全的场景主要有：

- SSL收发缓冲区；
- SSL Session Cache context
- CTR-DRBG context
- Entropy context
- RSA key context

 
目前的TLS线程安全由pthread_mutex互斥锁保证。 

**编写重入和线程安全代码**  
 
在单线程进程中，只存在一个控制流。因此，这些进程所执行的代码无需重入或是线程安全的。在多线程程序中，相同的功能和资源可以通过多个控制流并发访问。  

要保护资源的完整性，编写的多线程程序代码必须能重入并是线程安全的。重入和线程安全都与函数处理资源的方式相关。重入和线程安全是不同的概念：函数可以重入和/或线程安全化，或者两者都不可行。  

此部分提供了有关编写重入和线程安全程序的信息。其中不涉及有关编写高效线程程序的主题。高效线程程序是高效率的并行化程序。您必须在设计程序的时候考虑到线程的效率。现有的单线程程序可以成为高效线程程序，但是这需要将这些程序完全重新设计和重新编写。

除了上述方法，可以有变通方法编写：  

该变通方法很有用，尤其是在使用多线程程序中的线程不安全库进行测试时，或者同时在等待线程安全的库变成可用时。该变通方法会导致一些开销，因为它是通过对整个函数，甚至一组函数进行串行化来实现的。以下是可能的变通方法：  


- 对库使用全局锁定，并在每次使用该库的时候锁定它（通过调用一个库例程或使用一个库全局变量）。该解决方案可以会产生性能瓶颈，由于在任意给定的时间内只有一个线程能够对库的所有部分进行访问。以下伪码的解决方案只有在库很少被访问，或是作为一个初始的、快捷的实现变通方法时才是可接受的。  

{% highlight c %}  
/* this is pseudo code! */
	
	lock(library_lock);
	library_call();
	unlock(library_lock);
	
	lock(library_lock);
	x = library_var;
	unlock(library_lock);

{% endhighlight %}  
 
- 对每个库组件或组件组使用锁（例程或全局变量）。此种解决方案实现起来比上面的例子略微复杂一些，但是它可以改善性能。因为该变通方法只能在应用程序中使用，而不能在库中使用，所以可以使用 mutex 来锁定库。  

{% highlight c %}
	/* this is pseudo-code! */
	
	lock(library_moduleA_lock);
	library_moduleA_call();
	unlock(library_moduleA_lock);
	
	lock(library_moduleB_lock);
	x = library_moduleB_var;
	unlock(library_moduleB_lock);
{% endhighlight %}  

### 重入和线程安全库 ###

重入函数不在连续的调用中保存静态数据，也不返回指向静态数据的指针。所有的数据都是由函数的调用程序提供的。重入函数不得调用非重入函数。  

一般情况下，非重入函数是由其外部接口和用法标识的，但也并不总是这样。例如，strtok 子例程不是一个重入函数，因为它保存了将分割为多个标记的字符串。ctime 子例程同样不是重入函数；它返回了被每个调用所覆盖的静态数据的指针。  


重入库和线程安全库并不仅仅在线程中有用，而且在大范围的并行（和异步）编程环境中也很有用。一直使用和编写重入函数和线程安全函数是很好的编程实践。  

**使用库**  

标准C库 (libc.a)和Berkeley兼容性库(libbsd.a)以下库是线程安全的，有些标准 C 子例程是非重入的，比如 ctime 和 strtok 子例程。  

在编写多线程程序的时候，请使用子例程的重入版本来替代原来的版本。例如，以下代码段：  

{% highlight c %}
token[0] = strtok(string, separators);
i = 0;
do {
	i++;
    token[i] = strtok(NULL, separators);
} while (token[i] != NULL); 
{% endhighlight %}  


在多线程程序中应使用以下代码段来代替：  

{% highlight c %}
char *pointer;
...
token[0] = strtok_r(string, separators, &pointer);
i = 0;
do {
	i++;
    token[i] = strtok_r(NULL, separators, &pointer);
} while (token[i] != NULL);
{% endhighlight %}  


在一个程序中，线程不安全的库可能只由一个线程使用。请确保使用该库的线程的唯一性；否则，程序可出现意外的行为，甚至可能停止。

**转换库**  

在将一个现有的库转换为重入库和线程安全库的时候，请考虑以下问题。此信息只适用于 C 语言库。  

- 确定已导出的全局变量。那些变量通常是用 export 关键字在头文件中定义的。应将已导出的全局变量封装起来。变量应设为专用变量（使用 static 关键字在库源代码中定义），然后应创建访问（读和写）子例程。
- 确定静态变量和其他的共享资源。静态变量通常是用 static 关键字定义的。锁应该与任意的共享资源关联。锁定的详细程度，这样对锁数目的选择将影响到库的性能。可使用一次性初始化工具来初始化这些锁。
- 确定非重入函数并使之变为重入函数。有关更多信息，请参阅使函数成为重入函数。
- 确定线程不安全函数并使之成为线程安全函数。有关更多信息，请参阅使函数成为线程安全函数。   
